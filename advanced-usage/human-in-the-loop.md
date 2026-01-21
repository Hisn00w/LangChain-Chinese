# 人工介入

人工介入（Human-in-the-Loop，HITL）[中间件(https://docs.langchain.com/oss/python/langchain/middleware/built-in#human-in-the-loop) 允许您向 agent 工具调用添加人工监督。当模型提出可能需要审查的操作时——例如，写入文件或执行 SQL——中间件可以暂停执行并等待决策。

它通过根据可配置策略检查每个工具调用来实现这一点。如果需要干预，中间件会发出一个[中断](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)来停止执行。图状态使用 LangGraph 的[持久化层(https://docs.langchain.com/oss/python/langgraph/persistence)保存，因此执行可以安全暂停并在稍后恢复。

然后由人工决策决定接下来发生什么：操作可以按原样批准（`approve`）、修改后运行（`edit`），或拒绝并提供反馈（`reject`）。

## 中断决策类型

[中间件(https://docs.langchain.com/oss/python/langchain/middleware/built-in#human-in-the-loop) 定义了三种内置的人工响应中断的方式：

| 决策类型 | 描述 | 示例用例 |
|----------|------|----------|
| `approve` | 操作按原样批准，无需更改即可执行 | 完全按书面内容发送电子邮件草稿 |
| `edit` | 工具调用经过修改后执行 | 在发送电子邮件之前更改收件人 |
| `reject` | 工具调用被拒绝，并在对话中添加说明 | 拒绝电子邮件草稿并解释如何重写 |

每个工具可用的决策类型取决于您在 `interrupt_on` 中配置的策略。当多个工具调用同时暂停时，每个操作都需要单独的决策。决策必须按照中断请求中操作出现的相同顺序提供。

**编辑**工具参数时，请保守地进行更改。对原始参数的重大修改可能导致模型重新评估其方法，并可能多次执行工具或采取意外操作。

## 配置中断

要使用 HITL，在创建 agent 时将[中间件(https://docs.langchain.com/oss/python/langchain/middleware/built-in#human-in-the-loop)添加到 agent 的 `middleware` 列表中。

您可以使用工具操作到每个操作允许的决策类型的映射来配置它。当工具调用与映射中的操作匹配时，中间件将中断执行。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver


agent = create_agent(
    model="gpt-4o",
    tools=[write_file_tool, execute_sql_tool, read_data_tool],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "write_file": True,  # 所有决策（approve、edit、reject）都允许
                "execute_sql": {"allowed_decisions": ["approve", "reject"]},  # 不允许编辑
                # 安全操作，无需批准
                "read_data": False,
            },
            # 中断消息的前缀 — 与工具名称和参数组合形成完整消息
            # 例如，"Tool execution pending approval: execute_sql with query='DELETE FROM...'"
            # 各个工具可以通过在中断配置中指定"description"来覆盖此设置
            description_prefix="Tool execution pending approval",
        ),
    ],
    # 人工介入需要检查点来处理中断。
    # 在生产环境中，使用像 AsyncPostgresSaver 这样的持久化检查点。
    checkpointer=InMemorySaver(),
)
```

您必须配置检查点以在中断之间持久化图状态。在生产环境中，使用像 [`AsyncPostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver) 这样的持久化检查点。对于测试或原型设计，使用 [`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)。

调用 agent 时，传递一个包含**线程 ID**的 `config`，以将执行与对话线程关联。请参阅 [LangGraph 中断文档(https://docs.langchain.com/oss/python/langgraph/interrupts) 了解详细信息。

**配置选项：**

- `interrupt_on`（必需）：工具名称到批准配置的映射。值可以是 `True`（使用默认配置中断）、`False`（自动批准）或 `InterruptOnConfig` 对象。
- `description_prefix`（默认："Tool execution requires approval"）：操作请求描述的前缀

**`InterruptOnConfig` 选项：**

- `allowed_decisions`：允许的决策列表：`'approve'`、`'edit'` 或 `'reject'`
- `description`：静态字符串或用于自定义描述的可调用函数

## 响应中断

调用 agent 时，它会运行直到完成或引发中断。当工具调用与您在 `interrupt_on` 中配置的策略匹配时，会触发中断。在这种情况下，调用结果将包含一个 `__interrupt__` 字段，其中包含需要审查的操作。然后，您可以向审阅者展示这些操作，并在提供决策后恢复执行。

```python
from langgraph.types import Command

# 人工介入利用 LangGraph 的持久化层。
# 您必须提供线程 ID 以将执行与对话线程关联，
# 以便可以暂停和恢复对话（人工审查需要这样做）。
config = {"configurable": {"thread_id": "some_id"}}
# 运行图直到命中中断。
result = agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": "Delete old records from the database",
            }
        ]
    },
    config=config
)

# 中断包含带有 action_requests 和 review_configs 的完整 HITL 请求
print(result['__interrupt__'])

# 使用批准决策恢复
agent.invoke(
    Command(
        resume={"decisions": [{"type": "approve"}]}  # 或 "reject"
    ),
    config=config  # 相同的线程 ID 以恢复暂停的对话
)
```

### 决策类型

**批准（approve）：**

使用 `approve` 按原样批准工具调用，无需更改即可执行。

```python
agent.invoke(
    Command(
        # 决策以列表形式提供，每个待审阅的操作一个。
        # 决策的顺序必须与 `__interrupt__` 请求中列出的操作顺序匹配。
        resume={
            "decisions": [
                {
                    "type": "approve",
                }
            ]
        }
    ),
    config=config  # 相同的线程 ID 以恢复暂停的对话
)
```

**编辑（edit）：**

使用 `edit` 在执行前修改工具调用。提供带有新工具名称和参数的被编辑操作。

```python
agent.invoke(
    Command(
        # 决策以列表形式提供，每个待审阅的操作一个。
        # 决策的顺序必须与 `__interrupt__` 请求中列出的操作顺序匹配。
        resume={
            "decisions": [
                {
                    "type": "edit",
                    # 带工具名称和参数的被编辑操作
                    "edited_action": {
                        # 要调用的工具名称。
                        # 通常与原始操作相同。
                        "name": "new_tool_name",
                        # 传递给工具的参数。
                        "args": {"key1": "new_value", "key2": "original_value"},
                    }
                }
            ]
        }
    ),
    config=config  # 相同的线程 ID 以恢复暂停的对话
)
```

**编辑**工具参数时，请保守地进行更改。对原始参数的重大修改可能导致模型重新评估其方法，并可能多次执行工具或采取意外操作。

**拒绝（reject）：**

使用 `reject` 拒绝工具调用并提供反馈而不是执行。

```python
agent.invoke(
    Command(
        # 决策以列表形式提供，每个待审阅的操作一个。
        # 决策的顺序必须与 `__interrupt__` 请求中列出的操作顺序匹配。
        resume={
            "decisions": [
                {
                    "type": "reject",
                    # 关于为什么拒绝操作的解释
                    "message": "No, this is wrong because ..., instead do this ...",
                }
            ]
        }
    ),
    config=config  # 相同的线程 ID 以恢复暂停的对话
)
```

`message` 作为反馈添加到对话中，以帮助 agent 理解为什么操作被拒绝以及应该做什么。

**多个决策：**

当多个操作正在审阅时，按照它们在中断中出现的相同顺序为每个操作提供决策：

```python
{
    "decisions": [
        {"type": "approve"},
        {
            "type": "edit",
            "edited_action": {
                "name": "tool_name",
                "args": {"param": "new_value"}
            }
        },
        {
            "type": "reject",
            "message": "This action is not allowed"
        }
    ]
}
```

## 使用人工介入进行流式传输

您可以使用 `stream()` 而不是 `invoke()` 来在 agent 运行和处理中断时获取实时更新。使用 `stream_mode=['updates', 'messages']` 来流式传输 agent 进度和 LLM token。

```python
from langgraph.types import Command

config = {"configurable": {"thread_id": "some_id"}}

# 流式传输 agent 进度和 LLM token 直到中断
for mode, chunk in agent.stream(
    {"messages": [{"role": "user", "content": "Delete old records from the database"}]},
    config=config,
    stream_mode=["updates", "messages"],
):
    if mode == "messages":
        # LLM token
        token, metadata = chunk
        if token.content:
            print(token.content, end="", flush=True)
    elif mode == "updates":
        # 检查中断
        if "__interrupt__" in chunk:
            print(f"\n\nInterrupt: {chunk['__interrupt__']}")

# 人工决策后使用流式传输恢复
for mode, chunk in agent.stream(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config,
    stream_mode=["updates", "messages"],
):
    if mode == "messages":
        token, metadata = chunk
        if token.content:
            print(token.content, end="", flush=True)
```

有关流模式的更多详细信息，请参阅[流式传输(https://docs.langchain.com/oss/python/langchain/streaming)指南。

## 执行生命周期

中间件定义了一个 `after_model` 钩子，该钩子在模型生成响应之后但在任何工具调用执行之前运行：

1. agent 调用模型生成响应。
2. 中间件检查响应中的工具调用。
3. 如果任何调用需要人工输入，中间件会构建一个包含 `action_requests` 和 `review_configs` 的 `HITLRequest` 并调用[中断](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)。
4. agent 等待人工决策。
5. 根据 `HITLResponse` 决策，中间件执行批准或编辑的调用，为拒绝的调用合成 [ToolMessage](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage)，并恢复执行。

## 自定义 HITL 逻辑

对于更专业的工作流程，您可以使用[中断](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)原语和[中间件(https://docs.langchain.com/oss/python/langchain/middleware)抽象直接构建自定义 HITL 逻辑。

请参阅上面的[执行生命周期](#执行生命周期)了解如何将中断集成到 agent 的操作中。


