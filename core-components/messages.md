# 消息

消息是 LangChain 中模型上下文的基本单元。它们表示模型的输入和输出，携带与 LLM 交互时表示对话状态所需的内容和元数据。

消息是包含以下内容的对象：

* **角色** - 标识消息类型（如 `system`、`user`）
* **内容** - 表示消息的实际内容（如文本、图像、音频、文档等）
* **元数据** - 可选字段，如响应信息、消息 ID 和 token 使用量

LangChain 提供了一种跨所有模型提供商都能工作的标准消息类型，确保无论调用哪个模型都有一致的行为。

## 基础用法

使用消息最简单的方法是创建消息对象，并在调用模型时传递它们。

```python
from langchain.chat_models import init_chat_model
from langchain.messages import HumanMessage, AIMessage, SystemMessage

model = init_chat_model("gpt-5-nano")

system_msg = SystemMessage("You are a helpful assistant.")
human_msg = HumanMessage("Hello, how are you?")

# 与聊天模型一起使用
messages = [system_msg, human_msg]
response = model.invoke(messages)  # 返回 AIMessage
```

### 文本提示

文本提示是字符串，非常适合不需要保留对话历史的简单生成任务。

```python
response = model.invoke("Write a haiku about spring")
```

**在以下情况使用文本提示：**

* 您有单独的、独立的需求
* 您不需要对话历史
* 您希望最小化代码复杂性

### 消息提示

或者，您可以通过提供消息对象列表向模型传递消息列表。

```python
from langchain.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage("You are a poetry expert"),
    HumanMessage("Write a haiku about spring"),
    AIMessage("Cherry blossoms bloom...")
]
response = model.invoke(messages)
```

**在以下情况使用消息提示：**

* 管理多轮对话
* 处理多模态内容（图像、音频、文件）
* 包含系统指令

### 字典格式

您也可以直接使用 OpenAI 聊天补全格式指定消息。

```python
messages = [
    {"role": "system", "content": "You are a poetry expert"},
    {"role": "user", "content": "Write a haiku about spring"},
    {"role": "assistant", "content": "Cherry blossoms bloom..."}
]
response = model.invoke(messages)
```

## 消息类型

* 系统消息 - 告诉模型如何行为并为交互提供上下文
* 人类消息 - 表示用户输入和与模型的交互
* AI 消息 - 模型生成的响应，包括文本内容、工具调用和元数据
* 工具消息 - 表示工具调用的输出

### 系统消息

[`SystemMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.SystemMessage) 表示一组初始指令，用于设定模型的行为基调。您可以使用系统消息来设置语气、定义模型角色以及建立响应指南。

```python
system_msg = SystemMessage("You are a helpful coding assistant.")

messages = [
    system_msg,
    HumanMessage("How do I create a REST API?")
]
response = model.invoke(messages)
```

```python
from langchain.messages import SystemMessage, HumanMessage

system_msg = SystemMessage("""
You are a senior Python developer with expertise in web frameworks.
Always provide code examples and explain your reasoning.
Be concise but thorough in your explanations.
""")

messages = [
    system_msg,
    HumanMessage("How do I create a REST API?")
]
response = model.invoke(messages)
```

---

### 人类消息

[`HumanMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.HumanMessage) 表示用户输入和交互。它们可以包含文本、图像、音频、文件以及任何其他多模态内容。

#### 文本内容

```python
response = model.invoke([
    HumanMessage("What is machine learning?")
])
```

```python
# 使用字符串是单个 HumanMessage 的简写
response = model.invoke("What is machine learning?")
```

#### 消息元数据

```python
human_msg = HumanMessage(
    content="Hello!",
    name="alice",  # 可选：识别不同用户
    id="msg_123",  # 可选：用于追踪的唯一标识符
)
```

`name` 字段的行为因提供商而异——有些用于用户识别，有些则忽略它。如需检查，请参阅模型提供商的参考文档。

---

### AI 消息

[`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage) 表示模型调用的输出。它们可以包含多模态数据、工具调用以及您之后可以访问的特定提供商的元数据。

```python
response = model.invoke("Explain AI")
print(type(response))  # <class 'langchain.messages.AIMessage'>
```

[`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage) 对象在调用模型时返回，其中包含响应中的所有相关元数据。

提供商对不同类型消息的权重/上下文处理方式不同，因此有时手动创建一个新的 [`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage) 对象并将其插入消息历史中（就像它来自模型一样）会很有帮助。

```python
from langchain.messages import AIMessage, SystemMessage, HumanMessage

# 手动创建 AI 消息（例如，用于对话历史）
ai_msg = AIMessage("I'd be happy to help you with that question!")

# 添加到对话历史
messages = [
    SystemMessage("You are a helpful assistant"),
    HumanMessage("Can you help me?"),
    ai_msg,  # 插入就像它来自模型一样
    HumanMessage("Great! What's 2+2?")
]

response = model.invoke(messages)
```

#### 属性

| 属性 | 类型 | 描述 |
|------|------|------|
| `text` | string | 消息的文本内容 |
| `content` | string \| dict[] | 消息的原始内容 |
| `content_blocks` | ContentBlock[] | 消息的标准内容块 |
| `tool_calls` | dict[] \| None | 模型进行的工具调用 |
| `id` | string | 消息的唯一标识符 |
| `usage_metadata` | dict \| None | 消息的使用元数据 |
| `response_metadata` | ResponseMetadata \| None | 消息的响应元数据 |

#### 工具调用

当模型进行工具调用时，它们包含在 [`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage) 中：

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-5-nano")

def get_weather(location: str) -> str:
    """Get the weather at a location."""
    ...

model_with_tools = model.bind_tools([get_weather])
response = model_with_tools.invoke("What's the weather in Paris?")

for tool_call in response.tool_calls:
    print(f"Tool: {tool_call['name']}")
    print(f"Args: {tool_call['args']}")
    print(f"ID: {tool_call['id']}")
```

其他结构化数据，如推理或引用，也可能出现在消息内容中。

#### Token 使用量

[`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage) 可以在其 [`usage_metadata`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.UsageMetadata) 字段中保存 token 计数和其他使用元数据：

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-5-nano")

response = model.invoke("Hello!")
response.usage_metadata
```

```
{'input_tokens': 8,
 'output_tokens': 304,
 'total_tokens': 312,
 'input_token_details': {'audio': 0, 'cache_read': 0},
 'output_token_details': {'audio': 0, 'reasoning': 256}}
```

有关详细信息，请参阅 [`UsageMetadata`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.UsageMetadata)。

#### 流式传输和分块

在流式传输过程中，您将收到可以合并为完整消息对象的 [`AIMessageChunk`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessageChunk) 对象：

```python
chunks = []
full_message = None
for chunk in model.stream("Hi"):
    chunks.append(chunk)
    print(chunk.text)
    full_message = chunk if full_message is None else full_message + chunk
```

了解更多：

* [从聊天模型流式传输 token](/oss/python/langchain/models#stream)
* [从 agent 流式传输 token 和/或步骤](/oss/python/langchain/streaming)

---

### 工具消息

对于支持工具调用的模型，AI 消息可以包含工具调用。工具消息用于将单个工具执行的结果传递回模型。

[工具](/oss/python/langchain/tools) 可以直接生成 [`ToolMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage) 对象。下面我们展示一个简单示例。详细内容请参阅工具指南。

```python
from langchain.messages import AIMessage
from langchain.messages import ToolMessage

# 模型进行工具调用后
#（这里为简洁起见，我们演示手动创建消息）
ai_message = AIMessage(
    content=[],
    tool_calls=[{
        "name": "get_weather",
        "args": {"location": "San Francisco"},
        "id": "call_123"
    }]
)

# 执行工具并创建结果消息
weather_result = "Sunny, 72°F"
tool_message = ToolMessage(
    content=weather_result,
    tool_call_id="call_123"  # 必须与调用 ID 匹配
)

# 继续对话
messages = [
    HumanMessage("What's the weather in San Francisco?"),
    ai_message,  # 模型的工具调用
    tool_message,  # 工具执行结果
]
response = model.invoke(messages)  # 模型处理结果
```

#### 属性

| 属性 | 类型 | 描述 |
|------|------|------|
| `content` | string | 工具调用的字符串化输出 |
| `tool_call_id` | string | 此消息响应的工具调用 ID |
| `name` | string | 被调用的工具名称 |
| `artifact` | dict | 不会发送给模型但可以以编程方式访问的附加数据 |

`artifact` 字段存储不会发送给模型但可以以编程方式访问的补充数据。这对于存储原始结果、调试信息或用于下游处理的数据非常有用，而不会弄乱模型的上下文。

例如，检索工具可以从文档中检索一段文本供模型参考。消息 `content` 包含模型将引用的文本，而 `artifact` 可以包含文档标识符或其他应用程序可以使用的元数据（例如，渲染页面）。

```python
from langchain.messages import ToolMessage

# 发送给模型
message_content = "It was the best of times, it was the worst of times."

# 工件可用于下游
artifact = {"document_id": "doc_123", "page": 0}

tool_message = ToolMessage(
    content=message_content,
    tool_call_id="call_123",
    name="search_books",
    artifact=artifact,
)
```

有关构建带有 LangChain 的检索 agent 的端到端示例，请参阅 RAG 教程。

---

## 消息内容

您可以将消息的内容视为发送给模型的数据有效载荷。消息有一个 `content` 属性，该属性是松散类型的，支持字符串和未类型化对象列表（例如，字典）。这允许直接在 LangChain 聊天模型中支持提供商原生结构，例如多模态内容和其他数据。

另外，LangChain 为文本、推理、引用、多模态数据、服务器端工具调用和其他消息内容提供了专门的内容类型。

LangChain 聊天模型接受 `content` 属性中的消息内容。

这可以包含：

1. 一个字符串
2. 提供商原生格式的内容块列表
3. LangChain 标准内容块列表

下面是一个使用多模态输入的示例：

```python
from langchain.messages import HumanMessage

# 字符串内容
human_message = HumanMessage("Hello, how are you?")

# 提供商原生格式（例如 OpenAI）
human_message = HumanMessage(content=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
])

# 标准内容块列表
human_message = HumanMessage(content_blocks=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image", "url": "https://example.com/image.jpg"},
])
```

在初始化消息时指定 `content_blocks` 仍然会填充消息 `content`，但为此提供了类型安全的接口。

### 标准内容块

LangChain 提供了一种跨提供商工作的消息内容的标准表示。

消息对象实现了一个 `content_blocks` 属性，它将惰性地将 `content` 属性解析为标准的、类型安全的表示。例如，从 [`ChatAnthropic`](/oss/python/integrations/chat/anthropic) 或 [`ChatOpenAI`](/oss/python/integrations/chat/openai) 生成的消息将在各自提供商的格式中包含 `thinking` 或 `reasoning` 块，但可以惰性地解析为一致的 [`ReasoningContentBlock`](#content-block-reference) 表示：

**Anthropic：**

```python
from langchain.messages import AIMessage

message = AIMessage(
    content=[
        {"type": "thinking", "thinking": "...", "signature": "WaUjzkyp..."},
        {"type": "text", "text": "..."},
    ],
    response_metadata={"model_provider": "anthropic"}
)
message.content_blocks
```

```
[{'type': 'reasoning',
  'reasoning': '...',
  'extras': {'signature': 'WaUjzkyp...'}},
 {'type': 'text', 'text': '...'}]
```

**OpenAI：**

```python
from langchain.messages import AIMessage

message = AIMessage(
    content=[
        {
            "type": "reasoning",
            "id": "rs_abc123",
            "summary": [
                {"type": "summary_text", "text": "summary 1"},
                {"type": "summary_text", "text": "summary 2"},
            ],
        },
        {"type": "text", "text": "...", "id": "msg_abc123"},
    ],
    response_metadata={"model_provider": "openai"}
)
message.content_blocks
```

```
[{'type': 'reasoning', 'id': 'rs_abc123', 'reasoning': 'summary 1'},
 {'type': 'reasoning', 'id': 'rs_abc123', 'reasoning': 'summary 2'},
 {'type': 'text', 'text': '...', 'id': 'msg_abc123'}]
```

有关开始使用您选择推理提供商的更多信息，请参阅集成指南。

**序列化标准内容**

如果 LangChain 之外的应用程序需要访问标准内容块表示，您可以选择将内容块存储在消息内容中。

为此，您可以将 `LC_OUTPUT_VERSION` 环境变量设置为 `v1`。或者，使用 `output_version="v1"` 初始化任何聊天模型：

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-5-nano", output_version="v1")
```

### 多模态

**多模态** 是指处理不同形式数据的能力，如文本、音频、图像和视频。LangChain 包含这些数据的标准类型，可以跨提供商使用。

[聊天模型](/oss/python/langchain/models) 可以接受多模态数据作为输入并生成多模态输出。下面我们展示包含多模态数据的输入消息的简短示例。

额外键可以包含在内容块的顶层或嵌套在 `"extras": {"key": value}` 中。

例如，[OpenAI](/oss/python/integrations/chat/openai#pdfs) 和 [AWS BedRock Converse](/oss/python/integrations/chat/bedrock) 需要 PDF 的文件名。请参阅您所选模型的提供商页面了解具体细节。

**图像输入：**

```python
# 来自 URL
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this image."},
        {"type": "image", "url": "https://example.com/path/to/image.jpg"},
    ]
}

# 来自 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this image."},
        {
            "type": "image",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "image/jpeg",
        },
    ]
}

# 来自提供商管理的文件 ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this image."},
        {"type": "image", "file_id": "file-abc123"},
    ]
}
```

**PDF 文档输入：**

```python
# 来自 URL
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this document."},
        {"type": "file", "url": "https://example.com/path/to/document.pdf"},
    ]
}

# 来自 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this document."},
        {
            "type": "file",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "application/pdf",
        },
    ]
}

# 来自提供商管理的文件 ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this document."},
        {"type": "file", "file_id": "file-abc123"},
    ]
}
```

**音频输入：**

```python
# 来自 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this audio."},
        {
            "type": "audio",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "audio/wav",
        },
    ]
}

# 来自提供商管理的文件 ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this audio."},
        {"type": "audio", "file_id": "file-abc123"},
    ]
}
```

**视频输入：**

```python
# 来自 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this video."},
        {
            "type": "video",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "video/mp4",
        },
    ]
}

# 来自提供商管理的文件 ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this video."},
        {"type": "video", "file_id": "file-abc123"},
    ]
}
```

并非所有模型都支持所有文件类型。请查看模型提供商的参考文档了解支持的格式和大小限制。

### 内容块参考

内容块（在创建消息或访问 `content_blocks` 属性时）表示为类型化字典列表。列表中的每个项目必须符合以下块类型之一：

**核心内容块**

**TextContentBlock**

用途：标准文本输出

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"text"` |
| `text` | string | 文本内容 |
| `annotations` | object[] | 文本的注释列表 |
| `extras` | object | 额外的提供商特定数据 |

示例：
```python
{
    "type": "text",
    "text": "Hello world",
    "annotations": []
}
```

**ReasoningContentBlock**

用途：模型推理步骤

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"reasoning"` |
| `reasoning` | string | 推理内容 |
| `extras` | object | 额外的提供商特定数据 |

示例：
```python
{
    "type": "reasoning",
    "reasoning": "The user is asking about...",
    "extras": {"signature": "abc123"},
}
```

**多模态内容块**

**ImageContentBlock**

用途：图像数据

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"image"` |
| `url` | string | 指向图像位置的 URL |
| `base64` | string | Base64 编码的图像数据 |
| `id` | string | 此内容块的唯一标识符 |
| `mime_type` | string | 图像 MIME 类型 |

**AudioContentBlock**

用途：音频数据

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"audio"` |
| `url` | string | 指向音频位置的 URL |
| `base64` | string | Base64 编码的音频数据 |
| `id` | string | 此内容块的唯一标识符 |
| `mime_type` | string | 音频 MIME 类型 |

**VideoContentBlock**

用途：视频数据

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"video"` |
| `url` | string | 指向视频位置的 URL |
| `base64` | string | Base64 编码的视频数据 |
| `id` | string | 此内容块的唯一标识符 |
| `mime_type` | string | 视频 MIME 类型 |

**FileContentBlock**

用途：通用文件（PDF 等）

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"file"` |
| `url` | string | 指向文件位置的 URL |
| `base64` | string | Base64 编码的文件数据 |
| `id` | string | 此内容块的唯一标识符 |
| `mime_type` | string | 文件 MIME 类型 |

**PlainTextContentBlock**

用途：文档文本（`.txt`、`.md`）

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"text-plain"` |
| `text` | string | 文本内容 |
| `mime_type` | string | 文本的 MIME 类型 |

**工具调用内容块**

**ToolCall**

用途：函数调用

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"tool_call"` |
| `name` | string | 要调用的工具名称 |
| `args` | object | 传递给工具的参数 |
| `id` | string | 此工具调用的唯一标识符 |

示例：
```python
{
    "type": "tool_call",
    "name": "search",
    "args": {"query": "weather"},
    "id": "call_123"
}
```

**ToolCallChunk**

用途：流式工具调用片段

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"tool_call_chunk"` |
| `name` | string | 被调用的工具名称 |
| `args` | string | 部分工具参数（可能是 JSON 片段） |
| `id` | string | 工具调用标识符 |
| `index` | number \| string | 此分块在流中的位置 |

**InvalidToolCall**

用途：格式错误的调用，用于捕获 JSON 解析错误

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"invalid_tool_call"` |
| `name` | string | 未能调用的工具名称 |
| `args` | object | 传递给工具的参数 |
| `error` | string | 错误的描述 |

**服务器端工具执行**

**ServerToolCall**

用途：服务器端执行的工具调用

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"server_tool_call"` |
| `id` | string | 与工具调用关联的标识符 |
| `name` | string | 要调用的工具名称 |
| `args` | string | 部分工具参数 |

**ServerToolCallChunk**

用途：流式服务器端工具调用片段

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"server_tool_call_chunk"` |
| `id` | string | 与工具调用关联的标识符 |
| `name` | string | 被调用的工具名称 |
| `args` | string | 部分工具参数 |
| `index` | number \| string | 此分块在流中的位置 |

**ServerToolResult**

用途：搜索结果

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"server_tool_result"` |
| `tool_call_id` | string | 对应服务器工具调用的标识符 |
| `id` | string | 与服务器工具结果关联的标识符 |
| `status` | string | 服务器端工具的执行状态 |
| `output` | object | 已执行工具的输出 |

**提供商特定块**

**NonStandardContentBlock**

用途：提供商特定的转义机制

| 属性 | 类型 | 描述 |
|------|------|------|
| `type` | string | 始终为 `"non_standard"` |
| `value` | object | 提供商特定的数据结构 |

有关其他提供商特定的内容类型，请参阅每个模型提供商的参考文档。

在 [API 参考](https://reference.langchain.com/python/langchain/messages) 中查看规范类型定义。

内容块是作为 LangChain v1 中消息的新属性引入的，用于标准化跨提供商的内容格式，同时保持与现有代码的向后兼容性。

内容块不是 [`content`](https://reference.langchain.com/python/langchain_core/language_models/#langchain_core.messages.BaseMessage.content) 属性的替代品，而是一个新属性，可用于以标准化格式访问消息的内容。

## 与聊天模型一起使用

[聊天模型](/oss/python/langchain/models) 接受一系列消息对象作为输入，并返回 [`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage) 作为输出。交互通常是无状态的，因此简单的对话循环涉及使用不断增长的消息列表调用模型。

请参阅以下指南了解更多信息：

* 用于保留和管理对话历史的内置功能
* 管理上下文窗口的策略，包括修剪和总结消息

