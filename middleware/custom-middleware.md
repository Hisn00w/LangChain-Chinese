# 自定义中间件

通过实现在智能体执行流程的特定点运行的钩子来构建自定义中间件。

## 钩子

中间件提供了两种风格的钩子来拦截智能体执行：

- **节点风格钩子** — 在特定执行点按顺序运行
- **包装风格钩子** — 在每次模型或工具调用周围运行

### 节点风格钩子

在特定执行点按顺序运行。用于日志记录、验证和状态更新。

**可用的钩子：**

- `before_agent` — 在智能体启动之前（每次调用一次）
- `before_model` — 在每次模型调用之前
- `after_model` — 在每次模型响应之后
- `after_agent` — 在智能体完成之后（每次调用一次）

**示例：**

**装饰器方式：**

```python
from langchain.agents.middleware import before_model, after_model, AgentState
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any


@before_model(can_jump_to=["end"])
def check_message_limit(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    if len(state["messages"]) >= 50:
        return {
            "messages": [AIMessage("Conversation limit reached.")],
            "jumpTo": "end"
        }
    return None


@after_model
def log_response(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"Model returned: {state['messages'][-1].content}")
    return None
```

**类方式：**

```python
from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any

class MessageLimitMiddleware(AgentMiddleware):
    def __init__(self, max_messages: int = 50):
        super().__init__()
        self.max_messages = max_messages

    @hook_config(can_jump_to=["end"])
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        if len(state["messages"]) == self.max_messages:
            return {
                "messages": [AIMessage("Conversation limit reached.")],
                "jumpTo": "end"
            }
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"Model returned: {state['messages'][-1].content}")
        return None
```

### 包装风格钩子

拦截执行并控制处理程序的调用时机。用于重试、缓存和转换。

您可以决定处理程序被调用零次（短路）、一次（正常流程）或多次（重试逻辑）。

**可用的钩子：**

- `wrap_model_call` — 在每次模型调用周围
- `wrap_tool_call` — 在每次工具调用周围

**示例：**

**装饰器方式：**

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable


@wrap_model_call
def retry_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    for attempt in range(3):
        try:
            return handler(request)
        except Exception as e:
            if attempt == 2:
                raise
            print(f"Retry {attempt + 1}/3 after error: {e}")
```

**类方式：**

```python
from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
from typing import Callable

class RetryMiddleware(AgentMiddleware):
    def __init__(self, max_retries: int = 3):
        super().__init__()
        self.max_retries = max_retries

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        for attempt in range(self.max_retries):
            try:
                return handler(request)
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                print(f"Retry {attempt + 1}/{self.max_retries} after error: {e}")
```

## 创建中间件

您可以通过两种方式创建中间件：

- **基于装饰器的中间件** — 适用于单钩子中间件，简单快捷
- **基于类的中间件** — 适用于具有多个钩子或配置的复杂中间件，更强大

### 基于装饰器的中间件

适用于单钩子中间件，简单快捷。使用装饰器包装单个函数。

**可用的装饰器：**

**节点风格：**

- `@before_agent` — 在智能体启动之前运行（每次调用一次）
- `@before_model` — 在每次模型调用之前运行
- `@after_model` — 在每次模型响应之后运行
- `@after_agent` — 在智能体完成之后运行（每次调用一次）

**包装风格：**

- `@wrap_model_call` — 用自定义逻辑包装每次模型调用
- `@wrap_tool_call` — 用自定义逻辑包装每次工具调用

**便捷工具：**

- `@dynamic_prompt` — 生成动态系统提示

**示例：**

```python
from langchain.agents.middleware import (
    before_model,
    wrap_model_call,
    AgentState,
    ModelRequest,
    ModelResponse,
)
from langchain.agents import create_agent
from langgraph.runtime import Runtime
from typing import Any, Callable


@before_model
def log_before_model(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"About to call model with {len(state['messages'])} messages")
    return None


@wrap_model_call
def retry_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    for attempt in range(3):
        try:
            return handler(request)
        except Exception as e:
            if attempt == 2:
                raise
            print(f"Retry {attempt + 1}/3 after error: {e}")

agent = create_agent(
    model="gpt-4o",
    middleware=[log_before_model, retry_model],
    tools=[...],
)
```

**何时使用装饰器：**

- 需要单个钩子
- 无需复杂配置
- 快速原型设计

### 基于类的中间件

适用于具有多个钩子或配置的复杂中间件，更强大。当您需要为同一个钩子定义同步和异步实现，或想在单个中间件中组合多个钩子时，请使用类。

**示例：**

```python
from langchain.agents.middleware import (
    AgentMiddleware,
    AgentState,
    ModelRequest,
    ModelResponse,
)
from langgraph.runtime import Runtime
from typing import Any, Callable

class LoggingMiddleware(AgentMiddleware):
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"About to call model with {len(state['messages'])} messages")
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"Model returned: {state['messages'][-1].content}")
        return None

agent = create_agent(
    model="gpt-4o",
    middleware=[LoggingMiddleware()],
    tools=[...],
)
```

**何时使用类：**

- 为同一个钩子定义同步和异步实现
- 单个中间件中需要多个钩子
- 需要复杂配置（例如，可配置的阈值、自定义模型）
- 跨项目重用，并在初始化时配置

## 自定义状态模式

中间件可以使用自定义属性扩展智能体的状态。这使中间件能够：

- **跨执行跟踪状态**：维护计数器、标志或其他在整个智能体执行生命周期中持久化的值
- **在钩子之间共享数据**：将信息从 `before_model` 传递到 `after_model`，或在不同中间件实例之间传递
- **实现跨领域关注**：添加速率限制、使用跟踪、用户上下文或审计日志等功能，而无需修改核心智能体逻辑
- **做出条件决策**：使用累积状态来决定是否继续执行，跳转到不同节点，或动态修改行为

**装饰器方式：**

```python
from langchain.agents import create_agent
from langchain.messages import HumanMessage
from langchain.agents.middleware import AgentState, before_model, after_model
from typing_extensions import NotRequired
from typing import Any
from langgraph.runtime import Runtime


class CustomState(AgentState):
    model_call_count: NotRequired[int]
    user_id: NotRequired[str]


@before_model(state_schema=CustomState, can_jump_to=["end"])
def check_call_limit(state: CustomState, runtime: Runtime) -> dict[str, Any] | None:
    count = state.get("model_call_count", 0)
    if count > 10:
        return {"jumpTo": "end"}
    return None


@after_model(state_schema=CustomState)
def increment_counter(state: CustomState, runtime: Runtime) -> dict[str, Any] | None:
    return {"model_call_count": state.get("model_call_count", 0) + 1}


agent = create_agent(
    model="gpt-4o",
    middleware=[check_call_limit, increment_counter],
    tools=[],
)

# 使用自定义状态调用
result = agent.invoke({
    "messages": [HumanMessage("Hello")],
    "model_call_count": 0,
    "user_id": "user-123",
})
```

**类方式：**

```python
from langchain.agents import create_agent
from langchain.messages import HumanMessage
from langchain.agents.middleware import AgentState, AgentMiddleware
from typing_extensions import NotRequired
from typing import Any


class CustomState(AgentState):
    model_call_count: NotRequired[int]
    user_id: NotRequired[str]


class CallCounterMiddleware(AgentMiddleware[CustomState]):
    state_schema = CustomState

    def before_model(self, state: CustomState, runtime) -> dict[str, Any] | None:
        count = state.get("model_call_count", 0)
        if count > 10:
            return {"jumpTo": "end"}
        return None

    def after_model(self, state: CustomState, runtime) -> dict[str, Any] | None:
        return {"model_call_count": state.get("model_call_count", 0) + 1}


agent = create_agent(
    model="gpt-4o",
    middleware=[CallCounterMiddleware()],
    tools=[],
)

# 使用自定义状态调用
result = agent.invoke({
    "messages": [HumanMessage("Hello")],
    "model_call_count": 0,
    "user_id": "user-123",
})
```

## 执行顺序

使用多个中间件时，请了解它们的执行方式：

```python
agent = create_agent(
    model="gpt-4o",
    middleware=[middleware1, middleware2, middleware3],
    tools=[...],
)
```

**执行流程：**

**Before 钩子按顺序运行：**

1. `middleware1.before_agent()`
2. `middleware2.before_agent()`
3. `middleware3.before_agent()`

**智能体循环开始**

4. `middleware1.before_model()`
5. `middleware2.before_model()`
6. `middleware3.before_model()`

**包装钩子嵌套（像函数调用一样）：**

7. `middleware1.wrap_model_call()` → `middleware2.wrap_model_call()` → `middleware3.wrap_model_call()` → model

**After 钩子按逆序运行：**

8. `middleware3.after_model()`
9. `middleware2.after_model()`
10. `middleware1.after_model()`

**智能体循环结束**

11. `middleware3.after_agent()`
12. `middleware2.after_agent()`
13. `middleware1.after_agent()`

**关键规则：**

- `before_*` 钩子：从第一个到最后一个
- `after_*` 钩子：从最后一个到第一个（逆序）
- `wrap_*` 钩子：嵌套（第一个中间件包装所有其他中间件）

## 智能体跳转

要从中间件提前退出，请返回带有 `jump_to` 的字典：

**可用的跳转目标：**

- `'end'`：跳转到智能体执行的末尾（或第一个 `after_agent` 钩子）
- `'tools'`：跳转到工具节点
- `'model'`：跳转到模型节点（或第一个 `before_model` 钩子）

**装饰器方式：**

```python
from langchain.agents.middleware import after_model, hook_config, AgentState
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any


@after_model
@hook_config(can_jump_to=["end"])
def check_for_blocked(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    last_message = state["messages"][-1]
    if "BLOCKED" in last_message.content:
        return {
            "messages": [AIMessage("I cannot respond to that request.")],
            "jumpTo": "end"
        }
    return None
```

**类方式：**

```python
from langchain.agents.middleware import AgentMiddleware, hook_config, AgentState
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any

class BlockedContentMiddleware(AgentMiddleware):
    @hook_config(can_jump_to=["end"])
    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        last_message = state["messages"][-1]
        if "BLOCKED" in last_message.content:
            return {
                "messages": [AIMessage("I cannot respond to that request.")],
                "jumpTo": "end"
            }
        return None
```

## 最佳实践

1. 保持中间件专注 — 每个应该做好一件事
2. 妥善处理错误 — 不要让中间件错误使智能体崩溃
3. **使用适当类型的钩子**：
   - 节点风格用于顺序逻辑（日志记录、验证）
   - 包装风格用于控制流（重试、回退、缓存）
4. 清楚地记录任何自定义状态属性
5. 在集成之前独立地对中间件进行单元测试
6. 考虑执行顺序 — 将关键中间件放在列表中的第一位
7. 尽可能使用内置中间件

## 示例

### 动态模型选择

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.chat_models import init_chat_model
from typing import Callable


complex_model = init_chat_model("gpt-4o")
simple_model = init_chat_model("gpt-4o-mini")


@wrap_model_call
def dynamic_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    # 根据对话长度使用不同的模型
    if len(request.messages) > 10:
        model = complex_model
    else:
        model = simple_model
    return handler(request.override(model=model))
```

### 工具调用监控

```python
from langchain.agents.middleware import wrap_tool_call
from langchain.tools.tool_node import ToolCallRequest
from langchain.messages import ToolMessage
from langgraph.types import Command
from typing import Callable


@wrap_tool_call
def monitor_tool(
    request: ToolCallRequest,
    handler: Callable[[ToolCallRequest], ToolMessage | Command],
) -> ToolMessage | Command:
    print(f"Executing tool: {request.tool_call['name']}")
    print(f"Arguments: {request.tool_call['args']}")
    try:
        result = handler(request)
        print(f"Tool completed successfully")
        return result
    except Exception as e:
        print(f"Tool failed: {e}")
        raise
```

### 动态选择工具

在运行时选择相关工具以提高性能和准确性。

**好处：**

- **更短的提示** — 通过只暴露相关工具来降低复杂性
- **更好的准确性** — 模型从更少的选项中正确选择
- **权限控制** — 根据用户访问动态过滤工具

```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable


@wrap_model_call
def select_tools(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    """根据状态/上下文选择相关工具的中间件。"""
    # 根据状态/上下文选择一小部分相关工具
    relevant_tools = select_relevant_tools(request.state, request.runtime)
    return handler(request.override(tools=relevant_tools))


agent = create_agent(
    model="gpt-4o",
    tools=all_tools,  # 所有可用工具需要预先注册
    middleware=[select_tools],
)
```

### 使用系统消息

使用 `ModelRequest` 上的 `system_message` 字段在中间件中修改系统消息。

**示例：向系统消息添加上下文**

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.messages import SystemMessage
from typing import Callable


@wrap_model_call
def add_context(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    # 始终使用内容块
    new_content = list(request.system_message.content_blocks) + [
        {"type": "text", "text": "Additional context."}
    ]
    new_system_message = SystemMessage(content=new_content)
    return handler(request.override(system_message=new_system_message))
```

**使用缓存控制（Anthropic）的示例**

使用 Anthropic 模型时，您可以使用带有缓存控制指令的结构化内容块来缓存大型系统提示：

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.messages import SystemMessage
from typing import Callable


@wrap_model_call
def add_cached_context(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    # 始终使用内容块
    new_content = list(request.system_message.content_blocks) + [
        {
            "type": "text",
            "text": "Here is a large document to analyze:\n\n<document>...</document>",
            # 至此为止的内容被缓存
            "cache_control": {"type": "ephemeral"}
        }
    ]

    new_system_message = SystemMessage(content=new_content)
    return handler(request.override(system_message=new_system_message))
```

**注意：**

- `ModelRequest.system_message` 始终是一个 `SystemMessage` 对象，即使智能体是用 `system_prompt="string"` 创建的
- 使用 `SystemMessage.content_blocks` 将内容作为块列表访问，无论原始内容是字符串还是列表
- 修改系统消息时，使用 `content_blocks` 并追加新块以保留现有结构
- 对于缓存控制等高级用例，您可以直接将 `SystemMessage` 对象传递给 `create_agent` 的 `system_prompt` 参数

## 其他资源

- [中间件 API 参考](https://reference.langchain.com/python/langchain/middleware/)
- [内置中间件(https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [测试智能体(https://docs.langchain.com/oss/python/langchain/test)

---


