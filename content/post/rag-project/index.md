---
title: "智枢项目复盘：从 0 到 1 构建 RAG 知识管理平台"
description: "从架构设计到核心难点，复盘基于 RAG 架构的企业级知识管理系统完整开发过程。"
slug: "rag-project"
date: 2026-02-18
weight: 1
categories:
    - 项目实战
tags:
    - RAG
    - Spring Boot
    - Elasticsearch
    - Redis
    - Kafka
---

## 项目背景

在企业场景中，知识分散在各种格式的文档中（PDF、Word、PPT 等），员工查找信息效率低下。传统的关键词搜索无法理解语义，经常找不到真正需要的内容。

**智枢** 是一个基于 RAG（Retrieval-Augmented Generation）架构的企业级智能知识管理平台，支持多格式文档上传、解析与向量化，用户可以通过自然语言进行智能问答。

> 🔗 线上演示：[rag.canggo.com](https://rag.canggo.com/)

<!--more-->

## 整体架构

```
用户 → Nginx → Spring Boot Application
                   ├── MinIO（对象存储）
                   ├── Redis（缓存 / 分布式锁 / 会话管理）
                   ├── Kafka（异步消息队列）
                   │     └── 消费者 → Apache Tika（文档解析）
                   │                → 通义千问 Embedding（向量化）
                   │                → Elasticsearch（向量索引）
                   ├── Elasticsearch（混合检索引擎）
                   ├── DeepSeek API（大语言模型）
                   └── WebSocket（流式对话推送）
```

## 核心技术难点

### 一、大文件异步处理流水线

用户上传的文档可能非常大（几百 MB 甚至 1GB），同步处理会直接阻塞请求。

**解决方案：**

- 基于 **Redis BitMap** 记录每个分片的上传状态，配合 MinIO 存储分片文件，实现断点续传
- 通过文件 MD5 实现**秒传**：相同文件直接复用，避免重复上传
- 采用 `FileChannel.transferTo()` 实现**零拷贝合并**，1GB 文件合并耗时缩短至 3s 内
- 合并完成后通过 **Kafka 异步解耦**后续的解析与向量化流程

```java
// Redis BitMap 标记分片状态
redisTemplate.opsForValue().setBit(
    "upload:chunks:" + md5, chunkIndex, true);

// 零拷贝合并
try (FileChannel source = FileChannel.open(chunkPath, READ)) {
    source.transferTo(0, source.size(), targetChannel);
}
```

> 关于大文件上传方案的完整设计，详见 [大文件断点续传方案设计](/p/file-upload-design/)

### 二、混合检索与 RAG 增强

单纯的向量检索在短查询场景下表现不佳，单纯的关键词检索又无法理解语义。

**方案：KNN 向量召回 + BM25 关键词重排序**

1. 通过通义千问 Embedding 模型将文档和查询都转为 2048 维向量
2. **KNN 余弦相似度**召回语义相关的候选文档
3. **BM25** 对候选文档进行关键词相关性重排序
4. 取 Top-K 结果拼接为上下文，构建增强 Prompt 送入大模型

```java
// 混合检索：KNN 召回 + BM25 重排序
KnnSearchBuilder knn = new KnnSearchBuilder(
    "embedding_vector", queryVector, 10, 100, null);

Query bm25Query = QueryBuilders.match()
    .field("content").query(userQuery).build()._toQuery();
```

### 三、流式对话与服务降级

通过 **WebSocket** 长连接集成 DeepSeek Stream API，实现"打字机式"逐字生成体验。

**关键设计 — 自动降级机制：** 当向量 API 不可用时，自动降级至纯 BM25 文本搜索，保障服务可用性。

```java
try {
    results = vectorSearch(query);
} catch (VectorApiException e) {
    log.warn("向量检索异常，降级为文本搜索", e);
    results = bm25Search(query);  // 自动降级
}
```

### 四、细粒度权限管控

基于 Spring Security 实现 RBAC 权限模型，从角色、组织、文档多维度进行访问控制。

采用 **JWT 双令牌架构**（Access Token + Refresh Token），Access Token 短期有效（30 分钟），Refresh Token 长期有效（7 天），配合前端拦截器实现无感刷新。

## 收获与反思

1. **异步设计**是处理耗时任务的核心思路，Kafka 在解耦方面表现优秀
2. **混合检索**比单一检索方式效果好很多，融合策略的权重调优是关键
3. **降级机制**在依赖外部 API 的系统中必不可少
4. 如果重新做，会考虑引入更细粒度的文本切分策略（如语义分段），提升长文档的检索精度
