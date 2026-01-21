# 模型上下文协议 (MCP)

[模型上下文协议 (MCP)](https://modelcontextprotocol.io/introduction) 是一个开放协议，标准化了应用程序如何向 LLM 提供工具和上下文。LangChain 智能体可以使用 [`langchain-mcp-adapters`](https://github.com/langchain-ai/langchain-mcp-adapters) 库来使用 MCP 服务器上定义的工具。

## 快速开始

安装 `langchain-mcp-adapters` 库：

**pip 安装：**

```bash
pip install langchain-mcp-adapters
```

**uv 安装：**

```bash
uv add langchain-mcp-adapters
```

`langchain-mcp-adapters` 使智能体能够使用一个或多个 MCP 服务器上定义的工具。

<Note>
  `MultiServerMCPClient` **默认是无状态的**。每次工具调用都会创建一个新的 MCP `ClientSession`，执行工具，然后清理。请参阅[有状态会话](#有状态会话)部分了解更多详情。
</Note>

```python 访问多个 MCP 服务器
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent


client = MultiServerMCPClient(
    {
        "math": {
            "transport": "stdio",  # 本地子进程通信
            "command": "python",
            # math_server.py 文件的绝对路径
            "args": ["/path/to/math_server.py"],
        },
        "weather": {
            "transport": "http",  # 基于 HTTP 的远程服务器
            # 确保在端口 8000 上启动天气服务器
            "url": "http://localhost:8000/mcp",
        }
    }
)

tools = await client.get_tools()
agent = create_agent(
    "claude-sonnet-4-5-20250929",
    tools
)
math_response = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "what's (3 + 5) x 12?"}]}
)
weather_response = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "what is the weather in nyc?"}]}
)
```

## 自定义服务器

要创建自定义 MCP 服务器，请使用 [FastMCP](https://gofastmcp.com/getting-started/welcome) 库：

**pip 安装：**

```bash
pip install fastmcp
```

**uv 安装：**

```bash
uv add fastmcp
```

要使用 MCP 工具服务器测试您的智能体，请使用以下示例：

**数学服务器 (stdio 传输)：**

```python
from fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """将两个数字相加"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """将两个数字相乘"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

**天气服务器 (可流式 HTTP 传输)：**

```python
from fastmcp import FastMCP

mcp = FastMCP("Weather")

@mcp.tool()
async def get_weather(location: str) -> str:
    """获取地点的天气。"""
    return "It's always sunny in New York"

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

## 传输方式

MCP 支持不同的客户端-服务器通信传输机制。

### HTTP

`http` 传输（也称为 `streamable-http`）使用 HTTP 请求进行客户端-服务器通信。更多详情请参阅 [MCP HTTP 传输规范](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http)。

```python
client = MultiServerMCPClient(
    {
        "weather": {
            "transport": "http",
            "url": "http://localhost:8000/mcp",
        }
    }
)
```

#### 传递标头

通过 HTTP 连接到 MCP 服务器时，可以使用连接配置中的 `headers` 字段包含自定义标头（例如，用于身份验证或追踪）。这支持 `sse`（MCP 规范已弃用）和 `streamable_http` 传输。

```python 使用 MultiServerMCPClient 传递标头
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient(
    {
        "weather": {
            "transport": "http",
            "url": "http://localhost:8000/mcp",
            "headers": {
                "Authorization": "Bearer YOUR_TOKEN",
                "X-Custom-Header": "custom-value"
            },
        }
    }
)
tools = await client.get_tools()
agent = create_agent("openai:gpt-4.1", tools)
response = await agent.ainvoke({"messages": "what is the weather in nyc?"})
```

#### 身份验证

`langchain-mcp-adapters` 库在底层使用官方的 [MCP SDK](https://github.com/modelcontextprotocol/python-sdk)，这允许您通过实现 `httpx.Auth` 接口来提供自定义身份验证机制。

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient(
    {
        "weather": {
            "transport": "http",
            "url": "http://localhost:8000/mcp",
            "auth": auth,
        }
    }
)
```

* [自定义身份验证实现示例](https://github.com/modelcontextprotocol/python-sdk/blob/main/examples/clients/simple-auth-client/mcp_simple_auth_client/main.py)
* [内置 OAuth 流程](https://github.com/modelcontextprotocol/python-sdk/blob/main/src/mcp/client/auth.py#L179)

### stdio

客户端将服务器作为子进程启动，并通过标准输入/输出进行通信。适用于本地工具和简单设置。

<Note>
  与 HTTP 传输不同，`stdio` 连接本质上是**有状态的**——子进程在客户端连接的生命周期内持续存在。但是，在没有显式会话管理的情况下使用 `MultiServerMCPClient` 时，每次工具调用仍然会创建一个新会话。请参阅[有状态会话](#有状态会话)了解如何管理持久连接。
</Note>

```python
client = MultiServerMCPClient(
    {
        "math": {
            "transport": "stdio",
            "command": "python",
            "args": ["/path/to/math_server.py"],
        }
    }
)
```

## 有状态会话

默认情况下，`MultiServerMCPClient` 是**无状态的**——每次工具调用都会创建一个新的 MCP 会话，执行工具，然后清理。

如果您需要控制 MCP 会话的[生命周期](https://modelcontextprotocol.io/specification/2025-03-26/basic/lifecycle)（例如，当使用跨工具调用维护上下文的有状态服务器时），可以使用 `client.session()` 创建一个持久的 `ClientSession`。

```python 使用 MCP ClientSession 进行有状态工具使用
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain.agents import create_agent

client = MultiServerMCPClient({...})

# 显式创建一个会话
async with client.session("server_name") as session:
    # 传递会话以加载工具、资源或提示
    tools = await load_mcp_tools(session)
    agent = create_agent(
        "anthropic:claude-3-7-sonnet-latest",
        tools
    )
```

## 核心功能

### 工具

[工具](https://modelcontextprotocol.io/docs/concepts/tools) 允许 MCP 服务器公开可执行函数，LLM 可以调用这些函数来执行操作——例如查询数据库、调用 API 或与外部系统交互。LangChain 将 MCP 工具转换为 LangChain [工具](/oss/python/langchain/tools)，使它们可以直接在任何 LangChain 智能体或工作流中使用。

#### 加载工具

使用 `client.get_tools()` 从 MCP 服务器检索工具并将其传递给您的智能体：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient({...})
tools = await client.get_tools()
agent = create_agent("claude-sonnet-4-5-20250929", tools)
```

#### 结构化内容

MCP 工具可以在人类可读的文本响应之外返回[结构化内容](https://modelcontextprotocol.io/specification/2025-03-26/server/tools#structured-content)。当工具需要返回机器可解析数据（如 JSON）以及显示给模型的文本时，这很有用。

当 MCP 工具返回 `structuredContent` 时，适配器会将其包装在 [`MCPToolArtifact`](/docs/reference/langchain-mcp-adapters#MCPToolArtifact) 中并将其作为工具的产物返回。您可以使用 `ToolMessage` 上的 `artifact` 字段来访问它。您还可以使用[拦截器](#工具拦截器)来自动处理或转换结构化内容。

**从产物中提取结构化内容：**

调用智能体后，您可以访问响应中工具消息的结构化内容：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent
from langchain.messages import ToolMessage

client = MultiServerMCPClient({...})
tools = await client.get_tools()
agent = create_agent("claude-sonnet-4-5-20250929", tools)

result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "Get data from the server"}]}
)

# 从工具消息中提取结构化内容
for message in result["messages"]:
    if isinstance(message, ToolMessage) and message.artifact:
        structured_content = message.artifact["structured_content"]
```

**通过拦截器追加结构化内容：**

如果希望结构化内容在对话历史中可见（对模型可见），可以使用[拦截器](#工具拦截器)自动将结构化内容追加到工具结果中：

```python
import json

from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from mcp.types import TextContent

async def append_structured_content(request: MCPToolCallRequest, handler):
    """将产物中的结构化内容追加到工具消息中。"""
    result = await handler(request)
    if result.structuredContent:
        result.content += [
            TextContent(type="text", text=json.dumps(result.structuredContent)),
        ]
    return result

client = MultiServerMCPClient({...}, tool_interceptors=[append_structured_content])
```

#### 多模态工具内容

MCP 工具可以在其响应中返回[多模态内容](https://modelcontextprotocol.io/specification/2025-03-26/server/tools#tool-result)（图像、文本等）。当 MCP 服务器返回包含多个部分的内容（例如文本和图像）时，适配器会将它们转换为 LangChain 的[标准内容块](/oss/python/langchain/messages#standard-content-blocks)。您可以通过 `ToolMessage` 上的 `content_blocks` 属性访问标准化表示：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient({...})
tools = await client.get_tools()
agent = create_agent("claude-sonnet-4-5-20250929", tools)

result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "Take a screenshot of the current page"}]}
)

# 从工具消息访问多模态内容
for message in result["messages"]:
    if message.type == "tool":
        # 提供者原生格式的原始内容
        print(f"Raw content: {message.content}")

        # 标准化的内容块
        for block in message.content_blocks:
            if block["type"] == "text":
                print(f"Text: {block['text']}")
            elif block["type"] == "image":
                print(f"Image URL: {block.get('url')}")
                print(f"Image base64: {block.get('base64', '')[:50]}...")
```

这使您能够以与提供者无关的方式处理多模态工具响应，无论底层 MCP 服务器如何格式化其内容。

### 资源

[资源](https://modelcontextprotocol.io/docs/concepts/resources) 允许 MCP 服务器公开可供客户端读取的数据——例如文件、数据库记录或 API 响应。LangChain 将 MCP 资源转换为 [Blob](/docs/reference/langchain-core/documents#Blob) 对象，为处理文本和二进制内容提供统一接口。

#### 加载资源

使用 `client.get_resources()` 从 MCP 服务器加载资源：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({...})

# 从服务器加载所有资源
blobs = await client.get_resources("server_name")

# 或按 URI 加载特定资源
blobs = await client.get_resources("server_name", uris=["file:///path/to/file.txt"])

for blob in blobs:
    print(f"URI: {blob.metadata['uri']}, MIME type: {blob.mimetype}")
    print(blob.as_string())  # 对于文本内容
```

您也可以直接对会话使用 [`load_mcp_resources`](/docs/reference/langchain-mcp-adapters#load_mcp_resources) 以获得更多控制：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.resources import load_mcp_resources

client = MultiServerMCPClient({...})

async with client.session("server_name") as session:
    # 加载所有资源
    blobs = await load_mcp_resources(session)

    # 或按 URI 加载特定资源
    blobs = await load_mcp_resources(session, uris=["file:///path/to/file.txt"])
```

### 提示

[提示](https://modelcontextprotocol.io/docs/concepts/prompts) 允许 MCP 服务器公开可被客户端检索和使用的可重用提示模板。LangChain 将 MCP 提示转换为[消息](/docs/concepts/messages)，使它们易于集成到基于聊天的工作流中。

#### 加载提示

使用 `client.get_prompt()` 从 MCP 服务器加载提示：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({...})

# 按名称加载提示
messages = await client.get_prompt("server_name", "summarize")

# 加载带参数的提示
messages = await client.get_prompt(
    "server_name",
    "code_review",
    arguments={"language": "python", "focus": "security"}
)

# 在工作流中使用消息
for message in messages:
    print(f"{message.type}: {message.content}")
```

您也可以直接对会话使用 [`load_mcp_prompt`](/docs/reference/langchain-mcp-adapters#load_mcp_prompt) 以获得更多控制：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.prompts import load_mcp_prompt

client = MultiServerMCPClient({...})

async with client.session("server_name") as session:
    # 按名称加载提示
    messages = await load_mcp_prompt(session, "summarize")

    # 加载带参数的提示
    messages = await load_mcp_prompt(
        session,
        "code_review",
        arguments={"language": "python", "focus": "security"}
    )
```

## 高级功能

### 工具拦截器

MCP 服务器作为独立进程运行——它们无法访问 LangGraph 运行时信息，如[存储](/oss/python/langgraph/persistence#memory-store)、[上下文](/oss/python/langchain/context-engineering)或智能体状态。**拦截器通过在 MCP 工具执行期间为您提供对此运行时上下文的访问来弥合这一差距。**

拦截器还提供类似中间件的工具调用控制：您可以修改请求、实现重试、动态添加标头，或完全短路执行。

| 部分 | 描述 |
| ---- | ---- |
| [访问运行时上下文](#访问运行时上下文) | 读取用户 ID、API 密钥、存储数据和智能体状态 |
| [状态更新和命令](#状态更新和命令) | 使用 `Command` 更新智能体状态或控制图流程 |
| [编写拦截器](#自定义拦截器) | 修改请求、组合拦截器和错误处理的模式 |

#### 访问运行时上下文

当 MCP 工具在 LangChain 智能体中使用时（通过 `create_agent`），拦截器会收到访问 `ToolRuntime` 上下文的权限。这提供了对工具调用 ID、状态、配置和存储的访问——为访问用户数据、持久化信息和控制智能体行为提供了强大的模式。

**运行时上下文：**

访问特定于用户的配置，如用户 ID、API 密钥或权限，这些在调用时传递：

```python 将用户上下文注入 MCP 工具调用
from dataclasses import dataclass
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from langchain.agents import create_agent

@dataclass
class Context:
    user_id: str
    api_key: str

async def inject_user_context(
    request: MCPToolCallRequest,
    handler,
):
    """将用户凭证注入 MCP 工具调用。"""
    runtime = request.runtime
    user_id = runtime.context.user_id
    api_key = runtime.context.api_key

    # 将用户上下文添加到工具参数
    modified_request = request.override(
        args={**request.args, "user_id": user_id}
    )
    return await handler(modified_request)

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[inject_user_context],
)
tools = await client.get_tools()
agent = create_agent("gpt-4o", tools, context_schema=Context)

# 使用用户上下文调用
result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "Search my orders"}]},
    context={"user_id": "user_123", "api_key": "sk-..."}
)
```

**存储：**

访问长期记忆以检索用户偏好或在对话之间持久化数据：

```python 从存储中访问用户偏好
from dataclasses import dataclass
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from langchain.agents import create_agent
from langgraph.store.memory import InMemoryStore

@dataclass
class Context:
    user_id: str

async def personalize_search(
    request: MCPToolCallRequest,
    handler,
):
    """使用存储的偏好来个性化 MCP 工具调用。"""
    runtime = request.runtime
    user_id = runtime.context.user_id
    store = runtime.store

    # 从存储中读取用户偏好
    prefs = store.get(("preferences",), user_id)

    if prefs and request.name == "search":
        # 应用用户首选语言和结果限制
        modified_args = {
            **request.args,
            "language": prefs.value.get("language", "en"),
            "limit": prefs.value.get("result_limit", 10),
        }
        request = request.override(args=modified_args)

    return await handler(request)

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[personalize_search],
)
tools = await client.get_tools()
agent = create_agent(
    "gpt-4o",
    tools,
    context_schema=Context,
    store=InMemoryStore()
)
```

**状态：**

访问对话状态以根据当前会话做出决策：

```python 根据身份验证状态过滤工具
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from langchain.messages import ToolMessage

async def require_authentication(
    request: MCPToolCallRequest,
    handler,
):
    """如果用户未经过身份验证，则阻止敏感的 MCP 工具。"""
    runtime = request.runtime
    state = runtime.state
    is_authenticated = state.get("authenticated", False)

    sensitive_tools = ["delete_file", "update_settings", "export_data"]

    if request.name in sensitive_tools and not is_authenticated:
        # 返回错误而不是调用工具
        return ToolMessage(
            content="Authentication required. Please log in first.",
            tool_call_id=runtime.tool_call_id,
        )

    return await handler(request)

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[require_authentication],
)
```

**工具调用 ID：**

访问工具调用 ID 以返回正确格式化的响应或跟踪工具执行：

```python 使用工具调用 ID 返回自定义响应
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from langchain.messages import ToolMessage

async def rate_limit_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """对昂贵的 MCP 工具调用进行速率限制。"""
    runtime = request.runtime
    tool_call_id = runtime.tool_call_id

    # 检查速率限制（简化示例）
    if is_rate_limited(request.name):
        return ToolMessage(
            content="Rate limit exceeded. Please try again later.",
            tool_call_id=tool_call_id,
        )

    result = await handler(request)

    # 记录成功的工具调用
    log_tool_execution(tool_call_id, request.name, success=True)

    return result

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[rate_limit_interceptor],
)
```

有关更多上下文工程模式，请参阅[上下文工程](/oss/python/langchain/context-engineering)和[工具](/oss/python/langchain/tools)。

#### 状态更新和命令

拦截器可以返回 `Command` 对象来更新智能体状态或控制图执行流程。这对于跟踪任务进度、在智能体之间切换或提前结束执行很有用。

```python 标记任务完成并切换智能体
from langchain.agents import AgentState, create_agent
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from langchain.messages import ToolMessage
from langgraph.types import Command

async def handle_task_completion(
    request: MCPToolCallRequest,
    handler,
):
    """标记任务完成并移交给摘要智能体。"""
    result = await handler(request)

    if request.name == "submit_order":
        return Command(
            update={
                "messages": [result] if isinstance(result, ToolMessage) else [],
                "task_status": "completed",
            },
            goto="summary_agent",
        )

    return result
```

使用 `Command` 和 `goto="__end__"` 提前结束执行：

```python 任务完成时结束智能体运行
async def end_on_success(
    request: MCPToolCallRequest,
    handler,
):
    """当任务标记为完成时结束智能体运行。"""
    result = await handler(request)

    if request.name == "mark_complete":
        return Command(
            update={"messages": [result], "status": "done"},
            goto="__end__",
        )

    return result
```

#### 自定义拦截器

拦截器是包装工具执行的异步函数，支持请求/响应修改、重试逻辑和其他横切关注点。它们遵循"洋葱"模式，列表中的第一个拦截器是最外层。

**基本模式：**

拦截器是一个接收请求和处理程序的异步函数。您可以在调用处理程序之前修改请求，在之后修改响应，或完全跳过处理程序。

```python 基本拦截器模式
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest

async def logging_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """在执行前后记录工具调用。"""
    print(f"Calling tool: {request.name} with args: {request.args}")
    result = await handler(request)
    print(f"Tool {request.name} returned: {result}")
    return result

client = MultiServerMCPClient(
    {"math": {"transport": "stdio", "command": "python", "args": ["/path/to/server.py"]}},
    tool_interceptors=[logging_interceptor],
)
```

**修改请求：**

使用 `request.override()` 创建修改后的请求。这遵循不可变模式，保持原始请求不变。

```python 修改工具参数
async def double_args_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """在执行前将所有数字参数加倍。"""
    modified_args = {k: v * 2 for k, v in request.args.items()}
    modified_request = request.override(args=modified_args)
    return await handler(modified_request)

# 原始调用：add(a=2, b=3) 变成 add(a=4, b=6)
```

**在运行时修改标头：**

拦截器可以根据请求上下文动态修改 HTTP 标头：

```python 动态标头修改
async def auth_header_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """根据被调用的工具添加身份验证标头。"""
    token = get_token_for_tool(request.name)
    modified_request = request.override(
        headers={"Authorization": f"Bearer {token}"}
    )
    return await handler(modified_request)
```

**组合拦截器：**

多个拦截器按"洋葱"顺序组合——列表中的第一个拦截器是最外层：

```python 组合多个拦截器
async def outer_interceptor(request, handler):
    print("outer: before")
    result = await handler(request)
    print("outer: after")
    return result

async def inner_interceptor(request, handler):
    print("inner: before")
    result = await handler(request)
    print("inner: after")
    return result

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[outer_interceptor, inner_interceptor],
)

# 执行顺序：
# outer: before -> inner: before -> 工具执行 -> inner: after -> outer: after
```

**错误处理：**

使用拦截器捕获工具执行错误并实现重试逻辑：

```python 错误时重试
import asyncio

async def retry_interceptor(
    request: MCPToolCallRequest,
    handler,
    max_retries: int = 3,
    delay: float = 1.0,
):
    """使用指数退避重试失败的工具调用。"""
    last_error = None
    for attempt in range(max_retries):
        try:
            return await handler(request)
        except Exception as e:
            last_error = e
            if attempt < max_retries - 1:
                wait_time = delay * (2 ** attempt)  # 指数退避
                print(f"Tool {request.name} failed (attempt {attempt + 1}), retrying in {wait_time}s...")
                await asyncio.sleep(wait_time)
    raise last_error

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[retry_interceptor],
)
```

您还可以捕获特定错误类型并返回回退值：

```python 带回退的错误处理
async def fallback_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """如果工具执行失败则返回回退值。"""
    try:
        return await handler(request)
    except TimeoutError:
        return f"Tool {request.name} timed out. Please try again later."
    except ConnectionError:
        return f"Could not connect to {request.name} service. Using cached data."
```

### 进度通知

订阅长时间运行工具执行的进度更新：

```python 进度回调
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.callbacks import Callbacks, CallbackContext

async def on_progress(
    progress: float,
    total: float | None,
    message: str | None,
    context: CallbackContext,
):
    """处理来自 MCP 服务器的进度更新。"""
    percent = (progress / total * 100) if total else progress
    tool_info = f" ({context.tool_name})" if context.tool_name else ""
    print(f"[{context.server_name}{tool_info}] Progress: {percent:.1f}% - {message}")

client = MultiServerMCPClient(
    {...},
    callbacks=Callbacks(on_progress=on_progress),
)
```

`CallbackContext` 提供：

* `server_name`：MCP 服务器的名称
* `tool_name`：正在执行的工具名称（在工具调用期间可用）

### 日志记录

MCP 协议支持来自服务器的[日志](https://modelcontextprotocol.io/specification/2025-03-26/server/utilities/logging#log-levels)通知。使用 `Callbacks` 类订阅这些事件。

```python 日志回调
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.callbacks import Callbacks, CallbackContext
from mcp.types import LoggingMessageNotificationParams

async def on_logging_message(
    params: LoggingMessageNotificationParams,
    context: CallbackContext,
):
    """处理来自 MCP 服务器的日志消息。"""
    print(f"[{context.server_name}] {params.level}: {params.data}")

client = MultiServerMCPClient(
    {...},
    callbacks=Callbacks(on_logging_message=on_logging_message),
)
```

### 征求

[征求](https://modelcontextprotocol.io/specification/2025-11-25/client/elicitation#elicitation) 允许 MCP 服务器在工具执行期间向用户请求额外输入。不是要求所有输入 upfront，服务器可以交互式地根据需要请求信息。

#### 服务器设置

定义使用 `ctx.elicit()` 的工具来请求带有模式的用户输入：

```python 带征求的 MCP 服务器
from pydantic import BaseModel
from mcp.server.fastmcp import Context, FastMCP

server = FastMCP("Profile")

class UserDetails(BaseModel):
    email: str
    age: int

@server.tool()
async def create_profile(name: str, ctx: Context) -> str:
    """创建用户配置文件，通过征求请求详细信息。"""
    result = await ctx.elicit(
        message=f"Please provide details for {name}'s profile:",
        schema=UserDetails,
    )
    if result.action == "accept" and result.data:
        return f"Created profile for {name}: email={result.data.email}, age={result.data.age}"
    if result.action == "decline":
        return f"User declined. Created minimal profile for {name}."
    return "Profile creation cancelled."

if __name__ == "__main__":
    server.run(transport="http")
```

#### 客户端设置

通过向 `MultiServerMCPClient` 提供回调来处理征求请求：

```python 处理征求请求
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.callbacks import Callbacks, CallbackContext
from mcp.shared.context import RequestContext
from mcp.types import ElicitRequestParams, ElicitResult

async def on_elicitation(
    mcp_context: RequestContext,
    params: ElicitRequestParams,
    context: CallbackContext,
) -> ElicitResult:
    """处理来自 MCP 服务器的征求请求。"""
    # 在实际应用中，您会根据 params.message 和 params.requestedSchema 提示用户输入
    return ElicitResult(
        action="accept",
        content={"email": "user@example.com", "age": 25},
    )

client = MultiServerMCPClient(
    {
        "profile": {
            "url": "http://localhost:8000/mcp",
            "transport": "http",
        }
    },
    callbacks=Callbacks(on_elicitation=on_elicitation),
)
```

#### 响应操作

征求回调可以返回三种操作之一：

| 操作 | 描述 |
| ---- | ---- |
| `accept` | 用户提供了有效输入。在 `content` 字段中包含数据。 |
| `decline` | 用户选择不提供请求的信息。 |
| `cancel` | 用户完全取消操作。 |

```python 响应操作示例
# 接受并带数据
ElicitResult(action="accept", content={"email": "user@example.com", "age": 25})

# 拒绝（用户不想提供信息）
ElicitResult(action="decline")

# 取消（中止操作）
ElicitResult(action="cancel")
```

## 其他资源

* [MCP 文档](https://modelcontextprotocol.io/introduction)
* [MCP 传输文档](https://modelcontextprotocol.io/docs/concepts/transports)
* [`langchain-mcp-adapters`](https://github.com/langchain-ai/langchain-mcp-adapters)




