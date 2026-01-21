
# 快速入门


---

本快速入门指南将带你在**几分钟内**，从一个最简单的配置，构建出一个**功能完整的 AI Agent**。

> 💡 **LangChain Docs MCP Server**
>
> 如果你正在使用 AI 编程助手或 IDE（例如 Claude Code、Cursor 等），建议安装  
> [LangChain Docs MCP server](https://docs.langchain.com/use-these-docs)。  
> 这样可以确保你的 Agent 能访问**最新的 LangChain 文档与示例**，获得更好的使用体验。

---

## 前置要求

在开始之前，你需要完成以下准备工作：

- 安装 LangChain（见 [Install](install.md)）
- 注册一个 [Claude（Anthropic）](https://www.anthropic.com/) 账号并获取 API Key
- 在终端中设置环境变量 `ANTHROPIC_API_KEY`

尽管下面的示例使用的是 Claude，你也可以通过更换模型名称并配置对应的 API Key，来使用  
[任意受支持的模型](https://docs.langchain.com/oss/python/integrations/providers/overview)。

---

## 构建一个基础 Agent

我们先从一个**最简单的 Agent**开始。  
这个 Agent 可以回答问题，并调用工具：

- 使用 **Claude Sonnet 4.5** 作为语言模型
- 使用一个简单的天气函数作为工具
- 使用一个基础的 system prompt 引导 Agent 行为

```python
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
````

> 💡 如果你想学习如何使用 LangSmith 对 Agent 进行追踪与调试，
> 请参阅：[LangSmith 文档](https://docs.langchain.com/langsmith/trace-with-langchain)

---

## 构建生产级 Agent

接下来，我们将构建一个**更接近真实生产场景的天气预报 Agent**，并涵盖以下关键概念：

1. 更详细的 **system prompt**
2. **工具（Tools）** 与外部数据的集成
3. **模型配置**，以获得更稳定的输出
4. **结构化输出（Structured output）**
5. **对话记忆（Conversational memory）**
6. 创建并运行一个完整的 Agent

下面我们一步一步来。

---

### 步骤 1：定义系统提示词

System prompt 用于定义 Agent 的角色与行为，应尽量具体、可执行：

```python
SYSTEM_PROMPT = """You are an expert weather forecaster, who speaks in puns.

You have access to two tools:

- get_weather_for_location: use this to get the weather for a specific location
- get_user_location: use this to get the user's location

If a user asks you for the weather, make sure you know the location.
If you can tell from the question that they mean wherever they are,
use the get_user_location tool to find their location."""
```

---

### 步骤 2：创建工具

[Tools](../core-components/tools.md) 允许模型通过调用你定义的函数，与外部系统交互。
工具可以依赖 [runtime context](../advanced-usage/runtime.md)，也可以与
[agent memory](../core-components/short-term-memory.md) 结合使用。

下面示例展示了 `get_user_location` 如何使用 runtime context：

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime

@tool
def get_weather_for_location(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

@dataclass
class Context:
    """Custom runtime context schema."""
    user_id: str

@tool
def get_user_location(runtime: ToolRuntime[Context]) -> str:
    """Retrieve user information based on user ID."""
    user_id = runtime.context.user_id
    return "Florida" if user_id == "1" else "SF"
```

> 💡 工具需要良好的文档说明：
> 工具名、描述、参数名都会成为模型 prompt 的一部分。
> LangChain 的 `@tool` 装饰器会自动添加这些元数据，并支持通过 `ToolRuntime` 注入运行时上下文。

---

### 步骤 3：配置模型

为你的用例配置合适的 [语言模型](../core-components/models.md) 参数：

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    "claude-sonnet-4-5-20250929",
    temperature=0.5,
    timeout=10,
    max_tokens=1000
)
```

不同模型与提供商支持的参数可能不同，请参考对应的参考文档。

---

### 步骤 4：定义响应格式

如果你需要 Agent 的输出符合固定结构，可以定义**结构化响应格式**：

```python
from dataclasses import dataclass

# 使用 dataclass；同样也支持 Pydantic
@dataclass
class ResponseFormat:
    """Response schema for the agent."""
    # 必须返回的双关语回答
    punny_response: str
    # 可选的天气补充信息
    weather_conditions: str | None = None
```

---

### 步骤 5：添加记忆

为 Agent 添加 [记忆能力](../core-components/short-term-memory.md)，以在多轮对话中保持上下文状态：

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
```

> ℹ️ 在生产环境中，应使用**持久化的 checkpointer**（例如数据库）。
> 详见：[Add and manage memory](https://docs.langchain.com/oss/python/langgraph/add-memory#manage-short-term-memory)

---

### 步骤 6：创建并运行 Agent

现在，将所有组件组合起来，创建并运行一个完整的 Agent：

```python
from langchain.agents.structured_output import ToolStrategy

agent = create_agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=[get_user_location, get_weather_for_location],
    context_schema=Context,
    response_format=ToolStrategy(ResponseFormat),
    checkpointer=checkpointer
)

# thread_id 用于标识一次对话
config = {"configurable": {"thread_id": "1"}}

response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather outside?"}]},
    config=config,
    context=Context(user_id="1")
)

print(response["structured_response"])
```

你可以使用相同的 `thread_id` 继续对话：

```python
response = agent.invoke(
    {"messages": [{"role": "user", "content": "thank you!"}]},
    config=config,
    context=Context(user_id="1")
)

print(response["structured_response"])
```

---

## 完整示例代码

> 以下是将上述所有步骤整合在一起的完整示例代码：

```python
from dataclasses import dataclass

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.tools import tool, ToolRuntime
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents.structured_output import ToolStrategy


SYSTEM_PROMPT = """You are an expert weather forecaster, who speaks in puns.

You have access to two tools:

- get_weather_for_location
- get_user_location
"""

@dataclass
class Context:
    user_id: str

@tool
def get_weather_for_location(city: str) -> str:
    return f"It's always sunny in {city}!"

@tool
def get_user_location(runtime: ToolRuntime[Context]) -> str:
    return "Florida" if runtime.context.user_id == "1" else "SF"

model = init_chat_model(
    "claude-sonnet-4-5-20250929",
    temperature=0
)

@dataclass
class ResponseFormat:
    punny_response: str
    weather_conditions: str | None = None

checkpointer = InMemorySaver()

agent = create_agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=[get_user_location, get_weather_for_location],
    context_schema=Context,
    response_format=ToolStrategy(ResponseFormat),
    checkpointer=checkpointer
)
```

---

## 恭喜你 🎉

现在你已经构建了一个能够：

* **理解上下文并记住对话**
* **智能使用多个工具**
* **以结构化格式输出结果**
* **处理用户上下文信息**
* **在多轮对话中保持状态**

的 AI Agent。


