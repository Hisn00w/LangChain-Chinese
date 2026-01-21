# Agent 中的上下文工程

## 概述

构建 agent（或任何 LLM 应用程序）的难点在于使其足够可靠。虽然它们可能适用于原型设计，但在实际使用案例中往往会失败。

### 为什么 agent 会失败？

当 agent 失败时，通常是因为 agent 内部的 LLM 调用采取了错误的操作或没有做我们期望的事情。LLM 失败的原因有两个：

1. 底层的 LLM 能力不足
2. 没有向 LLM 传递"正确的"上下文

更多时候，实际上是第二个原因导致 agent 不可靠。

**上下文工程**是以正确的格式提供正确的信息和工具，使 LLM 能够完成任务。这是 AI 工程师的首要工作。这种"正确"上下文的缺乏是实现更可靠 agent 的最大障碍，而 LangChain 的 agent 抽象经过独特设计，以促进上下文工程。

不熟悉上下文工程？请从[概念概述](/oss/python/concepts/context)开始，了解不同类型的上下文以及何时使用它们。

### Agent 循环

典型的 agent 循环由两个主要步骤组成：

1. **模型调用** — 使用提示和可用工具调用 LLM，返回响应或执行工具的请求
2. **工具执行** — 执行 LLM 请求的工具，返回工具结果

此循环持续进行，直到 LLM 决定完成。

### 您可以控制的内容

要构建可靠的 agent，您需要控制 agent 循环每个步骤中发生的情况，以及步骤之间发生的情况。

| 上下文类型 | 您控制的内容 | 瞬态还是持久 |
| ---------- | ------------ | ------------ |
| **模型上下文** | 进入模型调用的内容（指令、消息历史、工具、响应格式） | 瞬态 |
| **工具上下文** | 工具可以访问和产生的内容（对状态的读写、存储、运行时上下文） | 持久 |
| **生命周期上下文** | 在模型和工具调用之间发生的情况（总结、护栏、日志记录等） | 持久 |

- **瞬态上下文** — LLM 单次调用看到的内容。您可以修改消息、工具或提示，而不会更改状态中保存的内容。
- **持久上下文** — 跨轮次保存在状态中的内容。生命周期钩子和工具写入会永久修改此内容。

### 数据源

在此过程中，您的 agent 访问（读取/写入）不同的数据源：

| 数据源 | 也称为 | 范围 | 示例 |
| ------ | ------ | ---- | ---- |
| **运行时上下文** | 静态配置 | 会话范围 | 用户 ID、API 密钥、数据库连接、权限、环境设置 |
| **状态** | 短期记忆 | 会话范围 | 当前消息、上传的文件、身份验证状态、工具结果 |
| **存储** | 长期记忆 | 跨会话 | 用户偏好、提取的见解、记忆、历史数据 |

### 工作原理

LangChain [中间件](/oss/python/langchain/middleware) 是使使用 LangChain 的开发者能够实际进行上下文工程的底层机制。

中间件允许您钩入 agent 生命周期的任何步骤，并：

- 更新上下文
- 跳转到 agent 生命周期的不同步骤

在本指南中，您将频繁看到中间件 API 的使用，作为实现上下文工程目的的手段。

## 模型上下文

控制进入每次模型调用的内容 — 指令、可用的工具、要使用的模型以及输出格式。这些决策直接影响可靠性和成本。

- **系统提示** — 开发者向 LLM 提供的基本指令
- **消息** — 发送给 LLM 的完整消息列表（对话历史）
- **工具** — agent 可用于执行操作的实用工具
- **模型** — 要调用的实际模型（包括配置）
- **响应格式** — 模型最终响应的模式规范

所有这些类型的模型上下文都可以从**状态**（短期记忆）、**存储**（长期记忆）或**运行时上下文**（静态配置）中获取。

### 系统提示

系统提示设置 LLM 的行为和能力。不同的用户、上下文或对话阶段需要不同的指令。成功的 agent 会利用记忆、偏好和配置，为对话的当前状态提供正确的指令。

**从状态获取：** 从状态访问消息计数或对话上下文：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import dynamic_prompt, ModelRequest

@dynamic_prompt
def state_aware_prompt(request: ModelRequest) -> str:
    # request.messages 是 request.state["messages"] 的快捷方式
    message_count = len(request.messages)

    base = "You are a helpful assistant."

    if message_count > 10:
        base += "\nThis is a long conversation - be extra concise."

    return base

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[state_aware_prompt]
)
```

**从存储获取：** 从长期记忆中访问用户偏好：

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import dynamic_prompt, ModelRequest
from langgraph.store.memory import InMemoryStore

@dataclass
class Context:
    user_id: str

@dynamic_prompt
def store_aware_prompt(request: ModelRequest) -> str:
    user_id = request.runtime.context.user_id

    # 从存储读取：获取用户偏好
    store = request.runtime.store
    user_prefs = store.get(("preferences",), user_id)

    base = "You are a helpful assistant."

    if user_prefs:
        style = user_prefs.value.get("communication_style", "balanced")
        base += f"\nUser prefers {style} responses."

    return base

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[store_aware_prompt],
    context_schema=Context,
    store=InMemoryStore()
)
```

**从运行时上下文获取：** 从运行时上下文访问用户 ID 或配置：

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import dynamic_prompt, ModelRequest

@dataclass
class Context:
    user_role: str
    deployment_env: str

@dynamic_prompt
def context_aware_prompt(request: ModelRequest) -> str:
    # 从运行时上下文读取：用户角色和环境
    user_role = request.runtime.context.user_role
    env = request.runtime.context.deployment_env

    base = "You are a helpful assistant."

    if user_role == "admin":
        base += "\nYou have admin access. You can perform all operations."
    elif user_role == "viewer":
        base += "\nYou have read-only access. Guide users to read operations only."

    if env == "production":
        base += "\nBe extra careful with any data modifications."

    return base

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[context_aware_prompt],
    context_schema=Context
)
```

### 消息

消息构成发送给 LLM 的提示。管理消息的内容以确保 LLM 有正确的信息来做出良好响应至关重要。

**从状态注入上下文：** 在相关时从状态注入上传的文件上下文：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable

@wrap_model_call
def inject_file_context(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse]
) -> ModelResponse:
    """注入用户在本会话中上传的文件的上下文。"""
    # 从状态读取：获取上传的文件元数据
    uploaded_files = request.state.get("uploaded_files", [])

    if uploaded_files:
        # 构建可用文件的上下文
        file_descriptions = []
        for file in uploaded_files:
            file_descriptions.append(
                f"- {file['name']} ({file['type']}): {file['summary']}"
            )

        file_context = f"""Files you have access to in this conversation:
{chr(10).join(file_descriptions)}

Reference these files when answering questions."""

        # 在最近的消息之前注入文件上下文
        messages = [
            *request.messages,
            {"role": "user", "content": file_context},
        ]
        request = request.override(messages=messages)

    return handler(request)

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[inject_file_context]
)
```

**从存储注入写作风格：** 从存储中注入用户的电子邮件写作风格以指导起草：

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable
from langgraph.store.memory import InMemoryStore

@dataclass
class Context:
    user_id: str

@wrap_model_call
def inject_writing_style(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse]
) -> ModelResponse:
    """从存储中注入用户的电子邮件写作风格。"""
    user_id = request.runtime.context.user_id

    # 从存储读取：获取用户的写作风格示例
    store = request.runtime.store
    writing_style = store.get(("writing_style",), user_id)

    if writing_style:
        style = writing_style.value
        # 从存储的示例构建风格指南
        style_context = f"""Your writing style:
- Tone: {style.get('tone', 'professional')}
- Typical greeting: "{style.get('greeting', 'Hi')}"
- Typical sign-off: "{style.get('sign_off', 'Best')}"
- Example email you've written:
{style.get('example_email', '')}"""

        # 附加在末尾 — 模型更关注最后的消息
        messages = [
            *request.messages,
            {"role": "user", "content": style_context}
        ]
        request = request.override(messages=messages)

    return handler(request)

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[inject_writing_style],
    context_schema=Context,
    store=InMemoryStore()
)
```

**瞬态 vs 持久消息更新：**

上面的示例使用 `wrap_model_call` 进行**瞬态**更新 — 修改单次调用中发送给模型的消息，而不会更改状态中保存的内容。

对于**持久**更新（如[生命周期上下文](#总结)示例中的总结），使用生命周期钩子如 `before_model` 或 `after_model` 来永久更新对话历史。

### 工具

工具让模型与数据库、API 和外部系统交互。您如何定义和选择工具直接影响模型是否能有效完成任务。

**定义工具：**

每个工具需要清晰的名称、描述、参数名称和参数描述。这些不仅仅是元数据 — 它们指导模型推理何时以及如何使用工具。

```python
from langchain.tools import tool

@tool(parse_docstring=True)
def search_orders(
    user_id: str,
    status: str,
    limit: int = 10
) -> str:
    """Search for user orders by status.

    Use this when the user asks about order history or wants to check
    order status. Always filter by the provided status.

    Args:
        user_id: Unique identifier for the user
        status: Order status: 'pending', 'shipped', or 'delivered'
        limit: Maximum number of results to return
    """
    pass
```

**选择工具：**

并非每种情况都适合使用所有工具。太多的工具可能会让模型不知所措（过载上下文）并增加错误；太少则限制能力。动态工具选择根据身份验证状态、用户权限、功能标志或对话阶段调整可用的工具集。

**基于状态筛选工具：** 在某些对话里程碑之后才启用高级工具：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable

@wrap_model_call
def state_based_tools(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse]
) -> ModelResponse:
    """根据对话状态筛选工具。"""
    # 从状态读取：检查用户是否已认证
    state = request.state
    is_authenticated = state.get("authenticated", False)
    message_count = len(state["messages"])

    # 仅在认证后才启用敏感工具
    if not is_authenticated:
        tools = [t for t in request.tools if t.name.startswith("public_")]
        request = request.override(tools=tools)
    elif message_count < 5:
        # 在对话早期限制工具
        tools = [t for t in request.tools if t.name != "advanced_search"]
        request = request.override(tools=tools)

    return handler(request)

agent = create_agent(
    model="gpt-4o",
    tools=[public_search, private_search, advanced_search],
    middleware=[state_based_tools]
)
```

**基于运行时上下文筛选工具：** 根据运行时上下文的用户权限筛选工具：

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable

@dataclass
class Context:
    user_role: str

@wrap_model_call
def context_based_tools(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse]
) -> ModelResponse:
    """根据运行时上下文权限筛选工具。"""
    # 从运行时上下文读取：获取用户角色
    user_role = request.runtime.context.user_role

    if user_role == "admin":
        # 管理员获得所有工具
        pass
    elif user_role == "editor":
        # 编辑者不能删除
        tools = [t for t in request.tools if t.name != "delete_data"]
        request = request.override(tools=tools)
    else:
        # 查看者获得只读工具
        tools = [t for t in request.tools if t.name.startswith("read_")]
        request = request.override(tools=tools)

    return handler(request)

agent = create_agent(
    model="gpt-4o",
    tools=[read_data, write_data, delete_data],
    middleware=[context_based_tools],
    context_schema=Context
)
```

### 模型

不同的模型有不同的优势、成本和上下文窗口。为手头的任务选择正确的模型，这可能在 agent 运行期间发生变化。

**基于状态选择模型：** 根据对话长度使用不同的模型：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.chat_models import init_chat_model
from typing import Callable

# 在外部初始化模型一次
large_model = init_chat_model("claude-sonnet-4-5-20250929")
standard_model = init_chat_model("gpt-4o")
efficient_model = init_chat_model("gpt-4o-mini")

@wrap_model_call
def state_based_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse]
) -> ModelResponse:
    """根据状态对话长度选择模型。"""
    message_count = len(request.messages)

    if message_count > 20:
        # 长对话 — 使用具有更大上下文窗口的模型
        model = large_model
    elif message_count > 10:
        # 中等对话
        model = standard_model
    else:
        # 短对话 — 使用高效模型
        model = efficient_model

    request = request.override(model=model)

    return handler(request)

agent = create_agent(
    model="gpt-4o-mini",
    tools=[...],
    middleware=[state_based_model]
)
```

### 响应格式

结构化输出将非结构化文本转换为经过验证的结构化数据。当提取特定字段或为下游系统返回数据时，自由形式的文本是不够的。

**基于状态配置结构化输出：**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from pydantic import BaseModel, Field
from typing import Callable

class SimpleResponse(BaseModel):
    """早期对话的简单响应。"""
    answer: str = Field(description="A brief answer")

class DetailedResponse(BaseModel):
    """已建立对话的详细响应。"""
    answer: str = Field(description="A detailed answer")
    reasoning: str = Field(description="Explanation of reasoning")
    confidence: float = Field(description="Confidence score 0-1")

@wrap_model_call
def state_based_output(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse]
) -> ModelResponse:
    """根据状态选择输出格式。"""
    message_count = len(request.messages)

    if message_count < 3:
        # 早期对话 — 使用简单格式
        request = request.override(response_format=SimpleResponse)
    else:
        # 已建立对话 — 使用详细格式
        request = request.override(response_format=DetailedResponse)

    return handler(request)

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[state_based_output]
)
```

## 工具上下文

工具的特殊之处在于它们既读取又写入上下文。

在最基本的情况下，当工具执行时，它接收 LLM 的请求参数并返回工具消息。工具完成其工作并产生结果。

工具还可以为模型获取重要信息，使其能够执行和完成任务。

### 读取

大多数现实世界的工具需要的不仅仅是 LLM 的参数。它们需要用于数据库查询的用户 ID、用于外部服务的 API 密钥或用于做出决策的当前会话状态。工具从状态、存储和运行时上下文读取以访问此信息。

**从状态读取：**

```python
from langchain.tools import tool, ToolRuntime
from langchain.agents import create_agent

@tool
def check_authentication(
    runtime: ToolRuntime
) -> str:
    """检查用户是否已认证。"""
    # 从状态读取：检查当前认证状态
    current_state = runtime.state
    is_authenticated = current_state.get("authenticated", False)

    if is_authenticated:
        return "User is authenticated"
    else:
        return "User is not authenticated"

agent = create_agent(
    model="gpt-4o",
    tools=[check_authentication]
)
```

**从存储读取：**

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime
from langchain.agents import create_agent
from langgraph.store.memory import InMemoryStore

@dataclass
class Context:
    user_id: str

@tool
def get_preference(
    preference_key: str,
    runtime: ToolRuntime[Context]
) -> str:
    """从存储获取用户偏好。"""
    user_id = runtime.context.user_id

    # 从存储读取：获取现有偏好
    store = runtime.store
    existing_prefs = store.get(("preferences",), user_id)

    if existing_prefs:
        value = existing_prefs.value.get(preference_key)
        return f"{preference_key}: {value}" if value else f"No preference set for {preference_key}"
    else:
        return "No preferences found"

agent = create_agent(
    model="gpt-4o",
    tools=[get_preference],
    context_schema=Context,
    store=InMemoryStore()
)
```

**从运行时上下文读取：**

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime
from langchain.agents import create_agent

@dataclass
class Context:
    user_id: str
    api_key: str
    db_connection: str

@tool
def fetch_user_data(
    query: str,
    runtime: ToolRuntime[Context]
) -> str:
    """使用运行时上下文配置获取数据。"""
    # 从运行时上下文读取：获取 API 密钥和数据库连接
    user_id = runtime.context.user_id
    api_key = runtime.context.api_key
    db_connection = runtime.context.db_connection

    # 使用配置获取数据
    results = perform_database_query(db_connection, query, api_key)

    return f"Found {len(results)} results for user {user_id}"

agent = create_agent(
    model="gpt-4o",
    tools=[fetch_user_data],
    context_schema=Context
)
```

### 写入

工具结果可用于帮助 agent 完成给定任务。工具可以直接向模型返回结果，并更新 agent 的记忆，使重要上下文对 future 步骤可用。

**写入状态：**

```python
from langchain.tools import tool, ToolRuntime
from langchain.agents import create_agent
from langgraph.types import Command

@tool
def authenticate_user(
    password: str,
    runtime: ToolRuntime
) -> Command:
    """认证用户并更新状态。"""
    # 执行身份验证（简化）
    if password == "correct":
        # 写入状态：使用 Command 标记为已认证
        return Command(
            update={"authenticated": True},
        )
    else:
        return Command(update={"authenticated": False})

agent = create_agent(
    model="gpt-4o",
    tools=[authenticate_user]
)
```

**写入存储：**

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime
from langchain.agents import create_agent
from langgraph.store.memory import InMemoryStore

@dataclass
class Context:
    user_id: str

@tool
def save_preference(
    preference_key: str,
    preference_value: str,
    runtime: ToolRuntime[Context]
) -> str:
    """将用户偏好保存到存储。"""
    user_id = runtime.context.user_id

    # 读取现有偏好
    store = runtime.store
    existing_prefs = store.get(("preferences",), user_id)

    # 与新偏好合并
    prefs = existing_prefs.value if existing_prefs else {}
    prefs[preference_key] = preference_value

    # 写入存储：保存更新后的偏好
    store.put(("preferences",), user_id, prefs)

    return f"Saved preference: {preference_key} = {preference_value}"

agent = create_agent(
    model="gpt-4o",
    tools=[save_preference],
    context_schema=Context,
    store=InMemoryStore()
)
```

## 生命周期上下文

控制**核心 agent 步骤之间**发生的情况 — 拦截数据流以实现跨领域关注，如总结、护栏和日志记录。

正如您在[模型上下文](#模型上下文)和[工具上下文](#工具上下文)中所看到的，[中间件](/oss/python/langchain/middleware) 是使上下文工程可行的机制。中间件允许您钩入 agent 生命周期的任何步骤，并：

1. **更新上下文** — 修改状态和存储以持久化更改，更新对话历史，或保存见解
2. **在生命周期中跳转** — 根据上下文移动到 agent 循环中的不同步骤（例如，如果满足条件则跳过工具执行，使用修改后的上下文重复模型调用）

### 示例：总结

最常见的生命周期模式之一是当对话历史太长时自动压缩对话历史。与[模型上下文](#消息)中显示的瞬态消息修剪不同，总结会**持久更新状态** — 永久替换旧消息，为所有 future 轮次保存摘要。

LangChain 为此提供了内置中间件：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4o-mini",
            trigger={"tokens": 4000},
            keep={"messages": 20},
        ),
    ],
)
```

当对话超过 token 限制时，`SummarizationMiddleware` 自动：

1. 使用单独的 LLM 调用总结旧消息
2. 用状态中的摘要消息替换它们（永久）
3. 保持最近的消息完整以保留上下文

## 最佳实践

1. **从简单开始** — 从静态提示和工具开始，仅在需要时添加动态内容
2. **增量测试** — 一次添加一个上下文工程功能
3. **监控性能** — 跟踪模型调用、token 使用量和延迟
4. **使用内置中间件** — 利用 `SummarizationMiddleware`、`LLMToolSelectorMiddleware` 等
5. **记录您的上下文策略** — 清楚传递什么上下文以及为什么
6. **理解瞬态 vs 持久**：模型上下文更改是瞬态的（每次调用），而生命周期上下文更改会持久化到状态

## 相关资源

- [上下文概念概述](/oss/python/concepts/context) — 了解上下文类型以及何时使用它们
- [中间件](/oss/python/langchain/middleware) — 完整中间件指南
- [工具](/oss/python/langchain/tools) — 工具创建和上下文访问
- [记忆](/oss/python/concepts/memory) — 短期和长期记忆模式
- [Agent](/oss/python/langchain/agents) — 核心 agent 概念

