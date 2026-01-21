# 内置中间件

> 适用于常见智能体用例的预构建中间件

LangChain 提供了适用于常见用例的预构建中间件。每个中间件都已准备好用于生产，并可根据您的特定需求进行配置。

## 供应商无关中间件

以下中间件适用于任何 LLM 提供商：

| 中间件 | 描述 |
|--------|------|
| [摘要](#摘要) | 在接近 token 限制时自动摘要对话历史。 |
| [人工介入](#人工介入) | 暂停执行以等待人工批准工具调用。 |
| [模型调用限制](#模型调用限制) | 限制模型调用次数以防止过度成本。 |
| [工具调用限制](#工具调用限制) | 通过限制调用次数来控制工具执行。 |
| [模型回退](#模型回退) | 当主模型失败时自动回退到备用模型。 |
| [PII 检测](#pii-检测) | 检测和处理个人身份信息。 |
| [待办事项列表](#待办事项列表) | 为智能体配备任务规划和跟踪功能。 |
| [LLM 工具选择器](#llm-工具选择器) | 在调用主模型之前使用 LLM 选择相关工具。 |
| [工具重试](#工具重试) | 使用指数退避自动重试失败的工具调用。 |
| [模型重试](#模型重试) | 使用指数退避自动重试失败的模型调用。 |
| [LLM 工具模拟器](#llm-工具模拟器) | 使用 LLM 模拟工具执行以进行测试。 |
| [上下文编辑](#上下文编辑) | 通过修剪或清除工具使用来管理对话上下文。 |
| [Shell 工具](#shell-工具) | 向智能体暴露持久化 shell 会话以执行命令。 |
| [文件搜索](#文件搜索) | 提供对文件系统的 Glob 和 Grep 搜索工具。 |

### 摘要

自动摘要对话历史，当接近 token 限制时，保留最近的消息同时压缩较旧的上下文。摘要适用于以下场景：

* 超过上下文窗口的长对话。
* 具有大量历史的多轮对话。
* 需要保留完整对话上下文的应用程序。

**API 参考：** [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.SummarizationMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4o-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20),
        ),
    ],
)
```

### 人工介入

暂停智能体执行以等待人工批准、编辑或拒绝工具调用，然后才执行它们。人工介入适用于以下场景：

* 需要人工批准的高风险操作（例如数据库写入、金融交易）。
* 必须有人工监督的合规工作流程。
* 人工反馈指导智能体的长对话。

**API 参考：** [`HumanInTheLoopMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.HumanInTheLoopMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-4o",
    tools=[your_read_email_tool, your_send_email_tool],
    checkpointer=InMemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "your_send_email_tool": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                },
                "your_read_email_tool": False,
            }
        ),
    ],
)
```

### 模型调用限制

限制模型调用次数以防止无限循环或过度成本。模型调用限制适用于以下场景：

* 防止失控智能体进行过多 API 调用。
* 在生产部署中强制执行成本控制。
* 在特定调用预算内测试智能体行为。

**API 参考：** [`ModelCallLimitMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ModelCallLimitMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-4o",
    checkpointer=InMemorySaver(),  # 线程限制需要此参数
    tools=[],
    middleware=[
        ModelCallLimitMiddleware(
            thread_limit=10,
            run_limit=5,
            exit_behavior="end",
        ),
    ],
)
```

### 工具调用限制

通过限制工具调用次数来控制智能体执行，可以是全局限制所有工具或针对特定工具。工具调用限制适用于以下场景：

* 防止对昂贵的外部 API 进行过多调用。
* 限制网络搜索或数据库查询。
* 对特定工具使用强制执行速率限制。
* 防止智能体循环失控。

**API 参考：** [`ToolCallLimitMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ToolCallLimitMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[search_tool, database_tool],
    middleware=[
        # 全局限制
        ToolCallLimitMiddleware(thread_limit=20, run_limit=10),
        # 特定工具限制
        ToolCallLimitMiddleware(
            tool_name="search",
            thread_limit=5,
            run_limit=3,
        ),
    ],
)
```

### 模型回退

当主模型失败时自动回退到备用模型。模型回退适用于以下场景：

* 构建能够处理模型故障的弹性智能体。
* 通过回退到更便宜的模型来优化成本。
* 跨 OpenAI、Anthropic 等提供商实现冗余。

**API 参考：** [`ModelFallbackMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ModelFallbackMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelFallbackMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        ModelFallbackMiddleware(
            "gpt-4o-mini",
            "claude-3-5-sonnet-20241022",
        ),
    ],
)
```

### PII 检测

使用可配置策略检测和处理对话中的个人身份信息。PII 检测适用于以下场景：

* 具有合规要求的医疗保健和金融应用程序。
* 需要清理日志的客户服务智能体。
* 处理敏感用户数据的任何应用程序。

**API 参考：** [`PIIMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.PIIMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
    ],
)
```

#### 自定义 PII 类型

您可以通过提供 `detector` 参数来创建自定义 PII 类型。这允许您检测超出内置类型的特定用例模式。

**创建自定义检测器的三种方式：**

1. **正则表达式字符串** - 简单模式匹配
2. **自定义函数** - 带验证的复杂检测逻辑

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
import re

# 方法 1：正则表达式字符串
agent1 = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32}",
            strategy="block",
        ),
    ],
)

# 方法 2：编译的正则表达式模式
agent2 = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        PIIMiddleware(
            "phone_number",
            detector=re.compile(r"\+?\d{1,3}[\s.-]?\d{3,4}[\s.-]?\d{4}"),
            strategy="mask",
        ),
    ],
)

# 方法 3：自定义检测器函数
def detect_ssn(content: str) -> list[dict[str, str | int]]:
    """检测社会安全号码并进行验证。"""
    import re
    matches = []
    pattern = r"\d{3}-\d{2}-\d{4}"
    for match in re.finditer(pattern, content):
        ssn = match.group(0)
        # 验证：前三位不应为 000、666 或 900-999
        first_three = int(ssn[:3])
        if first_three not in [0, 666] and not (900 <= first_three <= 999):
            matches.append({
                "text": ssn,
                "start": match.start(),
                "end": match.end(),
            })
    return matches

agent3 = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        PIIMiddleware(
            "ssn",
            detector=detect_ssn,
            strategy="hash",
        ),
    ],
)
```

### 待办事项列表

为智能体配备任务规划和跟踪功能，适用于复杂的多步骤任务。待办事项列表适用于以下场景：

* 需要跨多个工具协调的复杂多步骤任务。
* 进度可见性很重要的长时间运行操作。

**API 参考：** [`TodoListMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.TodoListMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[read_file, write_file, run_tests],
    middleware=[TodoListMiddleware()],
)
```

### LLM 工具选择器

在调用主模型之前使用 LLM 智能选择相关工具。LLM 工具选择器适用于以下场景：

* 具有许多工具（10+）的智能体，其中大多数与每个查询无关。
* 通过过滤无关工具来减少 token 使用。
* 提高模型专注度和准确性。

**API 参考：** [`LLMToolSelectorMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.LLMToolSelectorMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolSelectorMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[tool1, tool2, tool3, tool4, tool5, ...],
    middleware=[
        LLMToolSelectorMiddleware(
            model="gpt-4o-mini",
            max_tools=3,
            always_include=["search"],
        ),
    ],
)
```

### 工具重试

使用可配置的指数退避自动重试失败的工具调用。工具重试适用于以下场景：

* 处理外部 API 调用中的瞬时故障。
* 提高依赖网络的工具的可靠性。
* 构建能够优雅处理临时错误的弹性智能体。

**API 参考：** [`ToolRetryMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ToolRetryMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[search_tool, database_tool],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```

### 模型重试

使用可配置的指数退避自动重试失败的模型调用。模型重试适用于以下场景：

* 处理模型 API 调用中的瞬时故障。
* 提高依赖网络的模型请求的可靠性。
* 构建能够优雅处理临时模型错误的弹性智能体。

**API 参考：** [`ModelRetryMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ModelRetryMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelRetryMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[search_tool, database_tool],
    middleware=[
        ModelRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```

### LLM 工具模拟器

使用 LLM 模拟工具执行以进行测试，替换实际的工具调用为 AI 生成的响应。LLM 工具模拟器适用于以下场景：

* 在不执行真实工具的情况下测试智能体行为。
* 当外部工具不可用或昂贵时开发智能体。
* 在实现实际工具之前原型设计智能体工作流。

**API 参考：** [`LLMToolEmulator`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.LLMToolEmulator)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolEmulator

agent = create_agent(
    model="gpt-4o",
    tools=[get_weather, search_database, send_email],
    middleware=[
        LLMToolEmulator(),  # 模拟所有工具
    ],
)
```

### 上下文编辑

当达到 token 限制时清除较旧的工具调用输出，同时保留最近的结果。这有助于在具有许多工具调用的长对话中保持上下文窗口的可管理性。上下文编辑适用于以下场景：

* 具有超过 token 限制的许多工具调用的长对话。
* 通过删除不再相关的旧工具输出来减少 token 成本。
* 仅在上下文中保留最近的 N 个工具结果。

**API 参考：** [`ContextEditingMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ContextEditingMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

agent = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        ContextEditingMiddleware(
            edits=[
                ClearToolUsesEdit(
                    trigger=100000,
                    keep=3,
                ),
            ],
        ),
    ],
)
```

### Shell 工具

向智能体暴露持久化 shell 会话以执行命令。Shell 工具中间件适用于以下场景：

* 需要执行系统命令的智能体。
* 开发和部署自动化任务。
* 测试和验证工作流程。
* 文件系统操作和脚本执行。

**API 参考：** [`ShellToolMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ShellToolMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ShellToolMiddleware,
    HostExecutionPolicy,
)

agent = create_agent(
    model="gpt-4o",
    tools=[search_tool],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            execution_policy=HostExecutionPolicy(),
        ),
    ],
)
```

### 文件搜索

提供对文件系统的 Glob 和 Grep 搜索工具。文件搜索中间件适用于以下场景：

* 代码探索和分析。
* 按名称模式查找文件。
* 使用正则表达式搜索代码内容。
* 需要文件发现的大型代码库。

**API 参考：** [`FilesystemFileSearchMiddleware`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.FilesystemFileSearchMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import FilesystemFileSearchMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[],
    middleware=[
        FilesystemFileSearchMiddleware(
            root_path="/workspace",
            use_ripgrep=True,
        ),
    ],
)
```

## 供应商特定中间件

以下中间件针对特定的 LLM 进行了优化：

- **Anthropic** - Claude 模型的提示缓存、bash 工具、文本编辑器、内存和文件搜索中间件
- **OpenAI** - OpenAI 模型的内容审核中间件


