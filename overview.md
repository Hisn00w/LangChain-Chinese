LangChain 概览


LangChain 是一个开源框架，提供**预构建的 Agent 架构**，以及对**任意模型或工具的集成能力**——使你能够构建可以随着生态系统快速演进而同样快速适配的智能体。

LangChain 是开始构建由大语言模型（LLM）驱动的智能体（Agent）和应用程序的**最简单方式**。只需不到 10 行代码，你就可以连接到 OpenAI、Anthropic、Google，以及[更多提供商](../integrations/providers/overview.md)。  
LangChain 提供预构建的 Agent 架构与模型集成，帮助你快速上手，并将 LLM 无缝融入你的智能体与应用中。

如果你希望**快速构建智能体与自治应用**，我们推荐使用 LangChain。  
当你有更高级需求（需要结合**确定性流程**与**智能体流程**、进行**深度自定义**、并对**延迟进行精细控制**）时，请使用 LangGraph——它是一个更底层的智能体编排框架与运行时：  
[LangGraph](../langgraph/overview.md)

LangChain 的 [Agents](agents.md) 构建在 LangGraph 之上，以提供：

- 持久化执行（durable execution）
- 流式输出（streaming）
- 人类介入（human-in-the-loop）
- 状态持久化（persistence）
- 以及更多能力

在进行基本的 LangChain Agent 使用时，**你不需要事先了解 LangGraph**。

---

## ✨ 创建一个 Agent

```python
# pip install -qU langchain "langchain[anthropic]"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
