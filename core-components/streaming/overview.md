# 概述

> 从智能体运行中流式传输实时更新

LangChain 实现了一个流式传输系统，用于展示实时更新。

流式传输对于增强基于 LLM 构建的应用程序的响应能力至关重要。通过在完整响应准备好之前逐步显示输出，流式传输显著改善了用户体验（UX），特别是在处理 LLM 的延迟时。

## 概述

LangChain 的流式传输系统让您可以将智能体运行的实时反馈展示到您的应用程序中。

使用 LangChain 流式传输可以实现以下功能：

* [**流式传输智能体进度**](#智能体进度) — 在每个智能体步骤后获取状态更新。
* [**流式传输 LLM token**](#llm-token) — 在生成语言模型 token 时进行流式传输。
* [**流式传输自定义更新**](#自定义更新) — 发出用户定义的信号（例如，"已获取 10/100 条记录"）。
* [**流式传输多种模式**](#流式传输多种模式) — 可以选择 `updates`（智能体进度）、`messages`（LLM token + 元数据）或 `custom`（任意用户数据）。

请参阅下面的[常见模式](#常见模式)部分，获取更多端到端示例。

## 支持的流式传输模式

将一个或多个以下流式传输模式作为列表传递给 [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) 或 [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream) 方法：

| 模式 | 描述 |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `updates` | 在每个智能体步骤后流式传输状态更新。如果在同一步骤中进行多次更新（例如，运行多个节点），这些更新将分别流式传输。 |
| `messages` | 从调用 LLM 的任何图节点流式传输 `(token, metadata)` 元组。 |
| `custom` | 使用流式写入器从图节点内部流式传输自定义数据。 |

## 智能体进度

要流式传输智能体进度，请使用 [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) 或 [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream) 方法，并将 `stream_mode="updates"` 设置为在每个智能体步骤后发出一个事件。

例如，如果您有一个调用一次工具的智能体，您应该看到以下更新：

* **LLM 节点**：带有工具调用请求的 [`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage)
* **工具节点**：带有执行结果的 [`ToolMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage)
* **LLM 节点**：最终 AI 响应

```python title="流式传输智能体进度" theme={null}
from langchain.agents import create_agent


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"{city} 永远是晴天！"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)
for chunk in agent.stream(  # [!code highlight]
    {"messages": [{"role": "user", "content": "旧金山的天气怎么样？"}]},
    stream_mode="updates",
):
    for step, data in chunk.items():
        print(f"步骤: {step}")
        print(f"内容: {data['messages'][-1].content_blocks}")
```

```shell title="输出" theme={null}
步骤: model
内容: [{'type': 'tool_call', 'name': 'get_weather', 'args': {'city': 'San Francisco'}, 'id': 'call_OW2NYNsNSKhRZpjW0wm2Aszd'}]

步骤: tools
内容: [{'type': 'text', 'text': "旧金山永远是晴天！"}]

步骤: model
内容: [{'type': 'text', 'text': '旧金山永远是晴天！'}]
```

## LLM token

要在 LLM 生成 token 时流式传输它们，请使用 `stream_mode="messages"`。下面您可以看到智能体流式传输工具调用和最终响应的输出。

```python title="流式传输 LLM token" theme={null}
from langchain.agents import create_agent


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"{city} 永远是晴天！"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)
for token, metadata in agent.stream(  # [!code highlight]
    {"messages": [{"role": "user", "content": "旧金山的天气怎么样？"}]},
    stream_mode="messages",
):
    print(f"节点: {metadata['langgraph_node']}")
    print(f"内容: {token.content_blocks}")
    print("\n")
```

```shell title="输出" expandable theme={null}
节点: model
内容: [{'type': 'tool_call_chunk', 'id': 'call_vbCyBcP8VuneUzyYlSBZZsVa', 'name': 'get_weather', 'args': '', 'index': 0}]


节点: model
内容: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': '{"', 'index': 0}]


节点: model
内容: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': 'city', 'index': 0}]


节点: model
内容: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': '":"', 'index': 0}]


节点: model
内容: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': 'San', 'index': 0}]


节点: model
内容: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': ' Francisco', 'index': 0}]


节点: model
内容: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': '"}', 'index': 0}]


节点: model
内容: []


节点: tools
内容: [{'type': 'text', 'text': "旧金山永远是晴天！"}]


节点: model
内容: []


节点: model
内容: [{'type': 'text', 'text': 'Here'}]


节点: model
内容: [{'type': 'text', 'text': ''s'}]


节点: model
内容: [{'type': 'text', 'text': ' what'}]


节点: model
内容: [{'type': 'text', 'text': ' I'}]


节点: model
内容: [{'type': 'text', 'text': ' got'}]


节点: model
内容: [{'type': 'text', 'text': ':'}]


节点: model
内容: [{'type': 'text', 'text': ' "'}]


节点: model
内容: [{'type': 'text', 'text': "It's"}]


节点: model
内容: [{'type': 'text', 'text': ' always'}]


节点: model
内容: [{'type': 'text', 'text': ' sunny'}]


节点: model
内容: [{'type': 'text', 'text': ' in'}]


节点: model
内容: [{'type': 'text', 'text': ' San'}]


节点: model
内容: [{'type': 'text', 'text': ' Francisco'}]


节点: model
内容: [{'type': 'text', 'text': '!"\n\n'}]
```

## 自定义更新

要流式传输工具执行过程中的更新，您可以使用 [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer)。

```python title="流式传输自定义更新" theme={null}
from langchain.agents import create_agent
from langgraph.config import get_stream_writer  # [!code highlight]


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    writer = get_stream_writer()  # [!code highlight]
    # 流式传输任意数据
    writer(f"正在查找城市 {city} 的数据")
    writer(f"已获取城市 {city} 的数据")
    return f"{city} 永远是晴天！"

agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[get_weather],
)

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "旧金山的天气怎么样？"}]},
    stream_mode="custom"  # [!code highlight]
):
    print(chunk)
```

```shell title="输出" theme={null}
正在查找城市 San Francisco 的数据
已获取城市 San Francisco 的数据
```

<Note>
  如果您在工具中添加 [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer)，您将无法在 LangGraph 执行上下文之外调用该工具。
</Note>

## 流式传输多种模式

您可以通过将流式传输模式作为列表传递来指定多种流式传输模式：`stream_mode=["updates", "custom"]`。

流式传输的输出将是 `(mode, chunk)` 元组，其中 `mode` 是流式传输模式的名称，`chunk` 是该模式流式传输的数据。

```python title="流式传输多种模式" theme={null}
from langchain.agents import create_agent
from langgraph.config import get_stream_writer


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    writer = get_stream_writer()
    writer(f"正在查找城市 {city} 的数据")
    writer(f"已获取城市 {city} 的数据")
    return f"{city} 永远是晴天！"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)

for stream_mode, chunk in agent.stream(  # [!code highlight]
    {"messages": [{"role": "user", "content": "旧金山的天气怎么样？"}]},
    stream_mode=["updates", "custom"]
):
    print(f"流式传输模式: {stream_mode}")
    print(f"内容: {chunk}")
    print("\n")
```

```shell title="输出" theme={null}
流式传输模式: updates
内容: {'model': {'messages': [AIMessage(content='', response_metadata={'token_usage': {'completion_tokens': 280, 'prompt_tokens': 132, 'total_tokens': 412, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 256, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-5-nano-2025-08-07', 'system_fingerprint': None, 'id': 'chatcmpl-C9tlgBzGEbedGYxZ0rTCz5F7OXpL7', 'service_tier': 'default', 'finish_reason': 'tool_calls', 'logprobs': None}, id='lc_run--480c07cb-e405-4411-aa7f-0520fddeed66-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'San Francisco'}, 'id': 'call_KTNQIftMrl9vgNwEfAJMVu7r', 'type': 'tool_call'}], usage_metadata={'input_tokens': 132, 'output_tokens': 280, 'total_tokens': 412, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 256})]}}


流式传输模式: custom
内容: 正在查找城市 San Francisco 的数据


流式传输模式: custom
内容: 已获取城市 San Francisco 的数据


流式传输模式: updates
内容: {'tools': {'messages': [ToolMessage(content="旧金山永远是晴天！", name='get_weather', tool_call_id='call_KTNQIftMrl9vgNwEfAJMVu7r')]}}


流式传输模式: updates
内容: {'model': {'messages': [AIMessage(content='旧金山天气: 旧金山永远是晴天！\n\n', response_metadata={'token_usage': {'completion_tokens': 764, 'prompt_tokens': 168, 'total_tokens': 932, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 704, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-5-nano-2025-08-07', 'system_fingerprint': None, 'id': 'chatcmpl-C9tljDFVki1e1haCyikBptAuXuHYG', 'service_tier': 'default', 'finish_reason': 'stop', 'logprobs': None}, id='lc_run--acbc740a-18fe-4a14-8619-da92a0d0ee90-0', usage_metadata={'input_tokens': 168, 'output_tokens': 764, 'total_tokens': 932, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 704})]}}
```

## 常见模式

以下是展示流式传输常见用例的示例。

### 流式传输工具调用

您可能希望同时流式传输：

1. 工具调用生成时的部分 JSON（[工具调用(https://docs.langchain.com/oss/python/langchain/models#tool-calling)）
2. 执行的完整解析后的工具调用

指定 [`stream_mode="messages"`](#llm-token) 将流式传输智能体中所有 LLM 调用生成的增量[消息块(https://docs.langchain.com/oss/python/langchain/messages#streaming-and-chunks)。要访问带有解析工具调用的完成消息：

1. 如果这些消息被跟踪在[状态(https://docs.langchain.com/oss/python/langchain/agents#memory)中（如 [`create_agent`(https://docs.langchain.com/oss/python/langchain/agents) 的模型节点中），请使用 `stream_mode=["messages", "updates"]` 通过[状态更新](#智能体进度)访问完成的消息（如下面演示）。
2. 如果这些消息没有被跟踪在状态中，请使用[自定义更新](#自定义更新)或在流式传输循环期间聚合块（[下一节](#访问完成的消息)）。

<Note>
  如果您的智能体包含多个 LLM，请参阅下面的[从子智能体流式传输](#从子智能体流式传输)部分。
</Note>

```python  theme={null}
from typing import Any

from langchain.agents import create_agent
from langchain.messages import AIMessage, AIMessageChunk, AnyMessage, ToolMessage


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"{city} 永远是晴天！"


agent = create_agent("openai:gpt-5.2", tools=[get_weather])


def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)
    # 注意：所有内容都可以通过 token.content_blocks 访问


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"工具调用: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"工具响应: {message.content_blocks}")


input_message = {"role": "user", "content": "波士顿的天气怎么样？"}
for stream_mode, data in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],  # [!code highlight]
):
    if stream_mode == "messages":
        token, metadata = data
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)  # [!code highlight]
    if stream_mode == "updates":
        for source, update in data.items():
            if source in ("model", "tools"):  # `source` 捕获节点名称
                _render_completed_message(update["messages"][-1])  # [!code highlight]
```

```shell title="输出" expandable theme={null}
[{'name': 'get_weather', 'args': '', 'id': 'call_D3Orjr89KgsLTZ9hTzYv7Hpf', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'city', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
工具调用: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_D3Orjr89KgsLTZ9hTzYv7Hpf', 'type': 'tool_call'}]
工具响应: [{'type': 'text', 'text': "波士顿永远是晴天！"}]
天气|在| **|波士顿|**| 是| **|晴朗|**|。
```

#### 访问完成的消息

<Note>
  如果完成的消息被跟踪在智能体的[状态(https://docs.langchain.com/oss/python/langchain/agents#memory)中，您可以使用 `stream_mode=["messages", "updates"]` 如上所述[](#流式传输工具调用)在流式传输期间访问完成的消息。
</Note>

在某些情况下，完成的消息不会反映在[状态更新](#智能体进度)中。如果您可以访问智能体内部，可以使用[自定义更新](#自定义更新)在流式传输期间访问这些消息。否则，您可以在流式传输循环中聚合消息块（见下文）。

考虑下面的例子，我们将[流式写入器](#自定义更新)整合到一个简化的[护栏中间件(https://docs.langchain.com/oss/python/langchain/guardrails#after-agent-guardrails)中。该中间件演示了工具调用以生成"安全/不安全"的结构化评估（您也可以为此使用[结构化输出(https://docs.langchain.com/oss/python/langchain/models#structured-output)）：

```python  theme={null}
from typing import Any, Literal

from langchain.agents.middleware import after_agent, AgentState
from langgraph.runtime import Runtime
from langchain.messages import AIMessage
from langchain.chat_models import init_chat_model
from langgraph.config import get_stream_writer  # [!code highlight]
from pydantic import BaseModel


class ResponseSafety(BaseModel):
    """将响应评估为安全或不安全。"""
    evaluation: Literal["safe", "unsafe"]


safety_model = init_chat_model("openai:gpt-5.2")

@after_agent(can_jump_to=["end"])
def safety_guardrail(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """基于模型的护栏：使用 LLM 评估响应安全性。"""
    stream_writer = get_stream_writer()  # [!code highlight]
    # 获取模型响应
    if not state["messages"]:
        return None

    last_message = state["messages"][-1]
    if not isinstance(last_message, AIMessage):
        return None

    # 使用另一个模型评估安全性
    model_with_tools = safety_model.bind_tools([ResponseSafety], tool_choice="any")
    result = model_with_tools.invoke(
        [
            {
                "role": "system",
                "content": "将此 AI 响应评估为一般安全或不安全。"
            },
            {
                "role": "user",
                "content": f"AI 响应: {last_message.text}"
            }
        ]
    )
    stream_writer(result)  # [!code highlight]

    tool_call = result.tool_calls[0]
    if tool_call["args"]["evaluation"] == "unsafe":
        last_message.content = "我无法提供该响应。请重新表述您的请求。"

    return None
```

然后我们可以将此中间件整合到我们的智能体中，并包含其自定义流式传输事件：

```python  theme={null}
from typing import Any

from langchain.agents import create_agent
from langchain.messages import AIMessageChunk, AIMessage, AnyMessage


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"{city} 永远是晴天！"


agent = create_agent(
    model="openai:gpt-5.2",
    tools=[get_weather],
    middleware=[safety_guardrail],  # [!code highlight]
)

def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"工具调用: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"工具响应: {message.content_blocks}")


input_message = {"role": "user", "content": "波士顿的天气怎么样？"}
for stream_mode, data in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates", "custom"],  # [!code highlight]
):
    if stream_mode == "messages":
        token, metadata = data
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    if stream_mode == "updates":
        for source, update in data.items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
    if stream_mode == "custom":  # [!code highlight]
        # 在流中访问完成的消息
        print(f"工具调用: {data.tool_calls}")  # [!code highlight]
```

```shell title="输出" expandable theme={null}
[{'name': 'get_weather', 'args': '', 'id': 'call_je6LWgxYzuZ84mmoDalTYMJC', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'city', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
工具调用: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_je6LWgxYzuZ84mmoDalTYMJC', 'type': 'tool_call'}]
工具响应: [{'type': 'text', 'text': "波士顿永远是晴天！"}]
天气|在| **|波士顿|**| 是| **|晴朗|**|。|[{'name': 'ResponseSafety', 'args': '', 'id': 'call_O8VJIbOG4Q9nQF0T8ltVi58O', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'evaluation', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'safe', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
工具调用: [{'name': 'ResponseSafety', 'args': {'evaluation': 'safe'}, 'id': 'call_O8VJIbOG4Q9nQF0T8ltVi58O', 'type': 'tool_call'}]
```

或者，如果您无法向流添加自定义事件，您可以在流式传输循环内聚合消息块：

```python  theme={null}
input_message = {"role": "user", "content": "波士顿的天气怎么样？"}
full_message = None  # [!code highlight]
for stream_mode, data in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],
):
    if stream_mode == "messages":
        token, metadata = data
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
            full_message = token if full_message is None else full_message + token  # [!code highlight]
            if token.chunk_position == "last":  # [!code highlight]
                if full_message.tool_calls:  # [!code highlight]
                    print(f"工具调用: {full_message.tool_calls}")  # [!code highlight]
                full_message = None  # [!code highlight]
    if stream_mode == "updates":
        for source, update in data.items():
            if source == "tools":
                _render_completed_message(update["messages"][-1])
```

### 带人工介入的流式传输

要处理人工介入[中断(https://docs.langchain.com/oss/python/langchain/human-in-the-loop)，我们基于上面的[示例](#流式传输工具调用)构建：

1. 我们使用[人工介入中间件和检查点(https://docs.langchain.com/oss/python/langchain/human-in-the-loop#configuring-interrupts)配置智能体
2. 我们收集 `"updates`" 流式传输模式中产生的中断
3. 我们用[命令(https://docs.langchain.com/oss/python/langchain/human-in-the-loop#responding-to-interrupts)响应这些中断

```python  theme={null}
from typing import Any

from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.messages import AIMessage, AIMessageChunk, AnyMessage, ToolMessage
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command, Interrupt


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"{city} 永远是晴天！"


checkpointer = InMemorySaver()

agent = create_agent(
    "openai:gpt-5.2",
    tools=[get_weather],
    middleware=[  # [!code highlight]
        HumanInTheLoopMiddleware(interrupt_on={"get_weather": True}),  # [!code highlight]
    ],  # [!code highlight]
    checkpointer=checkpointer,  # [!code highlight]
)


def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"工具调用: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"工具响应: {message.content_blocks}")


def _render_interrupt(interrupt: Interrupt) -> None:  # [!code highlight]
    interrupts = interrupt.value  # [!code highlight]
    for request in interrupts["action_requests"]:  # [!code highlight]
        print(request["description"])  # [!code highlight]


input_message = {
    "role": "user",
    "content": (
        "你能查看波士顿和旧金山的天气吗？"
    ),
}
config = {"configurable": {"thread_id": "some_id"}}  # [!code highlight]
interrupts = []  # [!code highlight]
for stream_mode, data in agent.stream(
    {"messages": [input_message]},
    config=config,  # [!code highlight]
    stream_mode=["messages", "updates"],
):
    if stream_mode == "messages":
        token, metadata = data
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    if stream_mode == "updates":
        for source, update in data.items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
            if source == "__interrupt__":  # [!code highlight]
                interrupts.extend(update)  # [!code highlight]
                _render_interrupt(update[0])  # [!code highlight]
```

```shell title="输出" expandable theme={null}
[{'name': 'get_weather', 'args': '', 'id': 'call_GOwNaQHeqMixay2qy80padfE', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"ci', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'ty": ', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"Bosto', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'n"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': 'get_weather', 'args': '', 'id': 'call_Ndb4jvWm2uMA0JDQXu37wDH6', 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"ci', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'ty": ', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"San F', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'ranc', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'isco"', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '}', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
工具调用: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_GOwNaQHeqMixay2qy80padfE', 'type': 'tool_call'}, {'name': 'get_weather', 'args': {'city': 'San Francisco'}, 'id': 'call_Ndb4jvWm2uMA0JDQXu37wDH6', 'type': 'tool_call'}]
需要批准工具执行

工具: get_weather
参数: {'city': 'Boston'}
需要批准工具执行

工具: get_weather
参数: {'city': 'San Francisco'}
```

接下来，我们为每个中断收集一个[决策(https://docs.langchain.com/oss/python/langchain/human-in-the-loop#interrupt-decision-types)。重要的是，决策的顺序必须与我们收集的动作顺序匹配。

为了说明，我们将编辑一个工具调用并接受另一个：

```python  theme={null}
def _get_interrupt_decisions(interrupt: Interrupt) -> list[dict]:
    return [
        {
            "type": "edit",
            "edited_action": {
                "name": "get_weather",
                "args": {"city": "Boston, U.K."},
            },
        }
        if "boston" in request["description"].lower()
        else {"type": "approve"}
        for request in interrupt.value["action_requests"]
    ]

decisions = {}
for interrupt in interrupts:
    decisions[interrupt.id] = {
        "decisions": _get_interrupt_decisions(interrupt)
    }

decisions
```

```shell title="输出" theme={null}
{
    'a96c40474e429d661b5b32a8d86f0f3e': {
        'decisions': [
            {
                'type': 'edit',
                 'edited_action': {
                     'name': 'get_weather',
                     'args': {'city': 'Boston, U.K.'}
                 }
            },
            {'type': 'approve'},
        ]
    }
}
```

然后我们可以通过将[命令(https://docs.langchain.com/oss/python/langchain/human-in-the-loop#responding-to-interrupts)传递到同一个流式传输循环来恢复：

```python  theme={null}
interrupts = []
for stream_mode, data in agent.stream(
    Command(resume=decisions),  # [!code highlight]
    config=config,
    stream_mode=["messages", "updates"],
):
    # 流式传输循环保持不变
    if stream_mode == "messages":
        token, metadata = data
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    if stream_mode == "updates":
        for source, update in data.items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
            if source == "__interrupt__":
                interrupts.extend(update)
                _render_interrupt(update[0])
```

```shell title="输出" theme={null}
工具响应: [{'type': 'text', 'text': "波士顿，英国永远是晴天！"}]
工具响应: [{'type': 'text', 'text": "旧金山永远是晴天！"}]
-| **|波士顿|**|:| 它|在|波士顿|永远|是|晴朗|的|,| U|.K|。
|-| **|旧金山|**|:| 它|在|旧金山|永远|是|晴朗|的|!|
```

### 从子智能体流式传输

当智能体中任何时候有多个 LLM 时，通常需要消除生成消息时的来源歧义。

为此，请在创建每个智能体时向其传递一个 [`name`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent\(name\))。然后当在 `"messages"` 模式下流式传输时，此名称可通过 `lc_agent_name` 键在元数据中使用。

下面，我们更新[流式传输工具调用](#流式传输工具调用)示例：

1. 我们将工具替换为在内部调用智能体的 `call_weather_agent` 工具
2. 我们为每个智能体添加一个 `name`
3. 我们在创建流式传输时指定 [`subgraphs=True`(https://docs.langchain.com/oss/python/langgraph/use-subgraphs#stream-subgraph-outputs)
4. 我们的流式传输处理与之前相同，但我们添加逻辑以使用 `create_agent` 的 `name` 参数跟踪当前活动的智能体

<Tip>
  当您在智能体上设置 `name` 时，该名称也会附加到该智能体生成的任何 `AIMessage` 上。
</Tip>

首先我们构建智能体：

```python  theme={null}
from typing import Any

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.messages import AIMessage, AnyMessage


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"{city} 永远是晴天！"


weather_model = init_chat_model("openai:gpt-5.2")
weather_agent = create_agent(
    model=weather_model,
    tools=[get_weather],
    name="weather_agent",  # [!code highlight]
)


def call_weather_agent(query: str) -> str:
    """查询天气智能体。"""
    result = weather_agent.invoke({
        "messages": [{"role": "user", "content": query}]
    })
    return result["messages"][-1].text


supervisor_model = init_chat_model("openai:gpt-5.2")
agent = create_agent(
    model=supervisor_model,
    tools=[call_weather_agent],
    name="supervisor",  # [!code highlight]
)
```

接下来，我们将逻辑添加到流式传输循环中，以报告哪个智能体正在发出 token：

```python  theme={null}
def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"工具调用: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"工具响应: {message.content_blocks}")


input_message = {"role": "user", "content": "波士顿的天气怎么样？"}
current_agent = None  # [!code highlight]
for _, stream_mode, data in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],
    subgraphs=True,  # [!code highlight]
):
    if stream_mode == "messages":
        token, metadata = data
        if agent_name := metadata.get("lc_agent_name"):  # [!code highlight]
            if agent_name != current_agent:  # [!code highlight]
                print(f"🤖 {agent_name}: ")  # [!code highlight]
                current_agent = agent_name  # [!code highlight]
        if isinstance(token, AIMessage):
            _render_message_chunk(token)
    if stream_mode == "updates":
        for source, update in data.items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
```

```shell title="输出" expandable theme={null}
🤖 supervisor:
[{'name': 'call_weather_agent', 'args': '', 'id': 'call_asorzUf0mB6sb7MiKfgojp7I', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'query', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' weather', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' right', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' now', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' and', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': " today's", 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' forecast', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
工具调用: [{'name': 'call_weather_agent', 'args': {'query': "波士顿现在的天气和今天的预报"}, 'id': 'call_asorzUf0mB6sb7MiKfgojp7I', 'type': 'tool_call'}]
🤖 weather_agent:
[{'name': 'get_weather', 'args': '', 'id': 'call_LZ89lT8fW6w8vqck5pZeaDIx', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'city', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
工具调用: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_LZ89lT8fW6w8vqck5pZeaDIx', 'type': 'tool_call'}]
工具响应: [{'type': 'text', 'text': "波士顿永远是晴天！"}]
波士顿|天气|现在|:| **|晴朗|**|。

|今天|波士顿|的|预报|:| **|全天|晴朗|**|。|工具响应: [{'type': 'text', 'text': '波士顿现在的天气: **晴朗**。

今天波士顿的预报: **全天晴朗**。'}]
🤖 supervisor:
波士顿|天气|现在|:| **|晴朗|**|。

|今天|波士顿|的|预报|:| **|全天|晴朗|**|。
```

## 禁用流式传输

在某些应用程序中，您可能需要禁用给定模型的单个 token 流式传输。这在以下情况很有用：

* 使用[多智能体(https://docs.langchain.com/oss/python/langchain/multi-agent)系统来控制哪些智能体流式传输其输出
* 混合支持流式传输的模型和不支持的模型
* 部署到 [LangSmith(https://docs.smith.langchain.com/home) 并希望防止某些模型输出流式传输到客户端

在初始化模型时设置 `streaming=False`。

```python  theme={null}
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="gpt-4o",
    streaming=False  # [!code highlight]
)
```

<Tip>
  部署到 LangSmith 时，在您不希望流式传输到客户端的任何模型上设置 `streaming=False`。这是在部署前的图代码中配置的。
</Tip>

<Note>
  并非所有聊天模型集成都支持 `streaming` 参数。如果您的模型不支持，请改用 `disable_streaming=True`。此参数可通过基类在所有聊天模型上使用。
</Note>

更多详情请参阅 [LangGraph 流式传输指南(https://docs.langchain.com/oss/python/langgraph/streaming#disable-streaming-for-specific-chat-models)。

## 相关内容

* [前端流式传输](frontend.md) — 使用 `useStream` 构建具有实时智能体交互功能的 React UI
* [使用聊天模型进行流式传输](../models.md) — 直接从聊天模型流式传输 token，无需使用智能体或图
* [带人工介入的流式传输](../../advanced-usage/human-in-the-loop.md) — 在处理人工审查的中断时流式传输智能体进度
* [LangGraph 流式传输](https://python.langchain.com/langgraph/streaming) — 高级流式传输选项，包括 `values`、`debug` 模式和子图流式传输

***



