# Philosophy



> LangChain 的目标是：成为构建 LLM 应用最容易上手的起点，同时也要足够灵活、可用于生产环境。

LangChain 的设计由一些核心信念驱动：

- 大语言模型（LLM）是一项强大且全新的技术。
- 当你把 LLM 与外部数据源结合时，它会更强大。
- LLM 将改变未来应用的形态。更具体地说，未来的应用会越来越“智能体化（agentic）”。
- 我们仍处在这场变革的非常早期阶段。
- 做出一个“智能体应用”的原型很容易，但要把一个**可靠到可以上线**的 Agent 做出来仍然非常难。

在 LangChain 中，我们有两个核心关注点：

## 两个核心关注点

### 1) 让开发者可以使用最好的模型构建应用

不同的模型提供商暴露出不同的 API：模型参数不同、消息格式不同。  
对这些输入输出进行标准化是我们的核心工作之一——这样开发者就可以更容易地切换到最新的 SOTA 模型，避免被单一厂商锁定。

### 2) 让模型更容易编排复杂流程，并与数据与计算交互

模型不应该只用来做*文本生成*，还应被用于编排更复杂的流程，与其它数据与计算系统交互。  
LangChain 让你可以很容易地定义 LLM 可动态调用的 [tools](../core-components/tools.md)，并帮助你解析与访问非结构化数据。

---

## History

由于这一领域变化极快，LangChain 也在持续演进。下面是一条简短的时间线，展示 LangChain 如何随着 “用 LLM 构建应用” 这件事一起变化：

### 2022-10-24（v0.0.1）

在 ChatGPT 发布前一个月，**LangChain 以 Python 包的形式发布**。它包含两个主要部分：

- LLM 抽象（LLM abstractions）
- “Chains”（链）：为常见用例预先定义好的计算步骤。例如 RAG：先检索，再生成。

“LangChain” 这个名字来自 “Language（语言模型）” 和 “Chains”。

### 2022-12

LangChain 加入了第一批通用 Agent。

这些通用 Agent 基于 [ReAct 论文](https://arxiv.org/abs/2210.03629)（ReAct 即 Reasoning and Acting）。  
它们使用 LLM 生成表示工具调用的 JSON，然后解析 JSON 来决定调用哪些工具。

### 2023-01

OpenAI 发布 “Chat Completion” API。

在此之前，模型通常接收一个字符串并返回一个字符串。ChatCompletions API 变成接收一组 messages 并返回一条 message。  
其它模型提供商也跟进，LangChain 随之更新为支持消息列表。

### 2023-01

LangChain 发布 JavaScript 版本。

LLM 与 Agent 会改变应用的构建方式，而 JavaScript 是应用开发者最常用的语言。

### 2023-02

围绕开源 LangChain 项目，**LangChain Inc. 公司成立**。

主要目标是 “让智能体无处不在（make intelligent agents ubiquitous）”。  
团队认识到：虽然 LangChain 能让你轻松开始用 LLM，但要真正把 Agent 做到可靠可用，还需要更多组件。

### 2023-03

OpenAI 在 API 中发布 “function calling”。

这允许 API 直接生成表示工具调用的 payload。其它模型提供商也跟进，LangChain 更新为优先使用这种方式做工具调用（而不是解析 JSON）。

### 2023-06

**LangSmith 发布**（闭源平台），提供可观测性与评测（observability & evals）。

构建 Agent 的最大难点之一是可靠性。LangSmith 旨在解决这个问题，因此 LangChain 也更新为可与 LangSmith 无缝集成。

### 2024-01（v0.1.0）

**LangChain 发布 0.1.0**，首次脱离 0.0.x 版本序列。

行业从原型阶段走向生产阶段，因此 LangChain 更加关注稳定性。

### 2024-02

**LangGraph 发布**（开源库）。

最初的 LangChain 重点在两件事：LLM 抽象、以及用于常见应用的高层接口；但缺少一个“低层编排层”，让开发者能够精确控制 Agent 的执行流程。于是出现了 LangGraph。

在构建 LangGraph 的过程中，我们基于 LangChain 的经验加入了更多生产所需能力：流式输出、持久化执行、短期记忆、人类介入等。

### 2024-06

**LangChain 集成数超过 700**。

集成能力从核心 `langchain` 包中拆分出来：核心集成变成独立包，其它集成则进入 `langchain-community`。

### 2024-10

LangGraph 成为构建“超过一次 LLM 调用”的 AI 应用的首选方式。

开发者为提升应用可靠性，需要比高层接口更多的控制能力。LangGraph 提供了这种低层灵活性。  
因此，LangChain 中的大多数 chains 和 agents 被标记为 deprecated，并提供迁移到 LangGraph 的指南。  
同时，LangGraph 仍保留了一个高层抽象：**agent abstraction**。它构建在低层 LangGraph 之上，并与 LangChain 的 ReAct agents 有相同接口。

### 2025-04

模型 API 变得更“多模态”。

模型开始接收文件、图片、视频等输入。我们相应更新了 `langchain-core` 的消息格式，使开发者可以用统一标准表达多模态输入。

### 2025-10-20（v1.0.0）

**LangChain 发布 1.0**，带来两项重大变化：

1. 彻底重做 `langchain` 中的 chains 与 agents：  
   现在所有 chains/agents 都被替换为唯一的高层抽象：一个构建在 LangGraph 之上的 agent abstraction。  
   这个高层抽象最初是在 LangGraph 中创建的，后来迁移到 LangChain。

   对于仍在使用旧 LangChain chains/agents 且**不想升级**的用户（注：官方建议升级），可以通过安装 `langchain-classic` 包继续使用旧版本。

2. 标准化的消息内容格式：  
   模型 API 从“返回 content 字符串”演进为更复杂的输出（推理块、引用、服务端工具调用等）。  
   LangChain 也相应更新消息格式，用统一标准适配不同提供商的这些能力。


