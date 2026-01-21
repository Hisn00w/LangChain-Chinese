# LangSmith 部署

当您准备好将 LangChain 智能体部署到生产环境时，LangSmith 提供了一个专为智能体工作负载设计的托管托管平台。传统的托管平台是为无状态的、短期运行的 Web 应用程序构建的，而 LangGraph 是**专门为需要持久状态和后台执行的有状态的、长时间运行的智能体构建的**。LangSmith 处理基础设施、扩展和运营问题，因此您可以直接从仓库部署。

## 前置条件

在开始之前，请确保您具备以下条件：

* 一个 [GitHub 账户](https://github.com/)
* 一个 [LangSmith 账户](https://smith.langchain.com/)（免费注册）

## 部署您的智能体

### 1. 在 GitHub 上创建仓库

您的应用程序代码必须位于 GitHub 仓库中才能在 LangSmith 上部署。公共和私有仓库都支持。对于此快速入门，首先确保您的应用通过遵循[本地服务器设置指南(https://docs.langchain.com/oss/python/langchain/studio#setup-local-agent-server)与 LangGraph 兼容。然后，将您的代码推送到仓库。

### 2. 部署到 LangSmith

**步骤 1：导航到 LangSmith 部署**

登录 [LangSmith](https://smith.langchain.com/)。在左侧边栏中，选择**部署**。

**步骤 2：创建新部署**

点击 **+ 新建部署** 按钮。将打开一个窗格，您可以填写必填字段。

**步骤 3：链接仓库**

如果您是首次用户或添加以前未连接过的私有仓库，请点击 **添加新账户** 按钮，然后按照说明连接您的 GitHub 账户。

**步骤 4：部署仓库**

选择您的应用程序仓库。点击 **提交** 进行部署。这可能需要大约 15 分钟才能完成。您可以在**部署详情**视图中检查状态。

### 3. 在 Studio 中测试您的应用程序

部署应用程序后：

1. 选择您刚刚创建的部署以查看更多详情。
2. 点击右上角的 **Studio** 按钮。Studio 将打开并显示您的图。

### 4. 获取部署的 API URL

1. 在 LangGraph 的**部署详情**视图中，点击 **API URL** 将其复制到剪贴板。
2. 点击 `URL` 将其复制到剪贴板。

### 5. 测试 API

现在您可以测试 API：

**Python：**

1. 安装 LangGraph Python：

```bash
pip install langgraph-sdk
```

2. 向智能体发送消息：

```python
from langgraph_sdk import get_sync_client  # 或使用 get_client 进行异步操作

client = get_sync_client(url="your-deployment-url", api_key="your-langsmith-api-key")

for chunk in client.runs.stream(
    None,    # 无线程运行
    "agent", # 智能体名称。在 langgraph.json 中定义。
    input={
        "messages": [{
            "role": "human",
            "content": "什么是 LangGraph？",
        }],
    },
    stream_mode="updates",
):
    print(f"收到新事件，类型为：{chunk.event}...")
    print(chunk.data)
    print("\n\n")
```

**REST API：**

```bash
curl -s --request POST \
    --url <DEPLOYMENT_URL>/runs/stream \
    --header 'Content-Type: application/json' \
    --header "X-Api-Key: <LANGSMITH API KEY> \
    --data "{
        \"assistant_id\": \"agent\", `# 智能体名称。在 langgraph.json 中定义。`
        \"input\": {
            \"messages\": [
                {
                    \"role\": \"human\",
                    \"content\": \"什么是 LangGraph？\"
                }
            ]
        },
        \"stream_mode\": \"updates\"
    }"
```

LangSmith 提供额外的托管选项，包括自托管和混合托管。更多信息，请参阅[平台设置概述](https://docs.langchain.com/langsmith/platform-setup)。



