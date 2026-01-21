# 自定义工作流

在**自定义工作流**架构中，您使用 [LangGraph](/oss/python/langgraph/overview) 定义自己的自定义执行流。您可以完全控制图结构——包括顺序步骤、条件分支、循环和并行执行。

```mermaid
graph LR
    A([输入]) --> B{{条件}}
    B -->|路径_a| C[确定性步骤]
    B -->|路径_b| D((智能体步骤))
    C --> G([输出])
    D --> G([输出])
```

## 关键特征

* 完全控制图结构
* 将确定性逻辑与智能体行为混合
* 支持顺序步骤、条件分支、循环和并行执行
* 将其他模式作为节点嵌入您的工作流中

## 使用场景

当标准模式（子智能体、技能等）不符合您的需求时，使用自定义工作流，当您需要将确定性逻辑与智能体行为混合时，或您的用例需要复杂路由或多阶段处理时。

工作流中的每个节点可以是一个简单的函数、LLM 调用或带有[工具](/oss/python/langchain/tools)的整个[智能体](/oss/python/langchain/agents)。您也可以在自定义工作流中组合其他架构——例如，将多智能体系统嵌入作为单个节点。

有关自定义工作流的完整示例，请参阅下面的教程。

**教程：使用路由构建多源知识库**

[路由器模式](/oss/python/langchain/multi-agent/router) 是自定义工作流的一个例子。本教程逐步介绍构建一个并行查询 GitHub、Notion 和 Slack 的路由器，然后合成结果。

>

## 基本实现

核心洞察是您可以在任何 LangGraph 节点中直接调用 LangChain 智能体，将自定义工作流的灵活性与预构建智能体的便利性相结合：

```python
from langchain.agents import create_agent
from langgraph.graph import StateGraph, START, END

agent = create_agent(model="openai:gpt-4o", tools=[...])

def agent_node(state: State) -> dict:
    """调用 LangChain 智能体的 LangGraph 节点。"""
    result = agent.invoke({
        "messages": [{"role": "user", "content": state["query"]}]
    })
    return {"answer": result["messages"][-1].content}

# 构建简单工作流
workflow = (
    StateGraph(State)
    .add_node("agent", agent_node)
    .add_edge(START, "agent")
    .add_edge("agent", END)
    .compile()
)
```

## 示例：RAG 流程

一个常见的用例是将[检索](/oss/python/langchain/retrieval)与智能体结合。此示例构建了一个从知识库检索并可以获取实时新闻的 WNBA 统计助手。

**自定义 RAG 工作流：**

该工作流演示了三种类型的节点：

* **模型节点**（重写）：使用[结构化输出](/oss/python/langchain/structured-output)重写用户查询以获得更好的检索。
* **确定性节点**（检索）：执行向量相似性搜索——不涉及 LLM。
* **智能体节点**（智能体）：对检索到的上下文进行推理，并可以通过工具获取额外信息。

```mermaid
graph LR
    A([查询]) --> B{{重写}}
    B --> C[(检索)]
    C --> D((智能体))
    D --> E([响应])
```

<Tip>
  您可以使用 LangGraph 状态在工作流步骤之间传递信息。这允许您工作流的每个部分读取和更新结构化字段，使跨节点共享数据和上下文变得容易。
</Tip>

```python
from typing import TypedDict
from pydantic import BaseModel
from langgraph.graph import StateGraph, START, END
from langchain.agents import create_agent
from langchain.tools import tool
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore

class State(TypedDict):
    question: str
    rewritten_query: str
    documents: list[str]
    answer: str

# WNBA 知识库，包含名册、比赛结果和球员统计
embeddings = OpenAIEmbeddings()
vector_store = InMemoryVectorStore(embeddings)
vector_store.add_texts([
    # 名册
    "New York Liberty 2024 roster: Breanna Stewart, Sabrina Ionescu, Jonquel Jones, Courtney Vandersloot.",
    "Las Vegas Aces 2024 roster: A'ja Wilson, Kelsey Plum, Jackie Young, Chelsea Gray.",
    "Indiana Fever 2024 roster: Caitlin Clark, Aliyah Boston, Kelsey Mitchell, NaLyssa Smith.",
    # 比赛结果
    "2024 WNBA Finals: New York Liberty defeated Minnesota Lynx 3-2 to win the championship.",
    "June 15, 2024: Indiana Fever 85, Chicago Sky 79. Caitlin Clark had 23 points and 8 assists.",
    "August 20, 2024: Las Vegas Aces 92, Phoenix Mercury 84. A'ja Wilson scored 35 points.",
    # 球员统计
    "A'ja Wilson 2024 season stats: 26.9 PPG, 11.9 RPG, 2.6 BPG. Won MVP award.",
    "Caitlin Clark 2024 rookie stats: 19.2 PPG, 8.4 APG, 5.7 RPG. Won Rookie of the Year.",
    "Breanna Stewart 2024 stats: 20.4 PPG, 8.5 RPG, 3.5 APG.",
])
retriever = vector_store.as_retriever(search_kwargs={"k": 5})

@tool
def get_latest_news(query: str) -> str:
    """获取最新的 WNBA 新闻和更新。"""
    # 您的新闻 API 在这里
    return "Latest: The WNBA announced expanded playoff format for 2025..."

agent = create_agent(
    model="openai:gpt-4o",
    tools=[get_latest_news],
)

model = ChatOpenAI(model="gpt-4o")

class RewrittenQuery(BaseModel):
    query: str

def rewrite_query(state: State) -> dict:
    """重写用户查询以获得更好的检索。"""
    system_prompt = """重写此查询以检索相关的 WNBA 信息。
知识库包含：球队名册、比赛结果及比分，以及球员统计数据（PPG、RPG、APG）。
关注特定的球员名字、球队名称或统计类别。"""
    response = model.with_structured_output(RewrittenQuery).invoke([
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": state["question"]}
    ])
    return {"rewritten_query": response.query}

def retrieve(state: State) -> dict:
    """根据重写的查询检索文档。"""
    docs = retriever.invoke(state["rewritten_query"])
    return {"documents": [doc.page_content for doc in docs]}

def call_agent(state: State) -> dict:
    """使用检索到的上下文生成答案。"""
    context = "\n\n".join(state["documents"])
    prompt = f"Context:\n{context}\n\nQuestion: {state['question']}"
    response = agent.invoke({"messages": [{"role": "user", "content": prompt}]})
    return {"answer": response["messages"][-1].content_blocks}

workflow = (
    StateGraph(State)
    .add_node("rewrite", rewrite_query)
    .add_node("retrieve", retrieve)
    .add_node("agent", call_agent)
    .add_edge(START, "rewrite")
    .add_edge("rewrite", "retrieve")
    .add_edge("retrieve", "agent")
    .add_edge("agent", END)
    .compile()
)

result = workflow.invoke({"question": "谁赢得了 2024 年 WNBA 总冠军？"})
print(result["answer"])
```

