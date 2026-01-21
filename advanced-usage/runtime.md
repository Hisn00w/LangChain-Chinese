# 运行时

## 概述

LangChain 的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 在底层 LangGraph 的运行时上运行。

LangGraph 暴露了一个 [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/#langgraph.runtime.Runtime) 对象，包含以下信息：

1. **上下文**：静态信息，如用户 ID、数据库连接或 agent 调用依赖项
2. **存储**：用于[长期记忆](/oss/python/langchain/long-term-memory) 的 [BaseStore](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore) 实例
3. **流写入器**：通过 `"custom"` 流模式流式传输信息的对象

运行时上下文为您的工具和中间件提供**依赖注入**。无需硬编码值或使用全局状态，您可以在调用 agent 时注入运行时依赖项（如数据库连接、用户 ID 或配置）。这使您的工具更易于测试、可重用和灵活。

您可以在[工具](#在工具内部)和[中间件](#在中间件内部)中访问运行时信息。

## 访问

使用 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 创建 agent 时，您可以指定 `context_schema` 来定义 agent [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/#langgraph.runtime.Runtime) 中存储的 `context` 的结构。

调用 agent 时，传递 `context` 参数并包含运行的相关配置：

```python
from dataclasses import dataclass

from langchain.agents import create_agent


@dataclass
class Context:
    user_name: str

agent = create_agent(
    model="gpt-5-nano",
    tools=[...],
    context_schema=Context
)

agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    context=Context(user_name="John Smith")
)
```

### 在工具内部

您可以在工具内部访问运行时信息以：

- 访问上下文
- 读取或写入长期记忆
- 写入[自定义流](/oss/python/langchain/streaming#custom-updates)（例如，工具进度/更新）

使用 `ToolRuntime` 参数在工具内部访问 [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/#langgraph.runtime.Runtime) 对象。

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime

@dataclass
class Context:
    user_id: str

@tool
def fetch_user_email_preferences(runtime: ToolRuntime[Context]) -> str:
    """从存储中获取用户的电子邮件偏好。"""
    user_id = runtime.context.user_id

    preferences: str = "The user prefers you to write a brief and polite email."
    if runtime.store:
        if memory := runtime.store.get(("users",), user_id):
            preferences = memory.value["preferences"]

    return preferences
```

### 在中间件内部

您可以在中间件中访问运行时信息，以创建动态提示、修改消息，或根据用户上下文控制 agent 行为。

使用 `request.runtime` 在中间件装饰器中访问 [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/#langgraph.runtime.Runtime) 对象。运行时对象在传递给中间件函数的 [`ModelRequest`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ModelRequest) 参数中可用。

```python
from dataclasses import dataclass

from langchain.messages import AnyMessage
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import dynamic_prompt, ModelRequest, before_model, after_model
from langgraph.runtime import Runtime


@dataclass
class Context:
    user_name: str

# 动态提示
@dynamic_prompt
def dynamic_system_prompt(request: ModelRequest) -> str:
    user_name = request.runtime.context.user_name
    system_prompt = f"You are a helpful assistant. Address the user as {user_name}."
    return system_prompt

# 模型之前的钩子
@before_model
def log_before_model(state: AgentState, runtime: Runtime[Context]) -> dict | None:
    print(f"Processing request for user: {runtime.context.user_name}")
    return None

# 模型之后的钩子
@after_model
def log_after_model(state: AgentState, runtime: Runtime[Context]) -> dict | None:
    print(f"Completed request for user: {runtime.context.user_name}")
    return None

agent = create_agent(
    model="gpt-5-nano",
    tools=[...],
    middleware=[dynamic_system_prompt, log_before_model, log_after_model],
    context_schema=Context
)

agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    context=Context(user_name="John Smith")
)
```

