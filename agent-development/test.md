# 测试

智能体应用让 LLM 自主决定下一步行动来解决问题。这种灵活性非常强大，但由于模型的黑盒特性，很难预测对智能体某一部分的调整会如何影响整体。

本指南将介绍测试智能体应用的方法。我们将介绍：

1. **单元测试** — 使用 `GenericFakeChatModel` 和 `InMemorySaver` 测试单个组件和简单流程
2. **集成测试** — 使用 `AgentEvals` 测试完整轨迹和工具调用
3. **LangSmith 集成** — 使用 LangSmith 追踪和记录测试结果
4. **HTTP 录制回放** — 使用 `vcrpy` 录制和回放网络请求

## 单元测试

测试智能体的最简单方法是使用 LangChain 的测试工具。

### 使用 GenericFakeChatModel

[`GenericFakeChatModel`](/docs/core_components/chat_models/#genericfakechatmodel) 是一个轻量级的模拟聊天模型，适用于测试智能体组件而无需调用真实模型。

**功能特点：**

- 完全确定性 — 相同的输入产生相同的输出
- 无外部依赖 — 不需要 API 密钥或网络连接
- 快速执行 — 适合 CI/CD 环境

```python
from langchain.chat_models import GenericFakeChatModel
from langchain.messages import HumanMessage, AIMessage

model = GenericFakeChatModel(
    messages=[
        AIMessage(content="I'm a fake model.")
    ]
)

response = model.invoke([HumanMessage(content="Hi there!")])
print(response.content())  # "I'm a fake model."
```

**配置响应：**

您可以配置模型在收到特定输入时返回特定响应：

```python
from langchain.chat_models import GenericFakeChatModel
from langchain.schema.messages import HumanMessage

model = GenericFakeChatModel(
    # 设置一个映射：当收到包含 "weather" 的消息时，返回天气查询结果
    responses={
        "What's the weather?": [
            AIMessage(content="I see you're asking about the weather.")
        ]
    }
)

response = model.invoke([HumanMessage(content="What's the weather?")])
print(response.content())  # "I see you're asking about the weather."

`GenericFakeChatModel` 还可以配置为返回结构化输出或工具调用。详情请参阅 [API 参考](https://python.langchain.com/api_reference/core/chat_models/langchain_core.chat_models.fake.GenericFakeChatModel.html)。

### 测试工具调用

使用 `GenericFakeChatModel` 测试智能体时，您可以配置模型返回特定工具调用：

```python
from langchain.chat_models import GenericFakeChatModel
from langchain.tools import tool
from langchain.agents import create_agent
from langchain.messages import HumanMessage

# 定义一个简单的工具
@tool
def get_weather(city: str):
    """获取城市的天气信息。"""
    return f"City: {city}, Weather: sunny, 75°F"

# 配置模拟模型返回特定工具调用
model = GenericFakeChatModel(
    messages=[
        AIMessage(
            content="",
            tool_calls=[{
                "id": "1",
                "name": "get_weather",
                "args": {"city": "San Francisco"}
            }]
        ),
        AIMessage(content="The weather in San Francisco is sunny and 75°F.")
    ]
)

# 创建智能体
agent = create_agent(model, tools=[get_weather])

# 测试
result = agent.invoke({
    "messages": [HumanMessage(content="What's the weather in San Francisco?")]
})

for msg in result["messages"]:
    print(f"{type(msg).__name__}: {msg.content if hasattr(msg, 'content') else msg}")
```

**输出：**

```
HumanMessage: What's the weather in San Francisco?
AIMessage:
    ToolCall:
        id='1'
        name='get_weather'
        args={'city': 'San Francisco'}
ToolMessage: City: San Francisco, Weather: sunny, 75°F
AIMessage: The weather in San Francisco is sunny and 75°F.
```

### 测试人类介入场景

使用 `InMemorySaver` 测试需要人类介入的流程：

```python
from langchain.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, END
from langgraph.types import Command

# 创建一个带有人类介入节点的图
def human_node(state, command: Command):
    return {"messages": [*state["messages"], command.partial["human_message"]]}

graph = (
    StateGraph(dict)
    .add_node(human_node)
    .add_edge("__start__", "human_node")
    .add_edge("human_node", END)
    .compile(checkpointer=InMemorySaver())
)

# 模拟人类输入
inputs = {"messages": []}
human_input = "I want to order a pizza."

config = {"configurable": {"thread_id": "test-human-in-loop"}}
graph.invoke(Command(resume=human_input), config=config)

# 验证人类介入是否正常工作
state = graph.get_state(config)
print(state.values)
```

## 集成测试

集成测试验证智能体的完整执行轨迹。`agentevals` 库提供了专门的评估器来测试智能体行为。

### 轨迹匹配评估器

使用 `create_trajectory_match_evaluator` 根据预定义的参考轨迹验证智能体行为。

**可用模式：**

- `strict` — 精确匹配整个轨迹
- `unordered` — 工具调用顺序可以不同
- `superset` — 智能体至少调用了参考轨迹中的工具（允许额外调用）
- `subset` — 智能体只调用了参考轨迹中的工具（不允许额外调用）

**精确匹配示例：**

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langchain.messages import HumanMessage, AIMessage, ToolMessage
from agentevals.trajectory.match import create_trajectory_match_evaluator


@tool
def get_weather(city: str):
    """获取城市的天气信息。"""
    return f"It's 75 degrees and sunny in {city}."

agent = create_agent("gpt-4o", tools=[get_weather])

evaluator = create_trajectory_match_evaluator(
    trajectory_match_mode="strict",
)

def test_weather_tool_called_strict():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in San Francisco?")]
    })

    reference_trajectory = [
        HumanMessage(content="What's the weather in San Francisco?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_weather", "args": {"city": "San Francisco"}}
        ]),
        ToolMessage(content="It's 75 degrees and sunny in San Francisco.", tool_call_id="call_1"),
        AIMessage(content="The weather in San Francisco is 75 degrees and sunny."),
    ]

    evaluation = evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory
    )
    # {
    #     'key': 'trajectory_strict_match',
    #     'score': True,
    #     'comment': None,
    # }
    assert evaluation["score"] is True
```

**无序匹配示例：**

`unordered` 模式允许相同的工具调用以任意顺序执行，当您只想验证获取了特定信息而不关心顺序时非常有用。例如，智能体可能需要检查某个城市的天气和活动，但顺序并不重要。

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langchain.messages import HumanMessage, AIMessage, ToolMessage
from agentevals.trajectory.match import create_trajectory_match_evaluator


@tool
def get_weather(city: str):
    """获取城市的天气信息。"""
    return f"It's 75 degrees and sunny in {city}."

@tool
def get_events(city: str):
    """获取城市正在发生的活动。"""
    return f"Concert at the park in {city} tonight."

agent = create_agent("gpt-4o", tools=[get_weather, get_events])

evaluator = create_trajectory_match_evaluator(
    trajectory_match_mode="unordered",
)

def test_multiple_tools_any_order():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's happening in SF today?")]
    })

    # 参考轨迹以不同顺序显示工具调用
    reference_trajectory = [
        HumanMessage(content="What's happening in SF today?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_events", "args": {"city": "SF"}},
            {"id": "call_2", "name": "get_weather", "args": {"city": "SF"}},
        ]),
        ToolMessage(content="Concert at the park in SF tonight.", tool_call_id="call_1"),
        ToolMessage(content="It's 75 degrees and sunny in SF.", tool_call_id="call_2"),
        AIMessage(content="Today in SF: 75 degrees and sunny with a concert at the park tonight."),
    ]

    evaluation = evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory,
    )
    # {
    #     'key': 'trajectory_unordered_match',
    #     'score': True,
    # }
    assert evaluation["score"] is True
```

**子集和超集匹配示例：**

`superset` 和 `subset` 模式匹配部分轨迹。`superset` 模式验证智能体至少调用了参考轨迹中的工具，允许额外的工具调用。`subset` 模式确保智能体没有调用参考轨迹之外的任何工具。

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langchain.messages import HumanMessage, AIMessage, ToolMessage
from agentevals.trajectory.match import create_trajectory_match_evaluator


@tool
def get_weather(city: str):
    """获取城市的天气信息。"""
    return f"It's 75 degrees and sunny in {city}."

@tool
def get_detailed_forecast(city: str):
    """获取城市的详细天气预报。"""
    return f"Detailed forecast for {city}: sunny all week."

agent = create_agent("gpt-4o", tools=[get_weather, get_detailed_forecast])

evaluator = create_trajectory_match_evaluator(
    trajectory_match_mode="superset",
)

def test_agent_calls_required_tools_plus_extra():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in Boston?")]
    })

    # 参考轨迹只要求 get_weather，但智能体可能调用额外工具
    reference_trajectory = [
        HumanMessage(content="What's the weather in Boston?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_weather", "args": {"city": "Boston"}},
        ]),
        ToolMessage(content="It's 75 degrees and sunny in Boston.", tool_call_id="call_1"),
        AIMessage(content="The weather in Boston is 75 degrees and sunny."),
    ]

    evaluation = evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory,
    )
    # {
    #     'key': 'trajectory_superset_match',
    #     'score': True,
    #     'comment': None,
    # }
    assert evaluation["score"] is True
```

您还可以设置 `tool_args_match_mode` 属性和/或 `tool_args_match_overrides` 来自定义评估器如何考虑实际轨迹与参考轨迹中工具调用的相等性。默认情况下，只有参数相同的同一工具调用才被视为相等。详情请访问[仓库](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#tool-args-match-modes)。

### LLM-as-Judge 评估器

您还可以使用 LLM 通过 `create_trajectory_llm_as_judge` 函数来评估智能体的执行路径。与轨迹匹配评估器不同，它不需要参考轨迹，但如果提供了也可以使用。

**无参考轨迹示例：**

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langchain.messages import HumanMessage, AIMessage, ToolMessage
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT


@tool
def get_weather(city: str):
    """获取城市的天气信息。"""
    return f"It's 75 degrees and sunny in {city}."

agent = create_agent("gpt-4o", tools=[get_weather])

evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

def test_trajectory_quality():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in Seattle?")]
    })

    evaluation = evaluator(
        outputs=result["messages"],
    )
    # {
    #     'key': 'trajectory_accuracy',
    #     'score': True,
    #     'comment': 'The provided agent trajectory is reasonable...'
    # }
    assert evaluation["score"] is True
```

**有参考轨迹示例：**

如果您有参考轨迹，可以向提示添加额外变量并传入参考轨迹。下面，我们使用预构建的 `TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE` 提示并配置 `reference_outputs` 变量：

```python
evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE,
)
evaluation = evaluator(
    outputs=result["messages"],
    reference_outputs=reference_trajectory,
)
```

有关配置 LLM 如何评估轨迹的更多详细信息，请访问[仓库](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#trajectory-llm-as-judge)。

### 异步支持

所有 `agentevals` 评估器都支持 Python asyncio。对于使用工厂函数的评估器，可以通过在函数名中的 `create_` 后添加 `async` 来获得异步版本。

**异步评估器和评估器示例：**

```python
from agentevals.trajectory.llm import create_async_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT
from agentevals.trajectory.match import create_async_trajectory_match_evaluator

async_judge = create_async_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

async_evaluator = create_async_trajectory_match_evaluator(
    trajectory_match_mode="strict",
)

async def test_async_evaluation():
    result = await agent.ainvoke({
        "messages": [HumanMessage(content="What's the weather?")]
    })

    evaluation = await async_judge(outputs=result["messages"])
    assert evaluation["score"] is True
```

## LangSmith 集成

要随时间追踪实验，您可以将评估器结果记录到 [LangSmith](https://smith.langchain.com/)，这是一个用于构建生产级 LLM 应用的平台，包括追踪、评估和实验工具。

首先，通过设置所需的环境变量来设置 LangSmith：

```bash
export LANGSMITH_API_KEY="your_langsmith_api_key"
export LANGSMITH_TRACING="true"
```

LangSmith 提供了两种运行评估的主要方法：pytest 集成和 `evaluate` 函数。

**使用 pytest 集成：**

```python
import pytest
from langsmith import testing as t
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

trajectory_evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

@pytest.mark.langsmith
def test_trajectory_accuracy():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in SF?")]
    })

    reference_trajectory = [
        HumanMessage(content="What's the weather in SF?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_weather", "args": {"city": "SF"}},
        ]),
        ToolMessage(content="It's 75 degrees and sunny in SF.", tool_call_id="call_1"),
        AIMessage(content="The weather in SF is 75 degrees and sunny."),
    ]

    # 将输入、输出和参考输出记录到 LangSmith
    t.log_inputs({})
    t.log_outputs({"messages": result["messages"]})
    t.log_reference_outputs({"messages": reference_trajectory})

    trajectory_evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory
    )
```

使用 pytest 运行评估：

```bash
pytest test_trajectory.py --langsmith-output
```

结果将自动记录到 LangSmith。

**使用 evaluate 函数：**

或者，您可以在 LangSmith 中创建一个数据集并使用 `evaluate` 函数：

```python
from langsmith import Client
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

client = Client()

trajectory_evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

def run_agent(inputs):
    """返回轨迹消息的智能体函数。"""
    return agent.invoke(inputs)["messages"]

experiment_results = client.evaluate(
    run_agent,
    data="your_dataset_name",
    evaluators=[trajectory_evaluator]
)
```

结果将自动记录到 LangSmith。

要了解有关评估智能体的更多信息，请参阅 [LangSmith 文档](https://docs.langchain.com/langsmith/pytest)。

## 录制和回放 HTTP 调用

调用真实 LLM API 的集成测试可能很慢且昂贵，特别是在 CI/CD 管道中频繁运行时。我们建议使用库来录制 HTTP 请求和响应，然后在后续运行中回放而不进行实际网络调用。

您可以使用 [`vcrpy`](https://pypi.org/project/vcrpy/1.5.2/) 来实现这一点。如果您使用 `pytest`，[`pytest-recording` 插件](https://pypi.org/project/pytest-recording/) 提供了最少配置即可启用此功能的简单方法。请求/响应被记录在 cassette 中，然后在后续运行中用于模拟真实的网络调用。

设置您的 `conftest.py` 文件以过滤掉 cassette 中的敏感信息：

```python conftest.py
import pytest

@pytest.fixture(scope="session")
def vcr_config():
    return {
        "filter_headers": [
            ("authorization", "XXXX"),
            ("x-api-key", "XXXX"),
            # ... 其他您想要屏蔽的标头
        ],
        "filter_query_parameters": [
            ("api_key", "XXXX"),
            ("key", "XXXX"),
        ],
    }
```

然后配置您的项目以识别 `vcr` 标记：

```ini pytest.ini
[pytest]
markers =
    vcr: record/replay HTTP via VCR
addopts = --record-mode=once
```

```toml pyproject.toml
[tool.pytest.ini_options]
markers = [
    "vcr: record/replay HTTP via VCR"
]
addopts = "--record-mode=once"
```

`--record-mode=once` 选项在第一次运行时录制 HTTP 交互，然后在后续运行中回放它们。

现在，只需用 `vcr` 标记装饰您的测试：

```python
@pytest.mark.vcr()
def test_agent_trajectory():
    # ...
```

第一次运行此测试时，您的智能体会进行真正的网络调用，pytest 会在 `tests/cassettes` 目录中生成 cassette 文件 `test_agent_trajectory.yaml`。后续运行将使用该 cassette 来模拟真正的网络调用，前提是智能体的请求与上次运行相同。如果请求发生变化，测试将失败，您需要删除 cassette 并重新运行测试以记录新的交互。

当您修改提示、添加新工具或更改预期轨迹时，保存的 cassette 将变得过时，现有测试**将失败**。您应该删除相应的 cassette 文件并重新运行测试以记录新的交互。



