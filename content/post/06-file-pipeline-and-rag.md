---
title: "智枢项目 — 大文件异步流水线与 RAG 检索架构全拆解"
description: "从 1GB 文件上传优化到 3s、Redis BitMap 极限压缩、Kafka 异步解耦、双轨解析防 OOM、四级语义分块，到 RAG 混合检索打分权重设计，一次性把项目里最硬核的技术细节全部拆透。"
slug: file-pipeline-and-rag
date: 2026-04-14
categories:
    - 技术深入
tags:
    - Kafka
    - Redis
    - RAG
    - Elasticsearch
    - MinIO
    - 性能优化
    - 架构设计
---

## 写在前面

这篇文章是对我个人项目**「智枢 — 企业智能知识管理系统」**中**大文件异步流水线**模块的深度复盘。

这个模块要解决的核心问题是：用户上传一个 1GB 的大文件（PDF / Word / Excel 等），系统要完成**分片上传 → 合并 → 解析 → 智能分块 → 向量化入库**这整条链路，并且全程不能卡住用户、不能撑爆内存、不能丢数据。

下面的内容，我会按照数据流经过系统的真实顺序，一步步拆解每个环节的设计思路和踩过的坑。

<!--more-->

---

## 一、1GB 文件上传从 15s 优化到 3s

### 1.1 前端怎么切片？

前端拿到文件后，利用 HTML5 的 `Blob.slice()` 方法，按照固定大小（默认 5MB）进行循环切割。1GB 的文件会被切成大约 200 个分片。

切片前，前端会开一个 **Web Worker**（浏览器提供的后台独立线程），在后台计算整个文件的 MD5 值。这样做是为了不阻塞主线程，否则计算 1GB 文件的 MD5 会直接让网页卡死、失去响应。

> **为什么固定 5MB？**

> 因为 MinIO 兼容的 S3 协议有硬性要求：分片上传时，除了最后一个分片，其余分片**不能小于 5MB**。同时 5MB 也是一个很好的平衡点——太小会导致 HTTP 请求过于频繁，太大则一旦网络波动重传成本太高。

### 1.2 后端怎么接收？

后端的 `/api/v1/upload/chunk` 接口接收到分片后，**不会把数据落到应用服务器的本地磁盘**，而是直接通过流（Stream）透传写入 MinIO 对象存储的临时目录 `chunks/{fileMd5}/{chunkIndex}` 中。

写入成功后，做两件事：

1. 在 Redis BitMap 中标记该分片已完成

2. 在 MySQL 的 `chunk_info` 表中记录分片的存储路径和 MD5

另外，在接收第一个分片时，系统会调用**秒传检测**：查数据库和 MinIO，如果已经存在完全相同 MD5 的完整文件，直接复制元数据、返回 100% 进度，**耗时接近 0 秒**。

### 1.3 后端怎么合并？（核心优化点）

传统合并方案是：后端把 200 个分片从 MinIO 全部下载到 JVM 内存里拼接成一个 1GB 的大文件，再重新上传回 MinIO。在千兆内网下，这个"一下一上"的双向传输至少要 **15 秒**，还伴随着巨大的内存压力。

**智枢的破局方案：MinIO 服务端零拷贝合并。**

系统使用了 MinIO 的 `composeObject` API，本质上只是发送一条指令给 MinIO："请把这 200 个分片在你内部拼成一个新文件"。MinIO 在自己的磁盘阵列上直接执行数据块的链接，**文件数据完全不经过我们的 Java 服务**。

```java
List<ComposeSource> sources = partPaths.stream()

    .map(path -> ComposeSource.builder().bucket(bucketName).object(path).build())

    .collect(Collectors.toList());

minioClient.composeObject(

    ComposeObjectArgs.builder()

        .bucket(bucketName)

        .object("merged/" + fileMd5)

        .sources(sources)

        .build()

);

```

砍掉了 1GB 数据的内网双向传输开销后，合并耗时直接降到了 **3 秒以内**。

---

## 二、Redis BitMap —— 1000 个分片仅占 125 字节

### 2.1 为什么用 BitMap？

断点续传需要高频地记录和查询"哪个分片传了，哪个没传"。如果用 Redis 的 List 或 Set 来存，每个分片序号至少占几十字节，1000 个分片可能要占好几 KB。

而 BitMap 的思路极其简洁：**一个分片的状态 = 一个 Bit**。0 表示未上传，1 表示已上传。

### 2.2 具体怎么记录？

Key 的设计是 `upload:{userId}:{fileMd5}`，确保每个用户每个文件的状态独立。

```java
// 标记第 5 号分片已上传

redisTemplate.opsForValue().setBit("upload:user123:abc123", 5, true);

```

### 2.3 "125 字节"怎么算的？

1000 个分片 = 1000 个 Bit。

1 Byte = 8 Bits。

**1000 ÷ 8 = 125 Bytes。**

这个空间压缩是恐怖的——相比用 Set 存 1000 个整数（大约几 KB），BitMap 把内存占用压缩了上百倍。

### 2.4 查询进度时怎么高效解析？

系统**不会**循环发起 1000 次 Redis 请求。而是一次性把整个 BitMap 的字节数组拉取到 JVM 本地，然后通过位运算解析：

```java
// 一次性获取整个 BitMap

byte[] bitmapData = redisTemplate.execute((RedisCallback<byte[]>) connection -> {

    return connection.get(redisKey.getBytes());

});

// 在本地通过位运算检查每一位

private boolean isBitSet(byte[] bitmapData, int bitIndex) {

    int byteIndex = bitIndex / 8;

    int bitPosition = 7 - (bitIndex % 8);

    return (bitmapData[byteIndex] & (1 << bitPosition)) != 0;

}

```

CPU 指令级别的位运算，能在零点几毫秒内解析出所有分片的状态。

---

## 二、断点续传：Redis BitMap

使用 Redis BitMap 记录每个分片的上传状态。空间效率极高 —— **1000 个分片只需 125 字节**。

```java
// 标记分片已上传
public void markChunk(String md5, int index) {
    redisTemplate.opsForValue()
        .setBit("upload:chunks:" + md5, index, true);
}

// 检查所有分片是否上传完毕
public boolean allUploaded(String md5, int total) {
    Long count = redisTemplate.execute(
        (RedisCallback<Long>) conn ->
            conn.bitCount(("upload:chunks:" + md5).getBytes())
    );
    return count != null && count == total;
}

```

网络中断后，客户端重新上传时，服务端返回已上传的分片列表，**客户端只需补传缺失部分**。

## 三、Redis 挂了怎么办？三重保障机制

Redis 在这个系统里只是**加速缓存层**，不是数据安全的依赖。真正的数据安全靠的是 MySQL 和 MinIO 的双重持久化。

### 3.1 Redis 数据丢了会怎样？

前端查不到进度，会以为文件没传过，从第 0 个分片重新上传。但后端是**幂等的**——它会去查 MySQL 和 MinIO，发现分片已经存在，就跳过真实的存储上传，只在 Redis 里补上标记。用户只会感觉进度条"闪退"了一下又迅速跑满，并没有产生冗余的网络传输。

### 3.2 合并时怎么保证不损坏？

合并接口**完全不信任 Redis**。它直接去 MySQL 查所有分片记录，然后逐个调用 MinIO 的 `statObject` 做物理探查，确认每个分片文件真实存在。只有 100% 确认后才执行合并。

### 3.3 修复孤儿记录

代码中有一段自愈逻辑：如果 Redis 说"已上传"但 MySQL 没记录（或者反过来），系统会去 MinIO 确认真相，然后补齐缺失的那一方记录。

---

## 四、Kafka 异步解耦全流程

### 4.1 消息是什么时候发的？

这里有一个很容易搞混的细节。Kafka 消息**不是合并后立刻发的**，真实的顺序是：

```

① 合并分片 → 获得 1 小时有效的预签名下载链接

② 重新读取合并后的文件 → 预估向量化消耗 → 写入数据库

③ 构建 FileProcessingTask（把预签名链接塞进去）

④ 通过 Kafka 事务性发送消息

⑤ 返回响应给前端

```

### 4.2 为什么用事务性发送？

普通的异步发送没有原子性保证。可能出现"数据库更新了但消息没发出去"的半成品状态。

事务性发送（`executeInTransaction`）保证闭包内的所有发送操作要么全部成功提交（消费者才能看到消息），要么全部回滚（消息被丢弃）。

在智枢这种场景下，如果消息丢了但数据库标记为"已完成"，用户会看到文件上传成功了，但永远无法做问答检索——这种"静默失败"比直接报错更严重。

### 4.3 消费者处理流程

消费者从 Kafka 拉取到消息后，通过预签名链接从 MinIO 下载文件（**流式下载，不是一次性加载到内存**），然后依次执行：

1. **文件解析**（Tika / PDFBox 提取纯文本）

2. **智能分块**（四级语义切分）

3. **写入 MySQL**（document_vectors 表）

4. **调用向量化模型**（生成高维向量）

5. **写入 Elasticsearch**（knowledge_base 索引）

6. **回写实际消耗量**（更新 file_upload 表的 actualEmbeddingTokens）

### 4.4 消费失败了消息会丢吗？

不会。消费者处理异常时会**主动向上抛出**，触发两道防线：

- **自动重试：** 默认最多重试 10 次，每次间隔递增

- **死信队列：** 重试耗尽后，消息被转移到 `原始主题名.DLT` 的死信主题中，并且保持**与原始消息相同的分区编号**（方便排查来源）

死信消息不会被自动消费，需要人工排查修复后手动重新投递。

### 4.5 那个 1 小时的预签名链接是为消费者准备的

消费者通过这个临时链接下载文件，不需要知道 MinIO 的账号密码，降低了模块间的耦合。但如果 Kafka 消息积压超过 1 小时，链接就会过期，消费者会因为 403 而下载失败——这也是为什么需要合理设置 Kafka 分区数和消费者并发度。

---

## 五、向量化消耗预估与预扣费

### 5.1 为什么要预估？

因为 Kafka 消息是异步消费的，从发送到处理完成可能需要几分钟。如果不预扣，用户可能在这个时间窗口内连续上传 10 个大文件，每个都通过余额检查，但实际累计消耗远超余额。

### 5.2 怎么实现的？

合并完成后、发 Kafka 之前，系统会重新读取合并后的文件流，走一遍与正式解析**完全相同的分块算法**，但**只统计不存储**：

```java
private class StreamingEstimateHandler extends BodyContentHandler {

    private long estimatedTokens = 0L;

    private int estimatedChunkCount = 0;

    private void processParentChunk() {

        List<String> childChunks = splitTextIntoChunksWithSemantics(buffer.toString(), chunkSize);

        estimatedChunkCount += childChunks.size();

        estimatedTokens += usageQuotaService.estimateEmbeddingTokens(childChunks);

        buffer.setLength(0);

    }

}

```

预估完成后，系统先预扣用户余额。等消费者实际处理完成后，再用真实消耗量"多退少补"。

如果预估消耗超过用户余额，合并接口会直接拒绝，Kafka 消息根本不会被发送。

---

## 三、零拷贝合并

**传统方案的问题：** 读取所有分片到内存 → 写入目标文件。1GB 文件意味着至少 1GB 内存占用。

**零拷贝方案：** 利用 `FileChannel.transferTo()` 在内核态完成数据拷贝，不经过用户态缓冲区：

```java
public void mergeChunks(String md5, int totalChunks, Path target) {
    try (FileChannel dest = FileChannel.open(target,
            CREATE, WRITE, TRUNCATE_EXISTING)) {

        for (int i = 0; i < totalChunks; i++) {
            Path chunk = getChunkPath(md5, i);
            try (FileChannel src = FileChannel.open(chunk, READ)) {
                src.transferTo(0, src.size(), dest);
            }
        }
    }
}

```

**效果：1GB 文件合并耗时从 15s+ 降至 3s 内**，且内存占用几乎为零。

## 一、秒传：文件 MD5 去重

客户端上传前先计算文件 MD5，服务端检查是否已存在相同文件：

```java
public UploadResult checkFile(String md5) {
    String existingId = redisTemplate.opsForValue()
        .get("file:md5:" + md5);

    if (existingId != null) {
        return UploadResult.secondPass(existingId); // 秒传
    }

    Set<Integer> uploaded = getUploadedChunks(md5);
    return UploadResult.needUpload(uploaded); // 返回待上传分片
}

```

## 六、JVM 内存熔断机制

### 6.1 为什么需要？

对于 PDF 文件，PDFBox 需要把整个文档树加载到内存中（为了按页提取和去页眉页脚），一个 500 页的 PDF 可能吃掉几百 MB 内存。如果同时处理多个大 PDF，堆内存很容易被撑爆。

另外，即使是普通文档走 Tika 路线，底层的 POI 库在解析几十万行 Excel 时，也可能在内部缓存大量对象。

### 6.2 怎么实现的？

在所有解析任务的入口处，有一道"内存安全门卫"：

```java
private void checkMemoryThreshold() {

    Runtime runtime = Runtime.getRuntime();

    long usedMemory = runtime.totalMemory() - runtime.freeMemory();

    double memoryUsage = (double) usedMemory / runtime.maxMemory();

    if (memoryUsage > 0.8) {           // 超过 80%

        System.gc();                    // 尝试 GC 自救

        // 重新检测

        usedMemory = runtime.totalMemory() - runtime.freeMemory();

        memoryUsage = (double) usedMemory / runtime.maxMemory();

        if (memoryUsage > 0.8) {       // 回收后依然超标

            throw new RuntimeException("内存不足，拒绝处理");

        }

    }

}

```

三步走：**探测 → 自救（GC）→ 果断拒绝**。宁可让一条消息进入重试/死信，也绝不冒 OOM 崩溃的风险。

---

## 七、双轨解析架构（核心设计）

这是整个大文件流水线中我最满意的一块设计。

### 7.1 为什么 PDF 不能用 Tika？

Tika 当然能处理 PDF，但会引发两个灾难性后果：

1. **页眉页脚污染语义：** Tika 分不清正文和页眉，它会把"内部机密"、"第三章 架构设计"这种页眉页脚文本夹杂在正文中间，导致向量化后的数据质量极差。

2. **丢失页码信息：** 企业知识库场景下，用户提问后系统必须告诉他"答案来自第 15 页"。Tika 的流式处理是一条线读到底的，根本不知道当前读到了第几页。

所以我专门引入了 PDFBox，为 PDF 开辟了一条独立的精细化处理路线。

### 7.2 双轨是怎么分的？

```java
public void parseAndSave(String fileMd5, InputStream fileStream, ...) {

    checkMemoryThreshold();   // 先做内存体检

    if (isPdfDocument(bufferedStream)) {

        parsePdfAndSave(...); // PDF 专属路线

        return;               // 直接 return，不走 Tika！

    }

    // 非 PDF → Tika 通用路线

    StreamingContentHandler handler = new StreamingContentHandler(...);

    parser.parse(bufferedStream, handler, metadata, context);

}

```

**两条路线的核心区别：**

| 维度 | PDF 路线（PDFBox） | 非 PDF 路线（Tika） |
|:--|:--|:--|
| 解析方式 | 加载完整文档树，按物理页遍历 | 流式读取，1MB 缓冲区 |
| 页码信息 | ✅ 精准保留每页页码 | ❌ 无页码，存 null |
| 页眉页脚处理 | ✅ 专属去噪逻辑 | ❌ 不需要 |
| 内存占用 | 较高（需要加载文档树） | 极低（永远只有 1MB） |
| OOM 风险 | 有，靠内存熔断兜底 | 几乎没有 |

### 7.3 PDF 的页眉页脚是怎么去除的？

思路非常巧妙：**如果某行文本在很多页的顶部或底部都重复出现，那它大概率就是页眉或页脚。**

具体步骤：

1. 逐页提取文本，把每页按行切割

2. 收集每页**顶部前 3 行**和**底部后 3 行**的有效文本

3. 对这些边界行做**归一化处理**——关键操作是把所有数字替换为 `#`（这样"第 1 页"和"第 37 页"都变成"第 # 页"，被识别为同一行）

4. 统计归一化后的行在所有页中出现的次数，超过阈值（2~3 次）的判定为页眉/页脚

5. 从每页中剔除这些行

### 7.4 非 PDF 的流式处理具体怎么工作？

Tika 的工作模式是**"推送模型"**：它每次从输入流中读取几 KB 的原始数据，解码后提取出纯文本，然后调用我们自定义的 `characters` 回调方法。

我们在回调方法里把文本累积到一个 `StringBuilder` 缓冲区中，当缓冲区达到 1MB 时，就触发一次分块处理，然后立刻清空缓冲区：

```java
@Override

public void characters(char[] ch, int start, int length) {

    buffer.append(ch, start, length);

    if (buffer.length() >= parentChunkSize) {  // 1MB

        processParentChunk();

    }

}

```

**所以 JVM 内存中任何时刻最多只有约 1MB 的文本数据**，无论原始文件是 100MB 还是 10GB。数据像流水一样经过 JVM，处理完就释放，永远不会蓄积。

---

## 八、四级语义分块算法

PDF 和非 PDF 在前半段的"解析"走的是两条不同的路，但在后半段的"分块和入库"时，它们殊途同归，汇聚到同一个方法：`splitTextIntoChunksWithSemantics`。

### 8.1 为什么不直接按固定字数切？

固定切分（比如每 500 字一刀）极容易把完整的句子甚至专有名词拦腰截断，导致向量化后的语义是破碎的。比如"中华人民共和国"可能被切成"中华人民共"和"和国"。大模型拿到这种上下文，回答质量会严重下降。

### 8.2 四级递进式切分

我按语义边界的优先级进行层层降级：

**第一级 — 段落切分（最高优先级）：**

```java
String[] paragraphs = text.split("nn+");

```

段落是最天然的语义单元。优先在段落边界处切割，能最大程度保持语义的内聚性。

**第二级 — 句子切分（处理超长段落）：**

```java
String[] sentences = paragraph.split("(?<=[。！？；])|(?<=[.!?;])s+");

```

如果单个段落超过块大小限制，就降级到按中英文标点符号切割。

**第三级 — HanLP 智能分词（处理超长句子）：**

```java
List<Term> termList = StandardTokenizer.segment(sentence);

```

如果连单个句子都超长，调用 HanLP 自然语言处理模型按词语边界切分，确保"中华人民共和国"这样的专有名词不会被破坏。

**第四级 — 字符切分（终极保底）：**

```java
catch (Exception e) {

    chunks = splitByCharacters(sentence, chunkSize);

}

```

如果 HanLP 异常，退化为最原始的逐字符切分。只是保底，正常情况下不会触发。

### 8.3 页码是怎么附加的？

分块方法本身**完全不关心页码**，它只负责切字符串。页码是在外层的调用方控制的：

- **PDF 路线**调用时，传入真实的 `pageNumber`（比如 5）

- **非 PDF 路线**调用时，传入 `null`

保存方法 `saveChildChunks` 收到什么页码就存什么页码，不做任何分类判断：

```java
private int saveChildChunks(..., Integer pageNumber) {

    for (String chunk : chunks) {

        vector.setPageNumber(pageNumber); // PDF 传 5 就存 5，非 PDF 传 null 就存 null

        documentVectorRepository.save(vector);

    }

}

```

这种设计遵循了**高内聚、低耦合**的原则：分块算法是纯粹的、无状态的文本处理函数。未来如果要支持新格式（比如 PPT 按幻灯片页切分），核心分块算法一行代码都不用改。

---

## 九、混合检索（KNN + BM25）

### 9.1 为什么必须混合？

**只用 BM25（关键词检索）的问题：**

用户问"怎么请假"，但文档里写的是"员工带薪休假管理办法"。BM25 匹配不到任何关键词，直接返回空。

**只用 KNN（向量检索）的问题：**

用户搜"RFC 7519"，KNN 返回一堆跟"网络协议"语义相关但完全不包含"RFC 7519"这个关键词的文档，答非所问。

**混合检索让两种能力互补：** KNN 负责语义召回（把所有可能相关的候选都捞上来），BM25 负责精准排序（确保最终结果包含用户的原始关键词）。

### 9.2 两阶段架构与打分权重

**第一阶段 — KNN 粗召回 + 关键词硬过滤：**

先用 KNN 在向量空间中大范围召回 `topK × 30` 个语义相近的候选文档，然后用 `must` 子句强制要求候选文档必须包含用户查询中的关键词。语义再相似但不含关键词的，直接淘汰。

**第二阶段 — BM25 精排（Rescore 重打分）：**

```java
s.rescore(r -> r

    .windowSize(recallK)

    .query(rq -> rq

        .queryWeight(0.2d)           // KNN 分数权重：20%

        .rescoreQueryWeight(1.0d)    // BM25 分数权重：100%

        .query(rqq -> rqq.match(m -> m

            .field("textContent")

            .query(query)

            .operator(Operator.And)

        ))

    )

);

```

最终得分 = `0.2 × KNN 分 + 1.0 × BM25 分`。BM25 主导最终排序，KNN 只作为辅助信号。

在企业知识库场景中，用户提问通常已经包含了比较明确的关键词，关键词的精确匹配度对结果质量的影响远大于语义相似度。

### 9.3 向量化失败时的自动降级

如果向量模型接口异常（比如网络超时），系统不会直接报错，而是自动降级为纯 BM25 文本搜索：

```java
final List<Float> queryVector = embedToVectorList(query, userId);

if (queryVector == null) {

    logger.warn("向量生成失败，仅使用文本匹配进行搜索");

    return textOnlySearchWithPermission(query, userDbId, userEffectiveTags, topK);

}

```

这保证了即使外部依赖出问题，用户的搜索功能也不会完全不可用。

---

## 十、动态上下文增强策略

### 10.1 什么是长文本限制？

大语言模型有固定的上下文窗口（比如 8K / 32K / 128K 个 Token）。企业文档动辄几万字甚至几十万字，直接塞给模型要么超限报错，要么注意力分散、回答质量暴跌。

### 10.2 怎么突破？

核心思路：**不把整本书塞给模型，只把最相关的几页纸递给它。**

1. 文档上传时，就按四级分块算法切成一个个小型文本块，向量化后存入 Elasticsearch

2. 用户提问时，通过混合检索精准找出最相关的 5~10 个文本块

3. 将这些块**动态拼接**成上下文，注入大模型的提示词中

"动态"体现在：每次用户问的问题不同，检索到的块也不同，上下文是实时组装的。并且系统可以根据模型的窗口大小，动态调整召回的块数量。

---

## 四、异步处理

合并完成后通过 Kafka 异步触发后续流程，客户端无需等待：

```java
kafkaTemplate.send("file-process-topic",
    new FileProcessMessage(fileId, md5, filePath));

// 消费者异步执行：
// 1. Apache Tika 解析文档内容
// 2. 文本分段
// 3. 通义千问 Embedding 向量化
// 4. 写入 Elasticsearch 索引

```

## 写在最后

回顾整个大文件异步流水线，最让我有成就感的，不是某一个单独的技术点，而是**整条链路的协同设计**：

- 前端切片 + 后端 MinIO 零拷贝合并 → **解决传输瓶颈**

- Redis BitMap + MySQL + MinIO 三重保障 → **解决数据可靠性**

- Kafka 事务性发送 + 重试 + 死信 → **解决异步可靠性**

- 内存熔断 + 双轨解析 + 流式缓冲 → **解决稳定性**

- 四级语义分块 + 混合检索 → **解决 RAG 质量**

- 预估消耗 + 预扣费 → **解决计费准确性**

每个环节都不是孤立的炫技，而是为了整条链路的最终目标——**让用户上传任意格式的大文件后，能通过自然语言高质量地找到他想要的答案**。

## 方案总结

| 技术点 | 解决的问题 |
|--------|-----------|
| 文件 MD5 + Redis | 秒传，避免重复上传 |
| Redis BitMap | 断点续传，记录分片状态 |
| FileChannel.transferTo | 零拷贝合并，消除内存瓶颈 |
| Kafka | 异步解耦上传与处理 |

> 大文件处理的核心思路：**分而治之 + 异步解耦**。把大问题拆成小问题（分片），把慢操作放到后台（Kafka 异步）。

## 整体流程

```

客户端                        服务端                     存储
  │                             │                        │
  ├── 1. 计算文件 MD5 ────────>│                        │
  │                             ├── 2. Redis 查是否秒传   │
  │<── 3. 返回已上传分片列表 ──│                        │
  │                             │                        │
  ├── 4. 上传缺失分片 ────────>│── 5. 存入 MinIO ──────>│
  │                             ├── 6. BitMap 标记分片    │
  │                             │                        │
  ├── 7. 全部分片完毕 ────────>│                        │
  │                             ├── 8. 零拷贝合并         │
  │                             ├── 9. Kafka 异步处理     │
  │<── 10. 返回成功 ──────────│                        │

```
