# LangSmith Studio

在本地使用 LangChain 构建 agent 时，可视化 agent 内部发生的事情、实时与之交互以及调试出现的问题非常有用。**LangSmith Studio** 是一个免费的视觉界面，用于从本地机器开发和测试您的 LangChain agent。

Studio 连接到您本地运行的 agent，向您展示 agent 执行的每个步骤：发送给模型的提示、工具调用及其结果，以及最终输出。您可以测试不同的输入、检查中间状态，并迭代 agent 的行为，而无需额外的代码或部署。

本页描述如何设置 Studio 与您的本地 LangChain agent。

## 前置要求

在开始之前，请确保您具备以下条件：

- **LangSmith 账户**：在 [smith.langchain.com](https://smith.langchain.com) 注册（免费）或登录。
- **LangSmith API key**：按照[创建 API key](https://docs.langchain.com/langsmith/create-account-api-key#create-an-api-key) 指南操作。
- 如果您不希望数据被[追踪](https://docs.langchain.com/langsmith/observability-concepts#traces)到 LangSmith，请在应用程序的 `.env` 文件中设置 `LANGSMITH_TRACING=false`。禁用追踪后，不会数据离开您的本地服务器。

## 设置本地 Agent 服务器

### 1. 安装 LangGraph CLI

[LangGraph CLI](https://docs.langchain.com/langsmith/cli) 提供了一个本地开发服务器（也称为 [Agent Server](https://docs.langchain.com/langsmith/agent-server)），将您的 agent 连接到 Studio。

```shell
# 需要 Python >= 3.11。
pip install --upgrade "langgraph-cli[inmem]"
```

### 2. 准备您的 agent

如果您已经有一个 LangChain agent，可以直接使用。以下示例使用一个简单的电子邮件 agent：

```python
from langchain.agents import create_agent

def send_email(to: str, subject: str, body: str):
    """Send an email"""
    email = {
        "to": to,
        "subject": subject,
        "body": body
    }
    # ... 电子邮件发送逻辑

    return f"Email sent to {to}"

agent = create_agent(
    "gpt-4o",
    tools=[send_email],
    system_prompt="You are an email assistant. Always use the send_email tool.",
)
```

### 3. 环境变量

Studio 需要 LangSmith API key 来连接您的本地 agent。在项目的根目录创建 `.env` 文件，并添加来自 [LangSmith](https://smith.langchain.com/settings) 的 API key。

确保您的 `.env` 文件不会被提交到版本控制（如 Git）。

```bash
LANGSMITH_API_KEY=lsv2...
```

### 4. 创建 LangGraph 配置文件

LangGraph CLI 使用配置文件来定位您的 agent 并管理依赖项。在应用程序目录中创建 `langgraph.json` 文件：

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.py:agent"
  },
  "env": ".env"
}
```

[`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 函数自动返回已编译的 LangGraph 图，这就是配置文件中 `graphs` 键所期望的内容。

有关配置文件 JSON 对象中每个键的详细解释，请参阅 [LangGraph 配置文件参考](https://docs.langchain.com/langsmith/cli#configuration-file)。

此时，项目结构将如下所示：

```bash
my-app/
├── src
│   └── agent.py
├── .env
└── langgraph.json
```

### 5. 安装依赖项

从根目录安装项目依赖项：

**pip：**

```shell
pip install langchain langchain-openai
```

**uv：**

```shell
uv add langchain langchain-openai
```

### 6. 在 Studio 中查看您的 agent

启动开发服务器以将您的 agent 连接到 Studio：

```shell
langgraph dev
```

Safari 阻止 `localhost` 连接到 Studio。要解决此问题，请使用 `--tunnel` 运行上述命令，通过安全隧道访问 Studio。

服务器运行后，您的 agent 可以通过 `http://127.0.0.1:2024` 的 API 访问，也可以通过 `https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024` 的 Studio UI 访问。

通过 Studio 连接到您的本地 agent，您可以快速迭代 agent 的行为。运行测试输入，检查包含提示、工具参数、返回值以及 token/延迟指标的完整执行追踪。当出现问题时，Studio 会捕获异常及其周围状态，以帮助您了解发生了什么。

开发服务器支持热重载 — 在代码中更改提示或工具签名，Studio 会立即反映这些更改。从任何步骤重新运行对话线程以测试您的更改，而无需重新开始。此工作流程从简单的单工具 agent 扩展到复杂的多节点图。

有关如何运行 Studio 的更多信息，请参阅 LangSmith 文档中的以下指南：

- [运行应用程序](https://docs.langchain.com/langsmith/use-studio#run-application)
- [管理助手](https://docs.langchain.com/langsmith/use-studio#manage-assistants)
- [管理线程](https://docs.langchain.com/langsmith/use-studio#manage-threads)
- [迭代提示](https://docs.langchain.com/langsmith/observability-studio)
- [调试 LangSmith 追踪](https://docs.langchain.com/langsmith/observability-studio#debug-langsmith-traces)
- [将节点添加到数据集](https://docs.langchain.com/langsmith/observability-studio#add-node-to-dataset)


