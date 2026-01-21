# 长期记忆

## 概述

LangChain agent 使用 [LangGraph 持久化](/oss/python/langgraph/persistence#memory-store) 来启用长期记忆。这是一个更高级的主题，需要了解 LangGraph 才能使用。

## 记忆存储

LangGraph 将长期记忆作为 JSON 文档存储在[存储](/oss/python/langgraph/persistence#memory-store)中。

每个记忆在自定义的 `namespace`（类似于文件夹）和不同的 `key`（类似于文件名）下组织。Namespace 通常包含用户或组织 ID 或其他标签，以便更容易组织信息。

这种结构支持记忆的分层组织。然后通过内容过滤器支持跨命名空间搜索。

```python
from langgraph.store.memory import InMemoryStore


def embed(texts: list[str]) -> list[list[float]]:
    # 替换为实际的嵌入函数或 LangChain 嵌入对象
    return [[1.0, 2.0] * len(texts)]


# InMemoryStore 将数据保存到内存字典中。在生产环境中使用数据库支持的存储。
store = InMemoryStore(index={"embed": embed, "dims": 2})
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context)
store.put(
    namespace,
    "a-memory",
    {
        "rules": [
            "User likes short, direct language",
            "User only speaks English & python",
        ],
        "my-key": "my-value",
    },
)
# 通过 ID 获取"记忆"
item = store.get(namespace, "a-memory")
# 在此命名空间内搜索"记忆"，按内容等价性过滤，按向量相似性排序
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="language preferences"
)
```

有关记忆存储的更多信息，请参阅[持久化](/oss/python/langgraph/persistence#memory-store)指南。

## 在工具中读取长期记忆

```python
from dataclasses import dataclass

from langchain_core.runnables import RunnableConfig
from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime
from langgraph.store.memory import InMemoryStore


@dataclass
class Context:
    user_id: str

# InMemoryStore 将数据保存到内存字典中。在生产环境中使用数据库支持的存储。
store = InMemoryStore()

# 使用 put 方法将示例数据写入存储
store.put(
    ("users",),  # 命名空间，用于将相关数据分组在一起（用户数据的用户命名空间）
    "user_123",  # 命名空间内的键（用户 ID 作为键）
    {
        "name": "John Smith",
        "language": "English",
    }  # 为给定用户存储的数据
)

@tool
def get_user_info(runtime: ToolRuntime[Context]) -> str:
    """查找用户信息。"""
    # 访问存储 — 与提供给 `create_agent` 的相同
    store = runtime.store
    user_id = runtime.context.user_id
    # 从存储中检索数据 — 返回带有值和元数据的 StoreValue 对象
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"

agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[get_user_info],
    # 将存储传递给 agent — 使 agent 能够在运行工具时访问存储
    store=store,
    context_schema=Context
)

# 运行 agent
agent.invoke(
    {"messages": [{"role": "user", "content": "look up user information"}]},
    context=Context(user_id="user_123")
)
```

## 从工具写入长期记忆

```python
from dataclasses import dataclass
from typing_extensions import TypedDict

from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime
from langgraph.store.memory import InMemoryStore


# InMemoryStore 将数据保存到内存字典中。在生产环境中使用数据库支持的存储。
store = InMemoryStore()

@dataclass
class Context:
    user_id: str

# TypedDict 为 LLM 定义用户信息的结构
class UserInfo(TypedDict):
    name: str

# 允许 agent 更新用户信息的工具（对聊天应用程序很有用）
@tool
def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
    """保存用户信息。"""
    # 访问存储 — 与提供给 `create_agent` 的相同
    store = runtime.store
    user_id = runtime.context.user_id
    # 将数据存储在存储中（命名空间、键、数据）
    store.put(("users",), user_id, user_info)
    return "Successfully saved user info."

agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[save_user_info],
    store=store,
    context_schema=Context
)

# 运行 agent
agent.invoke(
    {"messages": [{"role": "user", "content": "My name is John Smith"}]},
    # 在上下文中传递 user_id 以标识正在更新谁的信息
    context=Context(user_id="user_123")
)

# 您可以直接访问存储以获取值
store.get(("users",), "user_123").value
```
