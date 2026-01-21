# LangSmith 可观测性

当您使用 LangChain 构建和运行智能体时，您需要了解它们的行为：它们调用哪些[工具(https://docs.langchain.com/oss/python/langchain/tools)、生成什么提示，以及如何做出决策。使用 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 构建的 LangChain 智能体自动支持通过 [LangSmith(https://docs.smith.langchain.com/home) 进行追踪，这是一个用于捕获、调试、评估和监控 LLM 应用程序行为的平台。

[*追踪*(https://docs.smith.langchain.com/observability-concepts#traces) 记录您智能体执行的每个步骤，从初始用户输入到最终响应，包括所有工具调用、模型交互和决策点。这些执行数据帮助您调试问题、评估不同输入的性能，并监控生产中的使用模式。

本指南向您展示如何为您的 LangChain 智能体启用追踪并使用 LangSmith 分析其执行。

## 前置条件

在开始之前，请确保您具备以下条件：

* **一个 LangSmith 账户**：在 [smith.langchain.com](https://smith.langchain.com) 注册（免费）或登录。
* **一个 LangSmith API 密钥**：请遵循[创建 API 密钥(https://docs.smith.langchain.com/create-account-api-key#create-an-api-key)指南。

## 启用追踪

所有 LangChain 智能体自动支持 LangSmith 追踪。要启用它，请设置以下环境变量：

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<your-api-key>
```

## 快速开始

无需额外代码即可将追踪记录到 LangSmith。只需像平常一样运行您的智能体代码：

```python
from langchain.agents import create_agent


def send_email(to: str, subject: str, body: str):
    """向收件人发送电子邮件。"""
    # ... 发送电子邮件逻辑
    return f"Email sent to {to}"

def search_web(query: str):
    """在网上搜索信息。"""
    # ... 网络搜索逻辑
    return f"Search results for: {query}"

agent = create_agent(
    model="gpt-4o",
    tools=[send_email, search_web],
    system_prompt="您是一个可以发送电子邮件和搜索网络的助手。"
)

# 运行智能体 - 所有步骤将自动被追踪
response = agent.invoke({
    "messages": [{"role": "user", "content": "搜索最新的 AI 新闻并向 john@example.com 发送摘要"}]
})
```

默认情况下，追踪将记录到名为 `default` 的项目。要配置自定义项目名称，请参阅[记录到项目](#记录到项目)。

## 选择性追踪

您可以使用 LangSmith 的 `tracing_context` 上下文管理器选择性地追踪特定调用或应用程序部分：

```python
import langsmith as ls

# 这将被追踪
with ls.tracing_context(enabled=True):
    agent.invoke({"messages": [{"role": "user", "content": "向 alice@example.com 发送测试电子邮件"}]})

# 这不会被追踪（如果未设置 LANGSMITH_TRACING）
agent.invoke({"messages": [{"role": "user", "content": "发送另一封电子邮件"}]})
```

## 记录到项目

**静态设置：**

您可以通过设置 `LANGSMITH_PROJECT` 环境变量为整个应用程序设置自定义项目名称：

```bash
export LANGSMITH_PROJECT=my-agent-project
```

**动态设置：**

您可以为特定操作以编程方式设置项目名称：

```python
import langsmith as ls

with ls.tracing_context(project_name="email-agent-test", enabled=True):
    response = agent.invoke({
        "messages": [{"role": "user", "content": "发送欢迎电子邮件"}]
    })
```

## 向追踪添加元数据

您可以使用自定义元数据和标签注释您的追踪：

```python
response = agent.invoke(
    {"messages": [{"role": "user", "content": "发送欢迎电子邮件"}]},
    config={
        "tags": ["production", "email-assistant", "v1.0"],
        "metadata": {
            "user_id": "user_123",
            "session_id": "session_456",
            "environment": "production"
        }
    }
)
```

`tracing_context` 也接受标签和元数据以进行细粒度控制：

```python
with ls.tracing_context(
    project_name="email-agent-test",
    enabled=True,
    tags=["production", "email-assistant", "v1.0"],
    metadata={"user_id": "user_123", "session_id": "session_456", "environment": "production"}):
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "发送欢迎电子邮件"}]}
    )
```

此自定义元数据和标签将附加到 LangSmith 中的追踪。

要了解有关如何使用追踪来调试、评估和监控您的智能体的更多信息，请参阅 [LangSmith 文档](https://docs.langchain.com/langsmith/home)。



