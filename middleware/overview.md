# 概述

> 在每个步骤中控制和自定义智能体执行

中间件提供了一种更紧密地控制智能体内部发生情况的方法。中间件适用于以下场景：

* 使用日志、分析和调试跟踪智能体行为。
* 转换提示词、工具选择和输出格式。
* 添加重试、回退和提前终止逻辑。
* 应用速率限制、护栏和 PII 检测。

通过将中间件传递给 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 来添加中间件：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware, HumanInTheLoopMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[
        SummarizationMiddleware(...),
        HumanInTheLoopMiddleware(...)
    ],
)
```

## 智能体循环

核心智能体循环涉及调用模型、让它选择要执行的工具，然后在它不再调用工具时完成：

中间件在这些步骤的前后暴露了钩子：

## 附加资源

- [内置中间件](built-in-middleware.md)
- [自定义中间件](custom-middleware.md)
- [中间件 API 参考](https://reference.langchain.com/python/langchain/middleware/)
- [测试智能体](../agent-development/test.md)

---

