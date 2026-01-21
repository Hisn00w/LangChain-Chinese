# 更新日志

> Python 包的更新和改进记录

您可以通过 RSS 订阅我们的更新日志：[RSS feed](https://docs.langchain.com/oss/python/releases/changelog/rss.xml)

---

## `langchain` v1.2.0（2025年12月15日）

* [`create_agent`](/oss/python/langchain/agents)：通过工具上的新 [`extras`](https://reference.langchain.com/python/langchain/tools/#langchain.tools.BaseTool.extras) 属性，简化了对特定提供商工具参数和定义的支持。示例：
  * 特定于提供商的配置，如 Anthropic 的[程序化工具调用](/oss/python/integrations/chat/anthropic#programmatic-tool-calling)和[工具搜索](/oss/python/integrations/chat/anthropic#tool-search)。
  * 在客户端执行的内置工具，支持 [Anthropic](/oss/python/integrations/chat/anthropic#built-in-tools)、[OpenAI](/oss/python/integrations/chat/openai#responses-api) 和其他提供商。
* 支持智能体 `response_format` 的严格模式适配（参见 [`ProviderStrategy`](/oss/python/langchain/structured-output#provider-strategy) 文档）。

## `langchain-google-genai` v4.0.0（2025年12月8日）

我们重写了 Google GenAI 集成，使用 Google 统一的 Generative AI SDK，该 SDK 在同一接口下提供对 Gemini API 和 Vertex AI Platform 的访问。这包括对 `langchain-google-vertexai` 中已弃用包的最小破坏性更改。

详见完整的[发布说明和迁移指南](https://github.com/langchain-ai/langchain-google/discussions/1422)。

## `langchain` v1.1.0（2025年11月25日）

* [模型配置](/oss/python/langchain/models#model-profiles)：聊天模型现在通过 `.profile` 属性公开支持的功能和能力。这些数据来自 [models.dev](https://models.dev)，这是一个提供模型能力数据的开源项目。
* [摘要中间件](/oss/python/langchain/middleware/built-in#summarization)：更新为支持使用模型配置进行上下文感知摘要的灵活触发点。
* [结构化输出](/oss/python/langchain/structured-output)：`ProviderStrategy` 支持（原生的结构化输出）现在可以从模型配置推断。
* [`create_agent` 的 `SystemMessage`](/oss/python/langchain/middleware/custom#working-with-system-messages)：支持将 `SystemMessage` 实例直接传递给 `create_agent` 的 `system_prompt` 参数，启用缓存控制和结构化内容块等高级功能。
* [模型重试中间件](/oss/python/langchain/middleware/built-in#model-retry)：用于使用可配置的指数退避自动重试失败模型调用的新中间件。
* [内容审核中间件](/oss/python/langchain/middleware/built-in#content-moderation)：OpenAI 内容审核中间件，用于检测和处理智能体交互中的不安全内容。支持检查用户输入、模型输出和工具结果。

## v1.0.0（2025年10月20日）

### `langchain`

* [发布说明](/oss/python/releases/langchain-v1)
* [迁移指南](/oss/python/migrate/langchain-v1)

### `langgraph`

* [发布说明](/oss/python/releases/langgraph-v1)
* [迁移指南](/oss/python/migrate/langgraph-v1)

如果您遇到任何问题或有反馈，请[提交问题](https://github.com/langchain-ai/docs/issues/new?template=01-langchain.yml)以便我们改进。要查看 v0.x 文档，请[转到存档内容](https://github.com/langchain-ai/langchain/tree/v0.3/docs/docs)和 [API 参考](https://reference.langchain.com/v0.3/python/)。

---

