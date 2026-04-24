---
title: "AI 智能体脚手架：从 YAML 配置到多 Agent 协同的工程实践"
description: "基于 DDD + 责任链模式构建 AI 智能体装配与编排框架，实现 Google ADK 与 Spring AI 双框架适配、多模态对话、流式传输的完整方案。"
slug: ai-agent-scaffold-deep-dive
date: 2026-04-20
categories:
    - 技术深入
tags:
    - Spring AI
    - AI Agent
    - DDD
    - MCP
    - 架构设计
---

这个项目要解决的问题是：怎么用一份 YAML 配置文件，就自动组装出一个能跑的、支持多智能体协同的 AI Agent？

听起来简单，实际做起来涉及两套框架的适配、多模态消息的格式转换、会话上下文的维持、流式传输……坑还挺多的。

本文把整个项目的核心设计从头到尾梳理一遍。

<!--more-->

## 一、整体架构：DDD 分层

项目用的是 DDD 领域驱动设计，不是传统的 MVC。核心区别就一句话：**MVC 里服务层依赖数据访问层，是正向依赖；DDD 里领域层定义接口，基础设施层去实现，是依赖倒置。**

一共 6 个模块：

┌─────────────────────────────────────────────────────┐
│  启动层（app）                                        │
│  Spring Boot 启动入口 + 监听器 + 线程池配置            │
├─────────────────────────────────────────────────────┤
│  触发层（trigger）                                    │
│  HTTP 控制器，接收前端请求，调用领域层                   │
├─────────────────────────────────────────────────────┤
│  接口契约层（api）                                     │
│  只定义接口 + DTO，不写任何实现                         │
├─────────────────────────────────────────────────────┤
│  领域层（domain）⭐ 核心                               │
│  装配链路、对话服务、适配器、消息转换器                   │
├─────────────────────────────────────────────────────┤
│  基础设施层（infrastructure）                          │
│  实现领域层定义的接口（数据库、外部API等）                │
├─────────────────────────────────────────────────────┤
│  公共类型层（types）                                   │
│  枚举、异常类、常量                                    │
└─────────────────────────────────────────────────────┘

```

为什么接口契约层和触发层要分开？因为接口契约层定义的是**对外的 API 契约**，将来如果要从 HTTP 换成 RPC，只需要新增一个触发层实现，接口契约层完全不动。而且其他服务要调用我们，只需要依赖这一个契约模块，不会碰到任何实现代码。

## 二、双框架适配：为什么要同时用两个框架

这个项目同时用了 Google ADK 和 Spring AI，各取所长：

| 框架 | 擅长的事 | 在项目中的角色 |
|------|---------|---------------|
| Google ADK | 多智能体编排、会话管理 | 上层，管调度 |
| Spring AI | 多模型接入、工具自动执行 | 下层，管调用 |

问题来了：两套框架的消息格式不一样。ADK 用的是 Content / Part 结构，Spring AI 用的是 Prompt / Message 结构。所以中间需要一个适配器来做转换。

这个适配器是整个项目的**枢纽**，它的核心方法做三件事：

```

SDK 格式的消息 ──→ 【消息转换器】 ──→ Spring AI 格式
                        ↓
              根据流式标志选择调用方式
              流式 → stream()
              非流式 → call()
                        ↓
Spring AI 格式的响应 ──→ 【消息转换器】 ──→ SDK 格式返回

```

## MySpringAI 适配器

### 为什么需要它

`LlmAgent` 来自 Google ADK 框架，它要求的模型接口是 `LlmModel`（Google 自己定义的）。`ChatModel` 来自 Spring AI 框架，方法名、参数类型、返回类型全都不一样。直接把 `ChatModel` 塞给 `LlmAgent`，编译都过不了。

`MySpringAI` 就是一个翻译官（适配器），对上实现 Google ADK 的 `LlmModel` 接口，对下内部调用 Spring AI 的 `ChatModel` 方法：

```java
public class MySpringAI implements LlmModel {
    private final ChatModel chatModel;  // 内部持有 Spring AI 的模型

    public LlmResponse generate(LlmRequest request) {
        Prompt springPrompt = convertToSpringPrompt(request);  // Google格式 → Spring格式
        ChatResponse response = chatModel.call(springPrompt);   // 用Spring AI真正调LLM
        return convertToAdkResponse(response);                  // Spring格式 → Google格式
    }
}

```

放在 `patch` 包里，说明是对 Google ADK 官方适配器 `SpringAI` 的补丁版本——可能修了 bug，可能加了自定义逻辑。

## 项目背景

这个项目要做的事情说起来就一句话：**从一份 YAML 配置文件出发，自动组装出一台能让 20 个 AI 智能体协作运转的机器，然后注册到 Spring 容器里等着被调用。**

听起来不复杂，但实际上组装过程涉及 API 连接创建、聊天模型构建、工具挂载、20 个基础 Agent 创建、8 个不同类型的工作流编排、还有嵌套引用……如果把这些逻辑全塞进一个大方法里，估计得写 500 行，改一个步骤就得在这 500 行里翻来翻去，以后想加个新步骤更是噩梦。

所以项目用了**规则树**这种设计模式，把整个组装过程拆成一个个独立的节点，像流水线一样一站接一站地执行。每个节点只管自己的事，干完活就告诉框架"下一站去哪"。

<!--more-->

## 三、装配链路：从 YAML 到可运行的智能体

### 3.1 触发阶段

Spring Boot 启动时把 YAML 里的属性自动映射到配置类。启动完成后发出一个"应用就绪"事件，监听器捕获到后取出所有智能体的配置列表，传给装配服务。

### 3.2 责任链执行

装配链路是一条 6 节点的责任链，每个节点只做一件事，通过**动态上下文对象**传递中间产物：

```

根节点（空节点，哨兵）
  │
  ▼
API 节点
  │  读取配置中的 URL 和密钥
  │  创建 API 连接对象 → 存入上下文
  ▼
模型节点
  │  取出 API 连接 + 模型名称
  │  遍历 MCP 工具列表（远程SSE/本地命令行/本地JavaBean）
  │  遍历 Skills 技能文件（文档+脚本）
  │  所有工具统一构建为 ToolCallback → 放入同一个列表
  │  创建 ChatModel 对象 → 存入上下文
  ▼
智能体节点
  │  遍历配置中的每个智能体
  │  取名称、描述、提示词、输出键
  │  用适配器包装 ChatModel
  │  创建 LLMAgent → 存入上下文的 Map 集合
  ▼
编排节点（1 主节点 + 3 子节点，逻辑循环）
  │  详见 3.3 节
  ▼
运行器节点
  │  取出配置指定的入口智能体
  │  创建 InMemoryRunner
  │  包装成注册对象 → 动态注册为 Spring Bean
  ▼
  装配完成 ✓

```

### 3.3 编排节点的逻辑循环

这是最复杂的节点。主节点和三个子节点之间形成了一种循环结构：

```

         ┌──────────────┐
         │   主节点      │
         │              │
         │ 计数器 >= 总数?│──── 是 ────→ 运行器节点
         │              │
         │   否，按类型   │
         │   路由到子节点  │
         └──┬───┬───┬───┘
            │   │   │
     ┌──────┘   │   └──────┐
     ▼          ▼          ▼
  串行子节点  并行子节点  循环子节点
     │          │          │
     └──────┬───┘──────────┘
            │
            │  创建完 → 存入集合 → 路由回主节点
            ▼
         ┌──────────────┐
         │   主节点      │  ← 继续循环
         └──────────────┘

```

三个子节点的路由方法都指回主节点，但这里有个工程细节——**子节点不能用字段注入来引用主节点**，否则会和主节点形成循环依赖，Spring 启动直接报错。所以子节点用的是运行时动态获取：

```java
// 子节点的路由方法
@Override
public StrategyHandler get(...) {
    // 不用 @Resource 注入，而是运行时从容器取
    // 因为主节点持有子节点引用，子节点再注入主节点就循环了
    return getBean("agentWorkflowNode");
}

```

字段注入在 Bean 初始化阶段就要拿到引用，此时对方可能还没创建完；动态获取是在运行时去容器里取，这时候所有 Bean 早就创建好了。

另外，编排智能体和普通的 LLM 智能体实现了**同一个接口**，所以可以放在同一个 Map 里。先创建的编排智能体可以被后创建的引用——这就是嵌套编排的实现原理。比如先创建一个并行编排，后面的串行编排可以把它当子节点用。

### 3.4 动态上下文里装了什么

```

DynamicContext（共享黑板）
├── API 连接对象      ← API 节点写入，模型节点读取
├── ChatModel 对象    ← 模型节点写入，智能体节点读取
├── 智能体 Map 集合   ← 智能体节点 + 编排节点共同写入
├── 编排步骤计数器    ← 编排节点用来控制循环
└── 当前编排配置      ← 编排节点用来判断类型和路由

```

每个智能体的装配都用一个全新的空上下文，多个智能体之间完全隔离。

### 3.5 为什么用责任链而不是写一个大方法

四个字：**开闭原则**。

想在装配流程中加一个新的处理步骤？新建一个节点类，改一下前后节点的路由指向，原有节点一行不动。如果是大方法，要在几百行代码中间找位置插入，还得处理变量作用域冲突。

还有一个很实际的好处：如果第 5 个节点需要第 2 个节点产生的数据，直接从上下文里读就行。如果用节点间传参，这个数据就得在第 3、第 4 个节点一路透传过去，中间节点被迫携带自己根本不需要的参数。

## 谁是 Bean 谁不是

这个点特别容易混淆，单独拎出来说清楚：

**是 Bean 的：**
- `AiAgentAutoConfigProperties`（`@EnableConfigurationProperties` 注册的）
- 所有 `@Service` 标记的节点类（RootNode、AiApiNode 等等）
- `AiAgentRegisterVO`（RunnerNode 里手动 `registerBean()` 注册的，整条流水线唯一一次手动注册）

**不是 Bean 的：**
- `AiAgentConfigTableVO`（只是配置类 Bean 里的一个字段值，不能被 `@Resource` 注入）
- `ArmoryCommandEntity`（每次调用时 new 出来的参数"信封"）
- `DynamicContext`（每次调用时 new 出来的上下文"托盘"）
- 流水线中间创建的所有对象（OpenAiApi、ChatModel、LlmAgent、LoopAgent 等等）

一句话记：**配置类**（Properties）生出**配置数据**（TableVO），配置数据装进**信封**（CommandEntity），信封送进流水线。

## MCP/Skills 工具创建细节

### 为什么需要工具

模型只会"说话"，不会"做事"。工具让模型能调用外部能力——搜索网页、读取文件、调用本地 Java 方法等等。不管工具从哪来，最终都要变成 Spring AI 统一的 `ToolCallback` 类型。

### 三种 MCP 工具类型

| 类型  | 通信方式                             | 比喻                   |
| ----- | ------------------------------------ | ---------------------- |
| SSE   | HTTP 长连接到远程服务器              | 打电话给远程专家       |
| Stdio | 启动本地子进程，通过标准输入输出通信 | 在电脑上开一个助手程序 |
| Local | 直接从 Spring 容器取 Bean            | 找身边的同事帮忙       |

工厂 `DefaultMcpClientFactory` 判断 `ToolMcp` 对象里 `sse`/`local`/`stdio` 三个字段哪个不为 null，就返回对应的创建服务——因为 YAML 的结构决定了每个 MCP 配置只会有一个子字段有值。

### Skills 工具

Skills 是写在文件里的工具描述（比如 Markdown 格式），有两种加载方式：

```java
if ("directory".equals(type)) {
    // 从文件系统绝对路径加载——适合本地开发
    SkillsTool.builder().addSkillsDirectory(path).build();
}
if ("resource".equals(type)) {
    // 从 classpath 资源路径加载——适合打包部署
    SkillsTool.builder().addSkillsResource(new ClassPathResource(path)).build();
}

```

### 一个 Java 小知识：`toArray(new ToolCallback[0])`

这是 Java 里把 List 转成指定类型数组的惯用写法。传 `new ToolCallback[0]` 不是为了用这个空数组装东西，纯粹是告诉 Java "我要的数组类型是 ToolCallback"——因为泛型在运行时会被擦除，List 不知道自己装的是什么类型。

直觉上传正确大小的 `new ToolCallback[list.size()]` 应该更快，但实际上传空数组反而更快：JIT 编译器能识别出 toArray 内部"创建数组后立刻被完全覆盖"的模式，直接跳过无意义的零初始化。而外部创建的数组 JIT 不敢优化，零初始化白白浪费了。

## 装配流水线的完整链路

### 整体结构

```

YAML（原材料）
  │
  ▼
第1站 RootNode        → 纯入口，像链表头指针，什么都不做
  │
  ▼
第2站 AiApiNode       → 创建 OpenAiApi（API连接对象）
  │
  ▼
第3站 ChatModelNode   → 创建 ChatModel（聊天模型 + MCP/Skills工具）
  │
  ▼
第4站 AgentNode       → 创建 20 个基础 LlmAgent，全部放入 agentGroup 货架
  │
  ▼
第5站 AgentWorkflowNode → 循环处理 8 个工作流配置，每次根据类型分发到子节点
  │     ├── LoopAgentNode       → 处理 loop 类型 → 回到 AgentWorkflowNode
  │     ├── ParallelAgentNode   → 处理 parallel 类型 → 回到 AgentWorkflowNode
  │     └── SequentialAgentNode → 处理 sequential 类型 → 回到 AgentWorkflowNode
  │
  ▼
第6站 RunnerNode      → 取出最终入口 Agent，包装成 InMemoryRunner，注册到 Spring
  │
  ▼
成品：AiAgentRegisterVO（包着 runner 的信封）

```

### 启动入口

Spring Boot 启动完成后，`AiAgentAutoConfig` 监听到 `ApplicationReadyEvent` 事件，自动触发装配：

```java
@Configuration
@EnableConfigurationProperties(AiAgentAutoConfigProperties.class)
public class AiAgentAutoConfig implements ApplicationListener<ApplicationReadyEvent> {

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        // 从配置类（Bean）里取出配置数据（不是Bean，只是普通Java对象）
        // 传给装配服务开始组装
        armoryService.acceptArmoryAgents(
            new ArrayList<>(aiAgentAutoConfigProperties.getTables().values()));
    }
}

```

这里有个容易混淆的点：`AiAgentAutoConfigProperties` 是 Bean（被 `@EnableConfigurationProperties` 注册的），但它里面的 `AiAgentConfigTableVO` 不是 Bean——只是 Spring Boot 用 `new` 创建出来填充字段的普通 Java 对象。就像"箱子在货架上有登记，但箱子里的东西没有单独登记"。

### 上下文：DynamicContext

上下文是自己定义的，不是框架提供的。它就是一个跟着流水线走的"托盘"，每个节点都能往上面放东西、从上面拿东西：

```java
public static class DynamicContext {
    private OpenAiApi openAiApi;          // 第2站放入
    private ChatModel chatModel;          // 第3站放入
    private Map<String, BaseAgent> agentGroup = new HashMap<>();  // 第4站开始持续放入
    private AtomicInteger currentStepIndex = new AtomicInteger(0); // 第5站的循环计数器
    private AgentWorkflow currentAgentWorkflow;  // 第5站每次循环覆盖写入
}

```

### 第5站：最复杂的 AgentWorkflowNode

这个节点是整棵规则树里最精妙的部分。前面 4 站都是直线走下去，到这里开始出现**循环 + 分支**。

循环不是用 for/while 实现的，而是通过 `currentStepIndex` 计数 + 子节点回指自身来形成逻辑上的循环：

```java
// AgentWorkflowNode.doApply() 核心逻辑
if (dynamicContext.getCurrentStepIndex() >= agentWorkflows.size()) {
    // 全部处理完了，跳到 RunnerNode
    dynamicContext.setCurrentAgentWorkflow(null);
    return router(requestParameter, dynamicContext);
}
// 取出当前索引对应的 workflow，放入上下文
dynamicContext.setCurrentAgentWorkflow(agentWorkflows.get(dynamicContext.getCurrentStepIndex()));
dynamicContext.addCurrentStepIndex();  // 索引 +1
return router(requestParameter, dynamicContext);

```

`get()` 方法根据 workflow 的 type 字段决定分支走向：

```java
return switch (node) {
    case "loopAgentNode"       -> loopAgentNode;
    case "parallelAgentNode"   -> parallelAgentNode;
    case "sequentialAgentNode" -> sequentialAgentNode;
    default -> runnerNode;
};

```

三个子节点处理完后都回到 `AgentWorkflowNode`——它们的 `get()` 都写的 `return getBean("agentWorkflowNode")`。用 `getBean()` 而不是 `@Resource` 是为了避免循环依赖。

### 嵌套的秘密：agentGroup

所有 Agent（不管是基础的 LlmAgent 还是组装好的 LoopAgent/ParallelAgent/SequentialAgent）都放在同一个 `agentGroup` 这个 Map 里。后面的 workflow 可以通过名字引用前面已经组装好的 workflow，因为它们**继承自同一个父类 BaseAgent**，类型是兼容的。

这就是为什么 YAML 里 workflow 的声明顺序必须正确——先被引用的必须先创建。如果顺序乱了，`queryAgentList()` 按名字去 agentGroup 里取的时候会取不到，被直接跳过，导致组装出来的 Agent 树缺胳膊少腿。

## 规则树到底是什么

### 一句话定义

规则树 = **策略模式 + 责任链模式 + 组合模式**的混合体。每个节点是一个独立策略，节点之间通过路由串成链，某些节点还能根据条件分叉出不同的分支，形成树形结构。

### 为什么叫"树"不叫"链"

如果每个节点的下一站都是固定的，那就是一条链。但这个项目里有一个关键节点 `AgentWorkflowNode`，它会**根据工作流的类型动态决定下一站去哪**——是 loop 就去 LoopAgentNode，是 parallel 就去 ParallelAgentNode，是 sequential 就去 SequentialAgentNode。这就产生了分支，画出来就是一棵树。

### 为什么不写一个大方法

不是做不到，是**做完之后没法维护**。任何用规则树实现的逻辑，用一个大方法都能实现，甚至可能 200 行就写完了。但规则树的价值不在于"能不能实现"，而在于"实现之后好不好改"。每个步骤独立成类，想改某一步就只改那一个类，想插入新步骤就写个新类改一下前后节点的指向，想删掉某个步骤就删类改指向——团队协作时多人改不同的节点类也不会冲突。

## 四、装配与运行的衔接

装配阶段的最终产物是一个注册对象，里面包含智能体 ID、名称、描述和**内存运行器**。

内存运行器持有四样东西：

```

InMemoryRunner
├── 入口智能体     （可以是 LLMAgent 也可以是编排智能体）
├── 智能体名称
├── 工具列表       （所有 MCP + Skills）
└── 会话服务       （管理用户会话的生命周期和对话历史）

```

装配链路最后一个节点通过 **Spring 动态 Bean 注册**，以 agentId 为名称把注册对象注册到 IOC 容器：

```java
// 注册逻辑（伪代码）
if (容器中已存在同名 Bean) {
    先移除旧的;  // 支持热替换
}
注册新的 Bean(agentId, 注册对象);

```

运行时用户请求过来，对话服务通过 agentId 从容器里取出注册对象，拿到运行器就能执行。装配不需要知道谁会调用，运行不需要知道怎么装配的——**完全解耦**。

为什么不直接用一个 Map 存？当然也行，但 Spring 容器帮你管理了对象的生命周期，全局可访问不需要到处传 Map 引用，热更新时"先移除再注册"的逻辑也更自然。

## InMemoryRunner 的内部结构

装配流水线的最终产物就是这个 runner。它里面装着 4 样东西：

| 组成部分       | 装配阶段放进去的 | 运行阶段动态产生的 | 作用                              |
| -------------- | ---------------- | ------------------ | --------------------------------- |
| Agent 树       | ✅                |                    | 固定不变的执行蓝图                |
| Session 管理器 |                  | ✅                  | 管理每个用户的对话状态            |
| 应用名         | ✅                |                    | 日志标识 + Session 隔离           |
| 插件列表       | ✅                |                    | 扩展 runner 的能力（demo 里为空） |

"InMemory"指的是 Session 数据存在内存里，应用重启后对话历史会丢失。

### Session 里的 outputKey 机制

这是 Agent 之间传递数据的核心机制。每个 Agent 配置了 `outputKey` 后，执行完毕的输出会被存到 `session.state` 这个 Map 里。后续 Agent 的 instruction 里写 `{request_brief}` 这样的占位符，ADK 框架会自动从 `session.state` 里查找对应的 key 并替换：

```

InputEchoAgent 执行完 → session.state.put("request_brief", "目标：分析AI趋势...")

QuestionDecomposerAgent 即将执行：
  instruction 里有 {request_brief}
  → 从 session.state 取出 "request_brief" 的值
  → 替换占位符后发给 LLM

```

## 五、会话机制：两层管理

```

我们代码管理的（外层）：
┌────────────────────────────────────┐
│  ConcurrentHashMap<用户ID, 会话ID>  │
│                                    │
│  "xiaofuge"  →  "a3f6-..."        │
│  "admin"     →  "b7e2-..."        │
└────────────────────────────────────┘
    ↑ 作用：快速查找用户有没有会话
    ↑ 只存映射关系，不存聊天内容

SDK 内部管理的（内层）：
┌────────────────────────────────────┐
│  SessionService                    │
│                                    │
│  Session "a3f6-..." {              │
│    events: [                       │
│      { role: user,  text: "你好" }  │
│      { role: model, text: "你好！" }│
│      { role: user,  text: "1+1" }  │
│      { role: model, text: "等于2" } │
│    ]                               │
│  }                                 │
└────────────────────────────────────┘
    ↑ 作用：存储真正的对话历史

```

**上下文维持的原理**：每次发起对话传入用户 ID 和会话 ID，SDK 内部根据会话 ID 找到对应的会话对象，取出历史记录和新消息拼在一起发给大模型。大模型的 API 本身是无状态的——所谓的"记忆"就是每次把完整历史重新发一遍。

外层 Map 创建会话时用的是原子操作，保证同一用户并发到达时只创建一次。

**当前方案的局限**：全在内存里，服务一重启就丢了。生产环境需要外层映射换 Redis（高频键值查询），内层会话历史存 MySQL（访问频率不高但数据重要）。还有个隐患是 Map 没有容量上限，用户越来越多会持续膨胀，长时间运行可能 OOM。

## 六、工具调用：注入、触发、执行

三个环节，**我们代码只负责第一个**：

```

【注入】我们的代码 ──────────────────────────────────
  装配时遍历 MCP + Skills
  构建 ToolCallback 列表
  注入到 ChatModel 对象中

【触发】Spring AI 自动完成 ──────────────────────────
  调用大模型时，把工具描述一起发过去
  大模型自己决定是否调用
  如果要调用，返回工具名称 + 参数

【执行】Spring AI 自动完成 ──────────────────────────
  从 ToolCallback 列表中匹配对应工具
  执行工具，拿到结果
  把结果连同之前的消息再发给大模型
  ↑ 这个过程可能多轮循环，直到大模型返回最终回答

```

### MCP 和 Skills 的区别

触发机制完全一样——都是构建 ToolCallback，大模型根据描述决定调用，Spring AI 自动执行。区别在于回调内部做的事：

|              | MCP                              | Skills                          |
| ------------ | -------------------------------- | ------------------------------- |
| 本质         | 通过协议调用独立服务             | 读取本地文件返回内容            |
| 工具逻辑在哪 | 远程服务 / 本地子进程 / JavaBean | 文档 + 脚本，由大模型理解后使用 |
| 类比         | 给智能体配了个**能干活的助手**   | 给智能体发了本**操作手册**      |
| 适用场景     | 重量级（数据库操作、代码执行）   | 轻量级（知识注入、简单脚本）    |

有个容易误解的点：Skills 里的脚本不是大模型自己执行的。大模型读到"你需要执行某个脚本"后，如果有代码执行类的 MCP 工具，它会再发一次工具调用让 MCP 去执行。大模型自己执行不了任何东西。

## 七、多模态：消息构建与格式转换修复

### 7.1 消息构建

用户发多模态消息时，接收到的不再是字符串，而是一个实体类，里面三个列表：

```

ChatCommandEntity
├── texts: ["请分析这张图片"]          → 转为文本类型 Part
├── inlineDatas: [{bytes, mimeType}]  → 转为内联数据类型 Part
└── fileUris: [{url, mimeType}]       → 转为 URI 类型 Part
                                           ↓
                                    合并为一个 Content 消息体

```

Content 和 Part 是包含和被包含的关系。Part 一共 5 种类型：

| Part 类型 | 携带数据            | 谁创建的             |
| --------- | ------------------- | -------------------- |
| 文本      | 纯文本字符串        | 我们的代码           |
| 内联数据  | 字节数组 + 类型标识 | 我们的代码           |
| URI       | 远程地址 + 类型标识 | 我们的代码           |
| 函数调用  | 工具名称 + 参数     | 大模型返回的         |
| 函数响应  | 工具执行结果        | Spring AI 自动生成的 |

Content 有 3 种角色：user（用户消息）、model（模型回复）、system（系统指令/提示词）。

### 7.2 消息转换器的修复

框架默认的消息转换器有个 bug——转换时只处理文本，直接跳过了多媒体数据。强行转换会丢失全部图片信息。

我们的修复策略是**最小侵入**——继承框架的转换器，只重写有问题的那一步：

```

原始消息（包含文本 + 图片）
        │
        ├──① 先把多媒体数据提取出来暂存
        │
        ├──② 调用父类的转换方法（多媒体会丢失）
        │
        ├──③ 把暂存的多媒体数据补回到转换结果的媒体字段
        │
        ▼
Spring AI 格式的消息（文本 + 图片都在）

```

为什么不完全重写？因为父类的转换方法还处理了很多其他逻辑（角色映射、系统指令、历史拼接），全部复制一遍工作量大，而且父类更新时我们不会自动同步。只修复有问题的部分，其他完全复用父类，这样改动范围最小。

## 规则树的四个核心方法

整棵规则树能自动一站接一站跑起来，全靠这 4 个方法的配合。这是最底层的机制，必须先搞明白这个，后面的所有代码才看得懂。

### StrategyHandler 接口（框架层）

框架提供了一个接口，定义了每个节点必须具备的能力：

```java
public interface StrategyHandler<T, D, R> {
    // 对外暴露的执行入口——外界调用这个来启动节点
    R apply(T requestParameter, D dynamicContext) throws Exception;

    // 获取下一个节点——每个节点必须告诉框架"下一站是谁"
    StrategyHandler<T, D, R> get(T requestParameter, D dynamicContext) throws Exception;
}

```

三个泛型参数：`T` 是输入参数类型，`D` 是上下文类型，`R` 是返回结果类型。

### AbstractMultiThreadStrategyRouter 抽象类（框架层）

接口只说了"要做什么"，这个抽象类实现了通用逻辑：

```java
public abstract class AbstractMultiThreadStrategyRouter<T, D, R>
        implements StrategyHandler<T, D, R> {

    // 子类必须实现的"干活"方法
    protected abstract R doApply(T requestParameter, D dynamicContext) throws Exception;

    // apply() 的实现——框架帮你写好了
    @Override
    public R apply(T requestParameter, D dynamicContext) throws Exception {
        return doApply(requestParameter, dynamicContext);
    }

    // ★ router()——找到下一个节点并执行它
    protected R router(T requestParameter, D dynamicContext) throws Exception {
        StrategyHandler<T, D, R> nextHandler = get(requestParameter, dynamicContext);
        if (nextHandler != null) {
            return nextHandler.apply(requestParameter, dynamicContext);
        }
        return null;
    }
}

```

### 四个方法的关系

| 方法        | 谁定义的   | 谁实现的     | 做什么                                    | 一句话记忆       |
| ----------- | ---------- | ------------ | ----------------------------------------- | ---------------- |
| `apply()`   | 接口定义   | **框架实现** | 启动一个节点的执行                        | "启动这个节点"   |
| `doApply()` | 抽象类定义 | **你实现**   | 这个节点的具体业务逻辑                    | "这个节点干什么" |
| `get()`     | 接口定义   | **你实现**   | 返回下一个节点是谁                        | "下一站去哪"     |
| `router()`  | 抽象类定义 | **框架实现** | 调用 get() 获取下一站，再调用它的 apply() | "帮我跳到下一站" |

所以写一个节点只需要写两个方法：`doApply()`（干活）和 `get()`（指路）。在 `doApply()` 的最后一行调用 `return router(参数, 上下文)` 就能自动跳到下一站。如果是最后一个节点，直接 return 结果就行，不调 `router()`。

## 八、流式与非流式

### 8.1 三个层面的区别

| 层面       | 非流式                    | 流式                           |
| ---------- | ------------------------- | ------------------------------ |
| 前端体验   | 等几秒后一次显示全部内容  | 文字像打字机一样逐步出现       |
| 控制器返回 | 普通 JSON 响应            | ResponseBodyEmitter 流式发射器 |
| 底层调用   | Spring AI 同步方法 call() | Spring AI 流式方法 stream()    |

### 8.2 流式发射器的作用

正常的 HTTP 接口 return 之后连接就关闭了。流式发射器告诉 Spring 框架："这个连接先别关，后面还有数据要推"。框架会设置响应头为流式传输格式，保持连接。

超时时间设 3 分钟，是整个连接的总生存上限。复杂的智能体可能要多轮工具调用，一两分钟很正常，3 分钟给了比较宽裕的余量。超过就强制关闭，防止异常情况导致连接资源永远不释放。

### 8.3 runner.runAsync() 的行为

返回的是事件流对象，**延迟执行**——调用后不会立即开始与大模型通信，只有对这个事件流进行订阅时才真正开始。不订阅就不执行，这是响应式编程的特点。

还有个容易混淆的点：**控制层的流式/非流式和底层调用 LLM 的流式/非流式是独立的**。控制层流式指的是返回给前端的方式，底层流式指的是调用大模型 API 的方式。底层大部分情况调的是流式，但也可能是同步——这也是为什么适配器内部即使底层是非流式，也要把响应包装成只有一个事件的事件流来返回，保持接口统一。

## 九、错误处理的现状与不足

当前的错误处理是基础的捕获与传递模型：

```

LLM 调用异常
    │
    ├── 记录到监控组件
    ├── 错误映射（统一格式）
    ├── 包装成 RuntimeException 向上抛
    │
    ├── 非流式 → 返回错误 JSON
    └── 流式 → emitter.completeWithError() 中断连接

```

能用，但在生产环境中有三个明显的不足：

**缺乏重试机制**。网络抖动、大模型限流（429）这些很常见，当前失败一次就直接报错。应该针对可恢复异常做指数退避重试（隔 1s、2s、4s 重试 3 次）。

**缺乏主备降级**。主模型挂了就服务不可用。应该配备用模型，重试无果后自动切换。

**工具调用失败会中断整个对话**。MCP 远程服务挂了，异常一路向上抛，用户直接看到报错。更好的做法是拦截工具异常，把错误信息作为工具的返回值发给大模型，让大模型自己决定是换个工具还是告诉用户"这个服务暂时不可用"——这才是智能体该有的行为。

## 十、多模态 HTTP 接口设计

当前 Controller 层的 `chat` 和 `chatStream` 两个接口，入参 `ChatRequestDTO` 只有纯文本字段。Service 层的多模态方法没有暴露出来。

如果要设计多模态接口，推荐**先上传后引用**的方案：

```

步骤 1：前端调用文件上传接口
POST /api/v1/files/upload  (multipart/form-data)
返回：{ "fileUri": "https://oss.xxx/dog.png", "mimeType": "image/png" }

步骤 2：前端调用对话接口（JSON，不变）
POST /api/v1/chat
{
  "agentId": "100003",
  "userId": "admin",
  "message": "请分析这张图片",
  "files": [
    { "fileUri": "https://oss.xxx/dog.png", "mimeType": "image/png" }
  ]
}

```

为什么不把图片 Base64 编码塞进 JSON？体积膨胀 33%，大 JSON 解析吃内存容易 OOM。为什么不用 multipart 表单？因为现有的 `chat` 和 `chatStream` 两个接口都是 JSON 格式，如果其中一个改成表单格式，前端就要写两套请求逻辑，流式接口（SSE）配合表单上传更是别扭。

先上传后引用的好处是：**两个现有接口都不需要破坏**，只在 DTO 里加一个 files 字段就完成升级，依然是干净的 JSON。大文件传输压力可以剥离给 OSS，不拖垮智能体服务。

## 方案总结

| 设计点                      | 解决的问题                              |
| --------------------------- | --------------------------------------- |
| DDD 分层 + 依赖倒置         | 业务逻辑不受技术实现变化影响            |
| 双框架适配器                | ADK 管编排 + Spring AI 管调用，各取所长 |
| 6 节点责任链                | 装配流程可扩展、可测试、可并行开发      |
| 编排节点逻辑循环 + 动态获取 | 支持嵌套编排，避免循环依赖              |
| 动态 Bean 注册              | 装配与运行完全解耦，支持热替换          |
| 两层会话管理                | 外层快速索引 + 内层存储历史             |
| 消息转换器重写              | 最小侵入修复多模态丢失                  |
| 流式发射器 + 事件流         | 打字机效果 + 统一的异步执行模型         |

> 智能体脚手架的核心思路：**配置驱动 + 职责分离**。把复杂的装配过程拆成独立节点（责任链），把异构框架的差异封装在适配器里（桥接），把运行时的状态交给容器管理（IOC）。每一层只关心自己的事，加起来就是一个完整的智能体。
