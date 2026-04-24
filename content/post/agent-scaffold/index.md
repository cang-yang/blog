---
title: "AI 智能体脚手架：多 Agent 编排框架的设计与实现"
description: "基于 DDD 架构设计的 AI Agent 编排框架，通过 YAML 声明式配置驱动多智能体自动装配与协同。"
slug: "agent-scaffold"
date: 2026-03-15
weight: 2
categories:
    - 项目实战
tags:
    - AI Agent
    - Spring AI
    - DDD
    - MCP
    - 架构设计
---

## 为什么要做这个框架？

随着大语言模型能力的提升，单个 Agent 已经无法满足复杂业务场景。多个 Agent 需要协同工作：一个负责意图理解，一个负责信息检索，一个负责内容生成...

目前主流的 Agent 框架（LangChain、AutoGen）大多基于 Python 生态，Java/Spring 生态缺乏成熟的多 Agent 编排方案。这个脚手架的目标是：**通过 YAML 配置就能快速搭建多 Agent 协同应用**。

> 🔗 线上演示：[agent.canggo.com](https://agent.canggo.com)

<!--more-->

## 整体架构（DDD 分层）

```
┌──────────────────────────────────────────────┐
│  Interface Layer（接口层）                     │
│  └── REST API / SSE 流式端点                  │
├──────────────────────────────────────────────┤
│  Application Layer（应用层）                   │
│  └── 编排执行服务 / 会话管理                    │
├──────────────────────────────────────────────┤
│  Domain Layer（领域层）                        │
│  ├── Agent 聚合根                             │
│  ├── 编排策略（顺序 / 并行 / 路由）             │
│  └── 工具 & 技能注册中心                       │
├──────────────────────────────────────────────┤
│  Infrastructure Layer（基础设施层）             │
│  ├── Spring AI 适配器                         │
│  ├── Google ADK 适配器                        │
│  └── MCP 工具连接器                           │
└──────────────────────────────────────────────┘
```

## 核心设计

### 一、双框架协同：Spring AI + Google ADK

这是框架最核心的设计挑战 —— 如何抹平两个 AI 框架的差异：

- **Google ADK** 提供了优秀的 Agent 编排引擎和会话管理
- **Spring AI** 提供了丰富的模型接入能力和工具执行机制

设计了适配层来桥接两者，上层复用 ADK 编排能力，下层复用 Spring AI 模型接入：

```java
public class SpringAiLlmAdapter implements LlmClient {

    private final ChatModel chatModel;

    @Override
    public ChatResponse call(Prompt prompt) {
        return chatModel.call(toSpringAiPrompt(prompt));
    }

    @Override
    public Flux<ChatResponse> stream(Prompt prompt) {
        return chatModel.stream(toSpringAiPrompt(prompt));
    }
}
```

### 二、YAML 声明式编排

通过 YAML 配置文件定义 Agent 及其编排关系，框架读取配置后自动完成 Bean 装配：

```yaml
agents:
  - id: intent-analyzer
    model: deepseek-chat
    system-prompt: "你是一个意图分析器..."

  - id: knowledge-retriever
    model: deepseek-chat
    tools: [search-tool, database-tool]

orchestration:
  type: sequential
  steps:
    - agent: intent-analyzer
    - agent: knowledge-retriever
      condition: "#{intent == 'query'}"
```

基于**策略路由树**解析 YAML，编排节点通过计数器遍历、类型路由与动态 Bean 回跳实现逻辑循环，支持顺序、并行、路由三种编排类型的**任意嵌套组合**。

### 三、MCP 工具统一接入

基于工厂模式统一对接三类 MCP 工具源：

| 类型 | 说明 | 示例 |
|------|------|------|
| Remote SSE | 远程 MCP 服务 | 联网搜索、API 调用 |
| Local CLI | 本地命令行工具 | 代码执行、文件操作 |
| Local Java | Java 组件 | 业务逻辑、数据查询 |

所有工具统一构建为技能列表注入模型，由框架自动完成多轮工具调用。

### 四、响应式流设计

- **流式模式**：SSE + RxJava3 桥接 Reactor 响应式流，逐块推送
- **非流式模式**：阻塞收集所有结果后统一返回
- 两种模式**共享同一套装配产物**，通过 Spring IOC 动态 Bean 注册实现装配与运行的解耦

```java
@GetMapping(value = "/chat/stream",
            produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamChat(
        @RequestParam String message) {
    return agentService.streamExecute(message)
        .map(chunk -> ServerSentEvent.builder(chunk).build());
}
```

## 落地验证：AI + Draw.io

基于该脚手架落地了一个 **AI + Draw.io 交互式绘图应用**，用户通过自然语言描述即可生成流程图。只需编写新的 YAML 配置和绘图工具实现，无需修改框架代码。这验证了框架的扩展性。

## 总结

- **抽象是关键** — 好的适配层能让不同框架无缝协作
- **配置驱动 > 代码驱动** — YAML 声明式配置大幅降低接入成本
- **响应式编程** 在流式对话场景下体验远优于传统阻塞式方案
