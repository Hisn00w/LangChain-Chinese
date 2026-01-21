# Agent 聊天界面

[Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) 是一个 Next.js 应用程序，为与任何 LangChain agent 交互提供对话界面。它支持实时聊天、工具可视化以及时间旅行调试和状态分叉等高级功能。Agent Chat UI 与使用 [`create_agent`](https://docs.langchain.com/oss/python/langchain/agents) 创建的 agent 无缝协作，无论您是在本地运行还是在部署的环境中（如 [LangSmith(https://docs.smith.langchain.com/home)），都能以最少的设置为您的 agent 提供交互式体验。

Agent Chat UI 是开源的，可以根据您的应用程序需求进行适配。

### 快速开始

最快的方式是使用托管版本：

1. **访问 [Agent Chat UI](https://agentchat.vercel.app)**
2. **连接您的 agent** — 输入您的部署 URL 或本地服务器地址
3. **开始聊天** — 界面将自动检测并呈现工具调用和中断

### 本地开发

要进行自定义或本地开发，您可以本地运行 Agent Chat UI：

**使用 npx：**

```bash
# 创建新的 Agent Chat UI 项目
npx create-agent-chat-app --project-name my-chat-ui
cd my-chat-ui

# 安装依赖并启动
pnpm install
pnpm dev
```

**克隆仓库：**

```bash
# 克隆仓库
git clone https://github.com/langchain-ai/agent-chat-ui.git
cd agent-chat-ui

# 安装依赖并启动
pnpm install
pnpm dev
```

### 连接到您的 agent

Agent Chat UI 可以连接[本地(https://docs.langchain.com/oss/python/langchain/studio#setup-local-agent-server)和[部署的 agent(https://docs.langchain.com/oss/python/langchain/deploy)。

启动 Agent Chat UI 后，您需要配置它以连接到您的 agent：

1. **Graph ID**：输入您的图名称（在 `langgraph.json` 文件的 `graphs` 下找到）
2. **部署 URL**：您的 Agent 服务器端点（例如，用于本地开发的 `http://localhost:2024`，或您的部署 agent 的 URL）
3. **LangSmith API key（可选）**：添加您的 LangSmith API key（如果您使用本地 Agent 服务器，则不需要）

配置后，Agent Chat UI 将自动获取并显示来自您的 agent 的任何中断线程。

Agent Chat UI 开箱即用地支持呈现工具调用和工具结果消息。要自定义显示的消息，请参阅[在聊天中隐藏消息](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat)。

您可以在 Agent Chat UI 中使用生成式界面。更多信息，请参阅[使用 LangGraph 实现生成式用户界面(https://docs.smith.langchain.com/generative-ui-react)。


