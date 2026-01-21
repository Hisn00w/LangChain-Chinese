# 结构化输出

结构化输出允许 agent 以特定的、可预测的格式返回数据。您无需解析自然语言响应，而是获得 JSON 对象、[Pydantic 模型](https://docs.pydantic.dev/latest/concepts/models/#basic-model-usage) 或数据类形式的结构化数据，您的应用程序可以直接使用这些数据。

LangChain 的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 自动处理结构化输出。用户设置他们期望的结构化输出模式，当模型生成结构化数据时，它会被捕获、验证并在 agent 状态的 `'structured_response'` 键中返回。

```python
def create_agent(
    ...
    response_format: Union[
        ToolStrategy[StructuredResponseT],
        ProviderStrategy[StructuredResponseT],
        type[StructuredResponseT],
        None,
    ]
```

## 响应格式

使用 `response_format` 控制 agent 如何返回结构化数据：

* **`ToolStrategy[StructuredResponseT]`**：使用工具调用进行结构化输出
* **`ProviderStrategy[StructuredResponseT]`**：使用提供商原生结构化输出
* **`type[StructuredResponseT]`**：模式类型 - 根据模型能力自动选择最佳策略
* **`None`**：未明确请求结构化输出

当直接提供模式类型时，LangChain 会自动选择：

* 如果所选的模型和提供商支持原生结构化输出（例如 [OpenAI](/oss/python/integrations/providers/openai)、[Anthropic (Claude)](/oss/python/integrations/providers/anthropic) 或 [xAI (Grok)](/oss/python/integrations/providers/xAI)），则使用 `ProviderStrategy`
* 对于所有其他模型，使用 `ToolStrategy`

如果使用 `langchain>=1.1`，原生结构化输出功能的支持会从模型的[配置文件数据](/oss/python/langchain/models#model-profiles)中动态读取。如果数据不可用，请使用其他条件或手动指定：

```python
custom_profile = {
    "structured_output": True,
    # ...
}
model = init_chat_model("...", profile=custom_profile)
```

如果指定了工具，模型必须支持同时使用工具和结构化输出。

结构化响应在 agent 最终状态的 `structured_response` 键中返回。

## 提供商策略

一些模型提供商通过其 API 原生支持结构化输出（例如 OpenAI、xAI (Grok)、Gemini、Anthropic (Claude)）。当可用时，这是最可靠的方法。

要使用此策略，请配置 `ProviderStrategy`：

```python
class ProviderStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    strict: bool | None = None
```

`strict` 参数需要 `langchain>=1.2`。

**参数：**

| 参数 | 类型 | 描述 |
|------|------|------|
| `schema` | required | 定义结构化输出模式的模式。支持：<br>- **Pydantic 模型**：带字段验证的 `BaseModel` 子类。返回经过验证的 Pydantic 实例。<br>- **数据类**：带类型注释的 Python 数据类。返回字典。<br>- **TypedDict**：类型化字典类。返回字典。<br>- **JSON Schema**：带 JSON 模式规范的字典。返回字典。 |
| `strict` | optional | 启用严格模式遵循的可选布尔参数。由某些提供商支持（例如 [OpenAI](/oss/python/integrations/chat/openai) 和 [xAI](/oss/python/integrations/chat/xai)）。默认为 `None`（禁用）。 |

当您将模式类型直接传递给 [`create_agent.response_format`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent\(response_format\)) 并且模型支持原生结构化输出时，LangChain 会自动使用 `ProviderStrategy`：

**Pydantic 模型：**

```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent


class ContactInfo(BaseModel):
    """Contact information for a person."""
    name: str = Field(description="The name of the person")
    email: str = Field(description="The email address of the person")
    phone: str = Field(description="The phone number of the person")

agent = create_agent(
    model="gpt-5",
    response_format=ContactInfo  # Auto-selects ProviderStrategy
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

print(result["structured_response"])
# ContactInfo(name='John Doe', email='john@example.com', phone='(555) 123-4567')
```

**数据类：**

```python
from dataclasses import dataclass
from langchain.agents import create_agent


@dataclass
class ContactInfo:
    """Contact information for a person."""
    name: str # The name of the person
    email: str # The email address of the person
    phone: str # The phone number of the person

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ContactInfo  # Auto-selects ProviderStrategy
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
```

**TypedDict：**

```python
from typing_extensions import TypedDict
from langchain.agents import create_agent


class ContactInfo(TypedDict):
    """Contact information for a person."""
    name: str # The name of the person
    email: str # The email address of the person
    phone: str # The phone number of the person

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ContactInfo  # Auto-selects ProviderStrategy
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
```

**JSON Schema：**

```python
from langchain.agents import create_agent


contact_info_schema = {
    "type": "object",
    "description": "Contact information for a person.",
    "properties": {
        "name": {"type": "string", "description": "The name of the person"},
        "email": {"type": "string", "description": "The email address of the person"},
        "phone": {"type": "string", "description": "The phone number of the person"}
    },
    "required": ["name", "email", "phone"]
}

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ProviderStrategy(contact_info_schema)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
```

提供商原生结构化输出提供了高可靠性和严格验证，因为模型提供商会强制执行模式。请在可用时使用它。

如果提供商原生支持您所选模型的结构化输出，那么编写 `response_format=ProductReview` 在功能上等同于编写 `response_format=ProviderStrategy(ProductReview)`。

无论哪种情况，如果不支持结构化输出，agent 将回退到工具调用策略。

## 工具调用策略

对于不支持原生结构化输出的模型，LangChain 使用工具调用来实现相同的结果。这适用于所有支持工具调用的模型（大多数现代模型）。

要使用此策略，请配置 `ToolStrategy`：

```python
class ToolStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    tool_message_content: str | None
    handle_errors: Union[
        bool,
        str,
        type[Exception],
        tuple[type[Exception], ...],
        Callable[[Exception], str],
    ]
```

**参数：**

| 参数 | 类型 | 描述 |
|------|------|------|
| `schema` | required | 定义结构化输出模式的模式。支持：<br>- **Pydantic 模型**：带字段验证的 `BaseModel` 子类。返回经过验证的 Pydantic 实例。<br>- **数据类**：带类型注释的 Python 数据类。返回字典。<br>- **TypedDict**：类型化字典类。返回字典。<br>- **JSON Schema**：带 JSON 模式规范的字典。返回字典。<br>- **联合类型**：多个模式选项。模型将根据上下文选择最合适的模式。 |
| `tool_message_content` | optional | 生成结构化输出时返回的工具消息的自定义内容。如果未提供，默认为显示结构化响应数据的消息。 |
| `handle_errors` | optional | 结构化输出验证失败的错误处理策略。默认为 `True`。<br>- **`True`**：使用默认错误模板捕获所有错误<br>- **`str`**：使用此自定义消息捕获所有错误<br>- **`type[Exception]`**：仅使用默认消息捕获此异常类型<br>- **`tuple[type[Exception], ...]`**：仅使用默认消息捕获这些异常类型<br>- **`Callable[[Exception], str]`**：返回错误消息的自定义函数<br>- **`False`**：不重试，让异常传播 |

**Pydantic 模型：**

```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductReview(BaseModel):
    """Analysis of a product review."""
    rating: int | None = Field(description="The rating of the product", ge=1, le=5)
    sentiment: Literal["positive", "negative"] = Field(description="The sentiment of the review")
    key_points: list[str] = Field(description="The key points of the review. Lowercase, 1-3 words each.")

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ToolStrategy(ProductReview)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# ProductReview(rating=5, sentiment='positive', key_points=['fast shipping', 'expensive'])
```

**数据类：**

```python
from dataclasses import dataclass
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


@dataclass
class ProductReview:
    """Analysis of a product review."""
    rating: int | None  # The rating of the product (1-5)
    sentiment: Literal["positive", "negative"]  # The sentiment of the review
    key_points: list[str]  # The key points of the review

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ToolStrategy(ProductReview)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
```

**TypedDict：**

```python
from typing import Literal
from typing_extensions import TypedDict
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductReview(TypedDict):
    """Analysis of a product review."""
    rating: int | None  # The rating of the product (1-5)
    sentiment: Literal["positive", "negative"]  # The sentiment of the review
    key_points: list[str]  # The key points of the review

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ToolStrategy(ProductReview)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
```

**JSON Schema：**

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


product_review_schema = {
    "type": "object",
    "description": "Analysis of a product review.",
    "properties": {
        "rating": {
            "type": ["integer", "null"],
            "description": "The rating of the product (1-5)",
            "minimum": 1,
            "maximum": 5
        },
        "sentiment": {
            "type": "string",
            "enum": ["positive", "negative"],
            "description": "The sentiment of the review"
        },
        "key_points": {
            "type": "array",
            "items": {"type": "string"},
            "description": "The key points of the review"
        }
    },
    "required": ["sentiment", "key_points"]
}

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ToolStrategy(product_review_schema)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
```

**联合类型：**

```python
from pydantic import BaseModel, Field
from typing import Literal, Union
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductReview(BaseModel):
    """Analysis of a product review."""
    rating: int | None = Field(description="The rating of the product", ge=1, le=5)
    sentiment: Literal["positive", "negative"] = Field(description="The sentiment of the review")
    key_points: list[str] = Field(description="The key points of the review. Lowercase, 1-3 words each.")

class CustomerComplaint(BaseModel):
    """A customer complaint about a product or service."""
    issue_type: Literal["product", "service", "shipping", "billing"] = Field(description="The type of issue")
    severity: Literal["low", "medium", "high"] = Field(description="The severity of the complaint")
    description: str = Field(description="Brief description of the complaint")

agent = create_agent(
    model="gpt-5",
    tools=tools,
    response_format=ToolStrategy(Union[ProductReview, CustomerComplaint])
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# ProductReview(rating=5, sentiment='positive', key_points=['fast shipping', 'expensive'])
```

### 自定义工具消息内容

`tool_message_content` 参数允许您自定义生成结构化输出时出现在对话历史中的消息：

```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class MeetingAction(BaseModel):
    """Action items extracted from a meeting transcript."""
    task: str = Field(description="The specific task to be completed")
    assignee: str = Field(description="Person responsible for the task")
    priority: Literal["low", "medium", "high"] = Field(description="Priority level")

agent = create_agent(
    model="gpt-5",
    tools=[],
    response_format=ToolStrategy(
        schema=MeetingAction,
        tool_message_content="Action item captured and added to meeting notes!"
    )
)

agent.invoke({
    "messages": [{"role": "user", "content": "From our meeting: Sarah needs to update the project timeline as soon as possible"}]
})
```

```
================================ Human Message =================================

From our meeting: Sarah needs to update the project timeline as soon as possible
================================== Ai Message ==================================
Tool Calls:
  MeetingAction (call_1)
 Call ID: call_1
  Args:
    task: Update the project timeline
    assignee: Sarah
    priority: high
================================= Tool Message =================================
Name: MeetingAction

Action item captured and added to meeting notes!
```

如果没有 `tool_message_content`，我们的最终 [`ToolMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage) 将是：

```
================================= Tool Message =================================
Name: MeetingAction

Returning structured response: {'task': 'update the project timeline', 'assignee': 'Sarah', 'priority': 'high'}
```

### 错误处理

模型在通过工具调用生成结构化输出时可能会出错。LangChain 提供智能重试机制来自动处理这些错误。

#### 多个结构化输出错误

当模型错误地调用多个结构化输出工具时，agent 会在 [`ToolMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage) 中提供错误反馈，并提示模型重试：

```python
from pydantic import BaseModel, Field
from typing import Union
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ContactInfo(BaseModel):
    name: str = Field(description="Person's name")
    email: str = Field(description="Email address")

class EventDetails(BaseModel):
    event_name: str = Field(description="Name of the event")
    date: str = Field(description="Event date")

agent = create_agent(
    model="gpt-5",
    tools=[],
    response_format=ToolStrategy(Union[ContactInfo, EventDetails])  # Default: handle_errors=True
)

agent.invoke({
    "messages": [{"role": "user", "content": "Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th"}]
})
```

```
================================ Human Message =================================

Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th
None
================================== Ai Message ==================================
Tool Calls:
  ContactInfo (call_1)
 Call ID: call_1
  Args:
    name: John Doe
    email: john@email.com
  EventDetails (call_2)
 Call ID: call_2
  Args:
    event_name: Tech Conference
    date: March 15th
================================= Tool Message =================================
Name: ContactInfo

Error: Model incorrectly returned multiple structured responses (ContactInfo, EventDetails) when only one is expected.
 Please fix your mistakes.
================================= Tool Message =================================
Name: EventDetails

Error: Model incorrectly returned multiple structured responses (ContactInfo, EventDetails) when only one is expected.
 Please fix your mistakes.
================================== Ai Message ==================================
Tool Calls:
  ContactInfo (call_3)
 Call ID: call_3
  Args:
    name: John Doe
    email: john@email.com
================================= Tool Message =================================
Name: ContactInfo

Returning structured response: {'name': 'John Doe', 'email': 'john@email.com'}
```

#### 模式验证错误

当结构化输出与预期模式不匹配时，agent 会提供具体的错误反馈：

```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductRating(BaseModel):
    rating: int | None = Field(description="Rating from 1-5", ge=1, le=5)
    comment: str = Field(description="Review comment")

agent = create_agent(
    model="gpt-5",
    tools=[],
    response_format=ToolStrategy(ProductRating),  # Default: handle_errors=True
    system_prompt="You are a helpful assistant that parses product reviews. Do not make any field or value up."
)

agent.invoke({
    "messages": [{"role": "user", "content": "Parse this: Amazing product, 10/10!"}]
})
```

```
================================ Human Message =================================

Parse this: Amazing product, 10/10!
================================== Ai Message ==================================
Tool Calls:
  ProductRating (call_1)
 Call ID: call_1
  Args:
    rating: 10
    comment: Amazing product
================================= Tool Message =================================
Name: ProductRating

Error: Failed to parse structured output for tool 'ProductRating': 1 validation error for ProductRating.rating
  Input should be less than or equal to 5 [type=less_than_equal, input_value=10, input_type=int].
 Please fix your mistakes.
================================== Ai Message ==================================
Tool Calls:
  ProductRating (call_2)
 Call ID: call_2
  Args:
    rating: 5
    comment: Amazing product
================================= Tool Message =================================
Name: ProductRating

Returning structured response: {'rating': 5, 'comment': 'Amazing product'}
```

#### 错误处理策略

您可以使用 `handle_errors` 参数自定义错误的处理方式：

**自定义错误消息：**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors="Please provide a valid rating between 1-5 and include a comment."
)
```

如果 `handle_errors` 是一个字符串，agent 将始终提示模型使用固定工具消息重试：

```
================================= Tool Message =================================
Name: ProductRating

Please provide a valid rating between 1-5 and include a comment.
```

**仅处理特定异常：**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors=ValueError  # Only retry on ValueError, raise others
)
```

如果 `handle_errors` 是一个异常类型，agent 仅在引发的异常是指定类型时才重试（使用默认错误消息）。在所有其他情况下，异常将被引发。

**处理多个异常类型：**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors=(ValueError, TypeError)  # Retry on ValueError and TypeError
)
```

如果 `handle_errors` 是一个异常元组，agent 仅在引发的异常是指定类型之一时才重试（使用默认错误消息）。在所有其他情况下，异常将被引发。

**自定义错误处理函数：**

```python
from langchain.agents.structured_output import StructuredOutputValidationError
from langchain.agents.structured_output import MultipleStructuredOutputsError

def custom_error_handler(error: Exception) -> str:
    if isinstance(error, StructuredOutputValidationError):
        return "There was an issue with the format. Try again."
    elif isinstance(error, MultipleStructuredOutputsError):
        return "Multiple structured outputs were returned. Pick the most relevant one."
    else:
        return f"Error: {str(error)}"


agent = create_agent(
    model="gpt-5",
    tools=[],
    response_format=ToolStrategy(
                        schema=Union[ContactInfo, EventDetails],
                        handle_errors=custom_error_handler
                    )  # Default: handle_errors=True
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th"}]
})

for msg in result['messages']:
    # 如果消息实际上是 ToolMessage 对象（不是字典），检查它的类名
    if type(msg).__name__ == "ToolMessage":
        print(msg.content)
    # 如果消息是字典或您想要回退
    elif isinstance(msg, dict) and msg.get('tool_call_id'):
        print(msg['content'])
```

当发生 `StructuredOutputValidationError` 时：

```
================================= Tool Message =================================
Name: ToolStrategy

There was an issue with the format. Try again.
```

当发生 `MultipleStructuredOutputsError` 时：

```
================================= Tool Message =================================
Name: ToolStrategy

Multiple structured outputs were returned. Pick the most relevant one.
```

当发生其他错误时：

```
================================= Tool Message =================================
Name: ToolStrategy

Error: <error message>
```

**不进行错误处理：**

```python
response_format = ToolStrategy(
    schema=ProductRating,
    handle_errors=False  # All errors raised
)
```

