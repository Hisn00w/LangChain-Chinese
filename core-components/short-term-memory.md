# 短期记忆

## 概述

记忆是一个记住以前交互信息的系统。对于 AI agent 来说，记忆至关重要，因为它让 agent 能够记住之前的交互、从反馈中学习并适应用户偏好。随着 agent 处理更复杂的任务和大量用户交互，这种能力对于效率和用户满意度都变得必不可少。

短期记忆让您的应用程序记住单个线程或对话中的先前交互。

线程将一个会话中的多次交互组织在一起，类似于电子邮件将消息分组在单个对话中的方式。

对话历史是最常见的短期记忆形式。长对话对当今的 LLM 构成了挑战；完整的历史可能无法容纳在 LLM 的上下文窗口中，导致上下文丢失或错误。

即使您的模型支持完整的上下文长度，大多数 LLM 在长上下文上仍然表现不佳。它们会被过时或离题的内容"分心"，同时还会遭受更慢的响应时间和更高成本的困扰。

聊天模型使用[消息](/oss/python/langchain/messages)接受上下文，其中包括指令（系统消息）和输入（人类消息）。在聊天应用程序中，消息在人类输入和模型响应之间交替，形成一个随时间增长的消息列表。由于上下文窗口有限，许多应用程序可以从使用删除或"遗忘"过时信息的技术中受益。

## 使用方法

要向 agent 添加短期记忆（线程级持久性），需要在创建 agent 时指定 `checkpointer`。

LangChain 的 agent 将短期记忆作为 agent 状态的一部分进行管理。

通过将这些内容存储在图的状态中，agent 可以在给定对话中访问完整上下文，同时保持不同线程之间的分离。

状态使用检查点持久化到数据库（或内存），以便线程可以随时恢复。

当调用 agent 或完成一个步骤（如工具调用）时，短期记忆会更新，并在每个步骤开始时读取状态。

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


agent = create_agent(
    "gpt-5",
    tools=[get_user_info],
    checkpointer=InMemorySaver(),  # [!code highlight]
)

agent.invoke(
    {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
    {"configurable": {"thread_id": "1"}},  # [!code highlight]
)
```

### 生产环境使用

在生产环境中，使用由数据库支持的检查点：

```shell
pip install langgraph-checkpoint-postgres
```

```python
from langchain.agents import create_agent

from langgraph.checkpoint.postgres import PostgresSaver  # [!code highlight]


DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup() # auto create tables in PostgresSql
    agent = create_agent(
        "gpt-5",
        tools=[get_user_info],
        checkpointer=checkpointer,  # [!code highlight]
    )
```

## 自定义 agent 记忆

默认情况下，agent 使用 [`AgentState`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.AgentState) 来管理短期记忆，具体是通过 `messages` 键管理对话历史。

您可以扩展 [`AgentState`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.AgentState) 以添加其他字段。自定义状态模式使用 [`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema) 参数传递给 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent)。

```python
from langchain.agents import create_agent, AgentState
from langgraph.checkpoint.memory import InMemorySaver


class CustomAgentState(AgentState):  # [!code highlight]
    user_id: str  # [!code highlight]
    preferences: dict  # [!code highlight]

agent = create_agent(
    "gpt-5",
    tools=[get_user_info],
    state_schema=CustomAgentState,  # [!code highlight]
    checkpointer=InMemorySaver(),
)

# 自定义状态可以在 invoke 中传递
result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "Hello"}],
        "user_id": "user_123",  # [!code highlight]
        "preferences": {"theme": "dark"}  # [!code highlight]
    },
    {"configurable": {"thread_id": "1"}})
```

## 常见模式

启用[短期记忆](#add-short-term-memory)后，长对话可能会超过 LLM 的上下文窗口。常见的解决方案是：

- **修剪消息** - 删除前 N 条或后 N 条消息（在调用 LLM 之前）
- **删除消息** - 从 LangGraph 状态中永久删除消息
- **总结消息** - 总结历史中较早的消息并用摘要替换它们
- **自定义策略** - 自定义策略（例如消息过滤等）

这允许 agent 跟踪对话而不会超过 LLM 的上下文窗口。

### 修剪消息

大多数 LLM 有最大支持的上下文窗口（以 token 为单位）。

决定何时截断消息的一种方法是计算消息历史中的 token 数，并在接近该限制时进行截断。如果您使用 LangChain，可以使用修剪消息工具并指定要从列表中保留的 token 数量，以及用于处理边界的 `strategy`（例如保留最后 `max_tokens`）。

要修剪 agent 中的消息历史，请使用 [`@before_model`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.before_model) 中间件装饰器：

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import before_model
from langgraph.runtime import Runtime
from langchain_core.runnables import RunnableConfig
from typing import Any


@before_model
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """只保留最后几条消息以适应上下文窗口。"""
    messages = state["messages"]

    if len(messages) <= 3:
        return None  # 不需要更改

    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages

    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),
            *new_messages
        ]
    }

agent = create_agent(
    your_model_here,
    tools=your_tools_here,
    middleware=[trim_messages],
    checkpointer=InMemorySaver(),
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob. You told me that earlier.
If you'd like me to call you a nickname or use a different name, just say the word.
"""
```

### 删除消息

您可以从图状态中删除消息以管理消息历史。

当您想要删除特定消息或清除整个消息历史时，这很有用。

要从图状态中删除消息，可以使用 `RemoveMessage`。

要使 `RemoveMessage` 工作，您需要使用带有 [`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages) [reducer](/oss/python/langgraph/graph-api#reducers) 的状态键。

默认的 [`AgentState`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.AgentState) 提供了这个功能。

删除特定消息：

```python
from langchain.messages import RemoveMessage  # [!code highlight]

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # 删除最早的两条消息
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}  # [!code highlight]
```

删除**所有**消息：

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES  # [!code highlight]

def delete_messages(state):
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}  # [!code highlight]
```

删除消息时，**确保**生成的消息历史是有效的。检查您使用的 LLM 提供商的限制。例如：

* 有些提供商期望消息历史以 `user` 消息开始
* 大多数提供商要求带有工具调用的 `assistant` 消息后面必须跟有相应的 `tool` 结果消息。

```python
from langchain.messages import RemoveMessage
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import after_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.runtime import Runtime
from langchain_core.runnables import RunnableConfig


@after_model
def delete_old_messages(state: AgentState, runtime: Runtime) -> dict | None:
    """删除旧消息以保持对话可管理。"""
    messages = state["messages"]
    if len(messages) > 2:
        # 删除最早的两条消息
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
    return None


agent = create_agent(
    "gpt-5-nano",
    tools=[],
    system_prompt="Please be concise and to the point.",
    middleware=[delete_old_messages],
    checkpointer=InMemorySaver(),
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

for event in agent.stream(
    {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
    config,
    stream_mode="values",
):
    print([(message.type, message.content) for message in event["messages"]])

for event in agent.stream(
    {"messages": [{"role": "user", "content": "what's my name?"}]},
    config,
    stream_mode="values",
):
    print([(message.type, message.content) for message in event["messages"]])
```

```
[('human', "hi! I'm bob")]
[('human', "hi! I'm bob"), ('ai', 'Hi Bob! Nice to meet you. How can I help you today? I can answer questions, brainstorm ideas, draft text, explain things, or help with code.')]
[('human', "hi! I'm bob"), ('ai', 'Hi Bob! Nice to meet you. How can I help you today? I can answer questions, brainstorm ideas, draft text, explain things, or help with code.'), ('human', "what's my name?")]
[('human', "hi! I'm bob"), ('ai', 'Hi Bob! Nice to meet you. How can I help you today? I can answer questions, brainstorm ideas, draft text, explain things, or help with code.'), ('human', "what's my name?"), ('ai', 'Your name is Bob. How can I help you today, Bob?')]
[('human', "what's my name?"), ('ai', 'Your name is Bob. How can I help you today, Bob?')]
```

### 总结消息

如上所示，修剪或删除消息的问题在于您可能会从消息队列的删除中丢失信息。因此，一些应用程序受益于使用聊天模型总结消息历史这种更复杂的方法。

![Summary](https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png)

要在 agent 中总结消息历史，请使用内置的 [`SummarizationMiddleware`](/oss/python/langchain/middleware#summarization)：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.runnables import RunnableConfig


checkpointer = InMemorySaver()

agent = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4o-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20)
        )
    ],
    checkpointer=checkpointer,
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}
agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob!
"""
```

有关更多配置选项，请参阅 [`SummarizationMiddleware`](/oss/python/langchain/middleware#summarization)。

## 访问记忆

您可以通过多种方式访问和修改 agent 的短期记忆（状态）：

### 工具

#### 在工具中读取短期记忆

使用 `runtime` 参数（在工具中类型化为 `ToolRuntime`）访问短期记忆（状态）。

`runtime` 参数在工具签名中是隐藏的（因此模型看不到它），但工具可以通过它访问状态。

```python
from langchain.agents import create_agent, AgentState
from langchain.tools import tool, ToolRuntime


class CustomState(AgentState):
    user_id: str

@tool
def get_user_info(
    runtime: ToolRuntime
) -> str:
    """查找用户信息。"""
    user_id = runtime.state["user_id"]
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_user_info],
    state_schema=CustomState,
)

result = agent.invoke({
    "messages": "look up user information",
    "user_id": "user_123"
})
print(result["messages"][-1].content)
# > User is John Smith.
```

#### 从工具写入短期记忆

要在执行期间修改 agent 的短期记忆（状态），可以直接从工具返回状态更新。

这对于持久化中间结果或使信息对后续工具或提示可用很有用。

```python
from langchain.tools import tool, ToolRuntime
from langchain_core.runnables import RunnableConfig
from langchain.messages import ToolMessage
from langchain.agents import create_agent, AgentState
from langgraph.types import Command
from pydantic import BaseModel


class CustomState(AgentState):  # [!code highlight]
    user_name: str

class CustomContext(BaseModel):
    user_id: str

@tool
def update_user_info(
    runtime: ToolRuntime[CustomContext, CustomState],
) -> Command:
    """查找并更新用户信息。"""
    user_id = runtime.context.user_id
    name = "John Smith" if user_id == "user_123" else "Unknown user"
    return Command(update={  # [!code highlight]
        "user_name": name,
        # 更新消息历史
        "messages": [
            ToolMessage(
                "Successfully looked up user information",
                tool_call_id=runtime.tool_call_id
            )
        ]
    })

@tool
def greet(
    runtime: ToolRuntime[CustomContext, CustomState]
) -> str | Command:
    """找到用户信息后使用此工具向用户打招呼。"""
    user_name = runtime.state.get("user_name", None)
    if user_name is None:
       return Command(update={
            "messages": [
                ToolMessage(
                    "Please call the 'update_user_info' tool it will get and update the user's name.",
                    tool_call_id=runtime.tool_call_id
                )
            ]
        })
    return f"Hello {user_name}!"

agent = create_agent(
    model="gpt-5-nano",
    tools=[update_user_info, greet],
    state_schema=CustomState, # [!code highlight]
    context_schema=CustomContext,
)

agent.invoke(
    {"messages": [{"role": "user", "content": "greet the user"}]},
    context=CustomContext(user_id="user_123"),
)
```

### 提示

在中间件中访问短期记忆（状态）以基于对话历史或自定义状态字段创建动态提示。

```python
from langchain.agents import create_agent
from typing import TypedDict
from langchain.agents.middleware import dynamic_prompt, ModelRequest


class CustomContext(TypedDict):
    user_name: str


def get_weather(city: str) -> str:
    """获取城市的天气。"""
    return f"The weather in {city} is always sunny!"


@dynamic_prompt
def dynamic_system_prompt(request: ModelRequest) -> str:
    user_name = request.runtime.context["user_name"]
    system_prompt = f"You are a helpful assistant. Address the user as {user_name}."
    return system_prompt


agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
    middleware=[dynamic_system_prompt],
    context_schema=CustomContext,
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    context=CustomContext(user_name="John Smith"),
)
for msg in result["messages"]:
    msg.pretty_print()

```

```shell
================================ Human Message =================================

What is the weather in SF?
================================== Ai Message ==================================
Tool Calls:
  get_weather (call_WFQlOGn4b2yoJrv7cih342FG)
 Call ID: call_WFQlOGn4b2yoJrv7cih342FG
  Args:
    city: San Francisco
================================= Tool Message =================================
Name: get_weather

The weather in San Francisco is always sunny!
================================== Ai Message ==================================

Hi John Smith, the weather in San Francisco is always sunny!
```

### 在模型之前

在 [`@before_model`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.before_model) 中间件中访问短期记忆（状态）以在模型调用之前处理消息。

```mermaid
%%{
    init: {
        "fontFamily": "monospace",
        "flowchart": {
        "curve": "basis"
        },
        "themeVariables": {"edgeLabelBackground": "transparent"}
    }
}%%
graph TD
    S(["\_\_start\_\_"])
    PRE(before_model)
    MODEL(model)
    TOOLS(tools)
    END(["\_\_end\_\_"])
    S --> PRE
    PRE --> MODEL
    MODEL -.-> TOOLS
    MODEL -.-> END
    TOOLS --> PRE
    classDef blueHighlight fill:#0a1c25,stroke:#0a455f,color:#bae6fd;
    class S blueHighlight;
    class END blueHighlight;
```

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import before_model
from langchain_core.runnables import RunnableConfig
from langgraph.runtime import Runtime
from typing import Any


@before_model
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """只保留最后几条消息以适应上下文窗口。"""
    messages = state["messages"]

    if len(messages) <= 3:
        return None  # 不需要更改

    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages

    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),
            *new_messages
        ]
    }


agent = create_agent(
    "gpt-5-nano",
    tools=[],
    middleware=[trim_messages],
    checkpointer=InMemorySaver()
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob. You told me that earlier.
If you'd like me to call you a nickname or use a different name, just say the word.
"""
```

### 在模型之后

在 [`@after_model`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.after_model) 中间件中访问短期记忆（状态）以在模型调用之后处理消息。

```mermaid
%%{
    init: {
        "fontFamily": "monospace",
        "flowchart": {
        "curve": "basis"
        },
        "themeVariables": {"edgeLabelBackground": "transparent"}
    }
}%%
graph TD
    S(["\_\_start\_\_"])
    MODEL(model)
    POST(after_model)
    TOOLS(tools)
    END(["\_\_end\_\_"])
    S --> MODEL
    MODEL --> POST
    POST -.-> END
    POST -.-> TOOLS
    TOOLS --> MODEL
    classDef blueHighlight fill:#0a1c25,stroke:#0a455f,color:#bae6fd;
    class S blueHighlight;
    class END blueHighlight;
    class POST greenHighlight;
```

```python
from langchain.messages import RemoveMessage
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import after_model
from langgraph.runtime import Runtime


@after_model
def validate_response(state: AgentState, runtime: Runtime) -> dict | None:
    """删除包含敏感词的消息。"""
    STOP_WORDS = ["password", "secret"]
    last_message = state["messages"][-1]
    if any(word in last_message.content for word in STOP_WORDS):
        return {"messages": [RemoveMessage(id=last_message.id)]}
    return None

agent = create_agent(
    model="gpt-5-nano",
    tools=[],
    middleware=[validate_response],
    checkpointer=InMemorySaver(),
)
```

