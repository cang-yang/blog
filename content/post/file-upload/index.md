---
title: "大文件上传方案：断点续传与秒传的工程实践"
description: "基于 Redis BitMap + MinIO 实现大文件分片上传、断点续传、秒传和零拷贝合并的完整方案。"
slug: "file-upload-design"
date: 2026-04-14
categories:
    - 技术深入
tags:
    - Redis
    - MinIO
    - 性能优化
    - 后端
---

在 [智枢项目](/p/rag-project/) 中，用户需要上传大量文档到知识库。当文件达到几百 MB 时，传统的单次上传方案完全不可行：网络波动导致上传失败、同步处理长时间阻塞。

本文分享项目中实现的方案：**分片上传 + 断点续传 + 秒传 + 零拷贝合并**。

<!--more-->

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

## 方案总结

| 技术点 | 解决的问题 |
|--------|-----------|
| 文件 MD5 + Redis | 秒传，避免重复上传 |
| Redis BitMap | 断点续传，记录分片状态 |
| FileChannel.transferTo | 零拷贝合并，消除内存瓶颈 |
| Kafka | 异步解耦上传与处理 |

> 大文件处理的核心思路：**分而治之 + 异步解耦**。把大问题拆成小问题（分片），把慢操作放到后台（Kafka 异步）。
