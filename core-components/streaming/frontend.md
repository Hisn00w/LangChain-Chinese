# 前端

> 使用 LangChain 智能体、LangGraph 图和自定义 API 构建具有实时流式传输功能的生成式 UI

`useStream` React 钩子提供与 LangGraph 流式传输功能的无缝集成。它处理流式传输、状态管理和分支逻辑的所有复杂性，让您专注于构建出色的生成式 UI 体验。

主要功能：

* **消息流式传输** — 处理消息块的流以形成完整消息
* **自动状态管理** — 用于消息、中断、加载状态和错误
* **对话分支** — 从聊天历史中的任何点创建备选对话路径
* **UI 无关设计** — 使用您自己的组件和样式

## 安装

安装 LangGraph SDK 以在 React 应用程序中使用 `useStream` 钩子：

## 基本用法

`useStream` 钩子连接到任何 LangGraph 图，无论是运行在您自己的端点还是使用 [LangSmith 部署](/langsmith/deployments) 部署的。

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";

function Chat() {
  const stream = useStream({
    assistantId: "agent",
    // 本地开发
    apiUrl: "http://localhost:2024",
    // 生产部署（LangSmith 托管）
    // apiUrl: "https://your-deployment.us.langgraph.app"
  });

  const handleSubmit = (message: string) => {
    stream.submit({
      messages: [
        { content: message, type: "human" }
      ],
    });
  };

  return (
    <div>
      {stream.messages.map((message, idx) => (
        <div key={message.id ?? idx}>
          {message.type}: {message.content}
        </div>
      ))}

      {stream.isLoading && <div>加载中...</div>}
      {stream.error && <div>错误: {stream.error.message}</div>}
    </div>
  );
}
```

了解如何[将您的智能体部署到 LangSmith](/oss/python/langchain/deploy)，以获得具有内置可观测性、身份验证和扩展的生产级托管。

**`useStream` 参数：**

| 参数 | 类型 | 描述 |
|------|------|------|
| `assistantId` | string（必需） | 要连接的智能体的 ID。使用 LangSmith 部署时，这必须与部署仪表板中显示的智能体 ID 匹配。对于自定义 API 部署或本地开发，这可以是服务器用来识别智能体的任何字符串。 |
| `apiUrl` | string | LangGraph 服务器的 URL。本地开发默认为 `http://localhost:2024`。 |
| `apiKey` | string | 身份验证的 API 密钥。连接到 LangSmith 上的部署智能体时需要。 |
| `threadId` | string | 连接到现有线程而不是创建新线程。可用于恢复对话。 |
| `onThreadId` | (id: string) => void | 创建新线程时调用的回调。使用此方法持久化线程 ID 以供以后使用。 |
| `reconnectOnMount` | boolean \| (() => Storage) | 组件挂载时自动恢复正在进行的运行。设置为 `true` 以使用会话存储，或提供自定义存储函数。 |
| `onCreated` | (run: Run) => void | 创建新运行时调用的回调。可用于持久化运行元数据以进行恢复。 |
| `onError` | (error: Error) => void | 流式传输过程中发生错误时调用。 |
| `onFinish` | (state: StateType, run?: Run) => void | 流成功完成时调用，包含最终状态。 |
| `onCustomEvent` | (data: unknown, context: { mutate }) => void | 处理使用 `writer` 从智能体发出的自定义事件。请参阅[自定义流式传输事件](#custom-streaming-events)。 |
| `onUpdateEvent` | (data: unknown, context: { mutate }) => void | 在每个图步骤之后处理状态更新事件。 |
| `onMetadataEvent` | (metadata: { run_id, thread_id }) => void | 处理包含运行和线程信息的元数据事件。 |
| `messagesKey` | string | 图状态中包含消息数组的键。默认为 `messages`。 |
| `throttle` | boolean | 批量状态更新以获得更好的渲染性能。禁用以获得即时更新。默认为 `true`。 |
| `initialValues` | StateType \| null | 在第一个流加载时显示的初始状态值。可用于立即显示缓存的线程数据。 |

**`useStream` 返回值：**

| 属性 | 类型 | 描述 |
|------|------|------|
| `messages` | Message[] | 当前线程中的所有消息，包括人类和 AI 消息。 |
| `values` | StateType | 当前图状态值。类型从智能体或图类型参数推断。 |
| `isLoading` | boolean | 当前是否有流正在进行。使用此选项显示加载指示器。 |
| `error` | Error \| null | 流式传输过程中发生的任何错误。无错误时为 `null`。 |
| `interrupt` | Interrupt \| undefined | 当前需要用户输入的中断，例如人工介入批准请求。 |
| `toolCalls` | ToolCallWithResult[] | 所有消息中的所有工具调用，包括其结果和状态（`pending`、`completed` 或 `error`）。 |
| `submit` | (input, options?) => Promise<void> | 向智能体提交新输入。当使用命令从中断恢复时传递 `null` 作为输入。选项包括用于分支的 `checkpoint`、用于乐观更新的 `optimisticValues` 和用于乐观线程创建的 `threadId`。 |
| `stop` | () => void | 立即停止当前流。 |
| `joinStream` | (runId: string) => void | 通过运行 ID 恢复现有流。与 `onCreated` 一起用于手动流恢复。 |
| `setBranch` | (branch: string) => void | 切换到对话历史中的不同分支。 |
| `getToolCalls` | (message) => ToolCall[] | 获取特定 AI 消息的所有工具调用。 |
| `getMessagesMetadata` | (message) => MessageMetadata | 获取消息的元数据，包括流式传输信息（如用于标识源节点的 `langgraph_node` 和用于分支的 `firstSeenState`）。 |
| `experimental_branchTree` | BranchTree | 线程的树表示，用于非基于消息的图中的高级分支控制。 |

## 线程管理

使用内置线程管理跟踪对话。您可以访问当前线程 ID，并在创建新线程时收到通知：

```tsx
import { useState } from "react";
import { useStream } from "@langchain/langgraph-sdk/react";

function Chat() {
  const [threadId, setThreadId] = useState<string | null>(null);

  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId: threadId,
    onThreadId: setThreadId,
  });

  // 创建新线程时更新 threadId
  // 将其存储在 URL 参数或 localStorage 中以实现持久化
}
```

我们建议存储 `threadId` 以让用户在页面刷新后恢复对话。

### 页面刷新后恢复

通过设置 `reconnectOnMount: true`，`useStream` 钩子可以在挂载时自动恢复正在进行的运行。这对于在页面刷新后继续流很有用，确保不会丢失停机期间生成的消息和事件。

```tsx
const stream = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: true,
});
```

默认情况下，创建的运行的 ID 存储在 `window.sessionStorage` 中，可以通过传递自定义存储函数来交换：

```tsx
const stream = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: () => window.localStorage,
});
```

要手动控制恢复过程，使用运行回调持久化元数据并使用 `joinStream` 来恢复：

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import { useEffect, useRef } from "react";

function Chat({ threadId }: { threadId: string | null }) {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    onCreated: (run) => {
      // 流开始时持久化运行 ID
      window.sessionStorage.setItem(`resume:${run.thread_id}`, run.run_id);
    },
    onFinish: (_, run) => {
      // 流完成时清理
      window.sessionStorage.removeItem(`resume:${run?.thread_id}`);
    },
  });

  // 挂载时恢复流（如果有存储的运行 ID）
  const joinedThreadId = useRef<string | null>(null);
  useEffect(() => {
    if (!threadId) return;
    const runId = window.sessionStorage.getItem(`resume:${threadId}`);
    if (runId && joinedThreadId.current !== threadId) {
      stream.joinStream(runId);
      joinedThreadId.current = threadId;
    }
  }, [threadId]);

  const handleSubmit = (text: string) => {
    // 使用 streamResumable 确保事件不会丢失
    stream.submit(
      { messages: [{ type: "human", content: text }] },
      { streamResumable: true }
    );
  };
}
```

查看 `session-persistence` 示例中带有 `reconnectOnMount` 和线程持久化的流恢复的完整实现。

## 乐观更新

您可以在执行网络请求之前乐观地更新客户端状态，向用户提供即时反馈：

```tsx
const stream = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
});

const handleSubmit = (text: string) => {
  const newMessage = { type: "human" as const, content: text };

  stream.submit(
    { messages: [newMessage] },
    {
      optimisticValues(prev) {
        const prevMessages = prev.messages ?? [];
        return { ...prev, messages: [...prevMessages, newMessage] };
      },
    }
  );
};
```

### 乐观线程创建

在 `submit` 中使用 `threadId` 选项来启用乐观 UI 模式，当您需要在线程创建之前知道线程 ID 时：

```tsx
import { useState } from "react";
import { useStream } from "@langchain/langgraph-sdk/react";

function Chat() {
  const [threadId, setThreadId] = useState<string | null>(null);
  const [optimisticThreadId] = useState(() => crypto.randomUUID());

  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    onThreadId: setThreadId,
  });

  const handleSubmit = (text: string) => {
    // 立即导航而不等待线程创建
    window.history.pushState({}, "", `/threads/${optimisticThreadId}`);

    // 使用预定 ID 创建线程
    stream.submit(
      { messages: [{ type: "human", content: text }] },
      { threadId: optimisticThreadId }
    );
  };
}
```

### 缓存线程显示

使用 `initialValues` 选项在从服务器加载历史记录时立即显示缓存的线程数据：

```tsx
function Chat({ threadId, cachedData }) {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    initialValues: cachedData?.values,
  });

  // 立即显示缓存的消息，然后在服务器响应时更新
}
```

## 分支

通过编辑之前的消息或重新生成 AI 响应来创建备选对话路径。使用 `getMessagesMetadata()` 访问分支的检查点信息：

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import { BranchSwitcher } from "./BranchSwitcher";

function Chat() {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
  });

  return (
    <div>
      {stream.messages.map((message) => {
        const meta = stream.getMessagesMetadata(message);
        const parentCheckpoint = meta?.firstSeenState?.parent_checkpoint;

        return (
          <div key={message.id}>
            <div>{message.content as string}</div>

            {/* 编辑人类消息 */}
            {message.type === "human" && (
              <button
                onClick={() => {
                  const newContent = prompt("编辑消息:", message.content as string);
                  if (newContent) {
                    stream.submit(
                      { messages: [{ type: "human", content: newContent }] },
                      { checkpoint: parentCheckpoint }
                    );
                  }
                }}
              >
                编辑
              </button>
            )}

            {/* 重新生成 AI 消息 */}
            {message.type === "ai" && (
              <button
                onClick={() => stream.submit(undefined, { checkpoint: parentCheckpoint })}
              >
                重新生成
              </button>
            )}

            {/* 在分支之间切换 */}
            <BranchSwitcher
              branch={meta?.branch}
              branchOptions={meta?.branchOptions}
              onSelect={(branch) => stream.setBranch(branch)}
            />
          </div>
        );
      })}
    </div>
  );
}

/**
 * 用于在对话分支之间导航的组件。
 * 显示当前分支位置并允许在备选方案之间切换。
 */
export function BranchSwitcher({
  branch,
  branchOptions,
  onSelect,
}: {
  branch: string | undefined;
  branchOptions: string[] | undefined;
  onSelect: (branch: string) => void;
}) {
  if (!branchOptions || !branch) return null;
  const index = branchOptions.indexOf(branch);

  return (
    <div className="flex items-center gap-2">
      <button
        type="button"
        disabled={index <= 0}
        onClick={() => onSelect(branchOptions[index - 1])}
      >
        ←
      </button>
      <span>{index + 1} / {branchOptions.length}</span>
      <button
        type="button"
        disabled={index >= branchOptions.length - 1}
        onClick={() => onSelect(branchOptions[index + 1])}
      >
        →
      </button>
    </div>
  );
}
```

查看 `branching-chat` 示例中带有编辑、重新生成和分支切换的对话分支的完整实现。

## 类型安全的流式传输

当与通过 @[`createAgent`] 创建的智能体或使用 [`StateGraph`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph) 创建的图一起使用时，`useStream` 钩子支持完整类型推断。传递 `typeof agent` 或 `typeof graph` 作为类型参数以自动推断工具调用类型。

### 使用 `createAgent`

使用 @[`createAgent`] 时，工具调用类型会自动从您注册到智能体的工具中推断：

```python
from langchain import create_agent, tool

@tool
def get_weather(location: str) -> str:
    """获取地点的天气。"""
    return f"地点天气: 晴天, 22°C"

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_weather],
)
```

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import type { AgentState } from "./types";

function Chat() {
  // 使用手动定义的状态类型
  const stream = useStream<AgentState>({
    assistantId: "agent",
    apiUrl: "http://localhost:2024",
  });

  // stream.toolCalls[0].call.name 键入为 "get_weather"
  // stream.toolCalls[0].call.args 键入为 { location: string }
}
```

```typescript
import type { Message } from "@langchain/langgraph-sdk";

// 定义工具调用类型以匹配您的 Python 智能体
export type GetWeatherToolCall = {
  name: "get_weather";
  args: { location: string };
  id?: string;
};

export type AgentToolCalls = GetWeatherToolCall;

export interface AgentState {
  messages: Message<AgentToolCalls>[];
}
```

### 使用 `StateGraph`

对于自定义 [`StateGraph`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph) 应用程序，状态类型从图的标注中推断：

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from typing import TypedDict, Annotated

class State(TypedDict):
    messages: Annotated[list, add_messages]

model = ChatOpenAI(model="gpt-4o-mini")

async def agent(state: State) -> dict:
    response = await model.ainvoke(state["messages"])
    return {"messages": [response]}

workflow = StateGraph(State)
workflow.add_node("agent", agent)
workflow.add_edge(START, "agent")
workflow.add_edge("agent", END)

graph = workflow.compile()
```

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import type { GraphState } from "./types";

function Chat() {
  // 使用手动定义的状态类型
  const stream = useStream<GraphState>({
    assistantId: "my-graph",
    apiUrl: "http://localhost:2024",
  });

  // stream.values 根据您定义的状态类型化
}
```

```typescript
import type { Message } from "@langchain/langgraph-sdk";

// 定义状态以匹配您 Python 图的 State TypedDict
export interface GraphState {
  messages: Message[];
}
```

### 使用标注类型

如果您使用 LangGraph.js，可以重用图的标注类型。确保只导入类型以避免导入整个 LangGraph.js 运行时：

### 高级类型配置

您可以为中断、自定义事件和可配置选项指定其他类型参数：

## 渲染工具调用

使用 `getToolCalls` 从 AI 消息中提取和渲染工具调用。工具调用包括调用详情、结果（如果完成）和状态。

```python
from langchain import create_agent, tool

@tool
def get_weather(location: str) -> str:
    """获取当前位置的天气。"""
    return f'{{"status": "success", "content": "{location}天气: 晴天, 22°C"}}'

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_weather],
)
```

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import type { AgentState, AgentToolCalls } from "./types";
import { ToolCallCard } from "./ToolCallCard";
import { MessageBubble } from "./MessageBubble";

function Chat() {
  const stream = useStream<AgentState>({
    assistantId: "agent",
    apiUrl: "http://localhost:2024",
  });

  return (
    <div className="flex flex-col gap-4">
      {stream.messages.map((message, idx) => {
        if (message.type === "ai") {
          const toolCalls = stream.getToolCalls(message);

          if (toolCalls.length > 0) {
            return (
              <div key={message.id ?? idx} className="flex flex-col gap-2">
                {toolCalls.map((toolCall) => (
                  <ToolCallCard key={toolCall.id} toolCall={toolCall} />
                ))}
              </div>
            );
          }
        }

        return <MessageBubble key={message.id ?? idx} message={message} />;
      })}
    </div>
  );
}

export function ToolCallCard({
  toolCall,
}: {
  toolCall: ToolCallWithResult<AgentToolCalls>;
}) {
  const { call, result, state } = toolCall;

  if (call.name === "get_weather") {
    return <WeatherCard call={call} result={result} state={state} />;
  }

  return <GenericToolCallCard call={call} result={result} state={state} />;
}

export function WeatherCard({
  call,
  result,
  state,
}: {
  call: GetWeatherToolCall;
  result?: ToolMessage;
  state: ToolCallState;
}) {
  const isLoading = state === "pending";
  const parsedResult = parseToolResult(result);

  return (
    <div className="relative overflow-hidden rounded-xl">
      <div className="absolute inset-0 bg-gradient-to-br from-sky-600 to-indigo-600" />
      <div className="relative p-4">
        <div className="flex items-center gap-2 text-white/80 text-xs mb-3">
          <span className="font-medium">{call.args.location}</span>
          {isLoading && <span className="ml-auto">加载中...</span>}
        </div>
        {parsedResult.status === "error" ? (
          <div className="bg-red-500/20 rounded-lg p-3 text-red-200 text-sm">
            {parsedResult.content}
          </div>
        ) : (
          <div className="text-white text-lg font-medium">
            {parsedResult.content || "正在获取天气..."}
          </div>
        )}
      </div>
    </div>
  );
}
```

```typescript
import type { Message } from "@langchain/langgraph-sdk";

// 定义工具调用类型以匹配您的 Python 智能体的工具
export type GetWeatherToolCall = {
  name: "get_weather";
  args: { location: string };
  id?: string;
};

// 智能体中所有工具调用的联合类型
export type AgentToolCalls = GetWeatherToolCall;

// 使用您的工具调用定义状态类型
export interface AgentState {
  messages: Message<AgentToolCalls>[];
}

export function parseToolResult(result?: ToolMessage): {
  status: string;
  content: string;
} {
  if (!result) return { status: "pending", content: "" };
  try {
    return JSON.parse(result.content as string);
  } catch {
    return { status: "success", content: result.content as string };
  }
}
```

查看 `tool-calling-agent` 示例中带有天气、计算器和笔记工具的工具调用渲染的完整实现。

## 自定义流式传输事件

使用工具或节点中的 `writer` 从智能体流式传输自定义数据。使用 `onCustomEvent` 回调在 UI 中处理这些事件。

```python
import asyncio
import time
from langchain import create_agent, tool
from langchain.types import ToolRuntime

@tool
async def analyze_data(data_source: str, *, config: ToolRuntime) -> str:
    """使用进度更新分析数据。"""
    steps = ["连接中...", "获取中...", "处理中...", "完成!"]

    for i, step in enumerate(steps):
        # 在执行期间发出进度事件
        if config.writer:
            config.writer({
                "type": "progress",
                "id": f"analysis-{int(time.time() * 1000)}",
                "message": step,
                "progress": ((i + 1) / len(steps)) * 100,
            })
        await asyncio.sleep(0.5)

    return '{"result": "分析完成"}'

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[analyze_data],
)
```

```tsx
import { useState, useCallback } from "react";
import { useStream } from "@langchain/langgraph-sdk/react";
import type { AgentState } from "./types";

interface ProgressData {
  type: "progress";
  id: string;
  message: string;
  progress: number;
}

function isProgressData(data: unknown): data is ProgressData {
  return (
    typeof data === "object" &&
    data !== null &&
    "type" in data &&
    (data as ProgressData).type === "progress"
  );
}

function CustomStreamingUI() {
  const [progressData, setProgressData] = useState<Map<string, ProgressData>>(
    new Map()
  );

  const handleCustomEvent = useCallback((data: unknown) => {
    if (isProgressData(data)) {
      setProgressData((prev) => {
        const updated = new Map(prev);
        updated.set(data.id, data);
        return updated;
      });
    }
  }, []);

  const stream = useStream<AgentState>({
    assistantId: "custom-streaming",
    apiUrl: "http://localhost:2024",
    onCustomEvent: handleCustomEvent,
  });

  return (
    <div>
      {Array.from(progressData.values()).map((data) => (
        <div key={data.id} className="bg-neutral-800 rounded-lg p-4 mb-4">
          <div className="flex justify-between mb-2">
            <span className="text-sm text-white">{data.message}</span>
            <span className="text-xs text-neutral-400">{data.progress}%</span>
          </div>
          <div className="w-full bg-neutral-700 rounded-full h-2">
            <div
              className="bg-blue-500 h-2 rounded-full transition-all"
              style={{ width: `${data.progress}%` }}
            />
          </div>
        </div>
      ))}
    </div>
  );
}
```

```typescript
import type { Message } from "@langchain/langgraph-sdk";

// 定义工具调用以匹配您的 Python 智能体
export type AnalyzeDataToolCall = {
  name: "analyze_data";
  args: { data_source: string };
  id?: string;
};

export type AgentToolCalls = AnalyzeDataToolCall;

export interface AgentState {
  messages: Message<AgentToolCalls>[];
}
```

查看 `custom-streaming` 示例中带有进度条、状态徽章和文件操作卡的自定义事件的完整实现。

## 事件处理

`useStream` 钩子提供回调选项，使您可以访问不同类型的流式传输事件。您不需要显式配置流模式——只需为您要处理的事件类型传递回调：

### 可用回调

| 回调 | 描述 | 流模式 |
|------|------|--------|
| `onUpdateEvent` | 在每个图步骤之后收到状态更新时调用 | `updates` |
| `onCustomEvent` | 从图中收到自定义事件时调用 | `custom` |
| `onMetadataEvent` | 使用运行和线程元数据调用 | `metadata` |
| `onError` | 发生错误时调用 | - |
| `onFinish` | 流完成时调用 | - |

## 多智能体流式传输

当使用多智能体系统或具有多个节点的图时，使用消息元数据来识别每个消息是由哪个节点生成的。当多个 LLM 并行运行并且您想用独特的视觉样式显示它们的输出时，这特别有用。

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END, Send
from langgraph.graph.state import CompiledStateGraph
from langchain.messages import BaseMessage, AIMessage
from typing import TypedDict, Annotated
import operator

# 为多样性使用不同的模型实例
analytical_model = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)
creative_model = ChatOpenAI(model="gpt-4o-mini", temperature=0.9)
practical_model = ChatOpenAI(model="gpt-4o-mini", temperature=0.5)

class State(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    topic: str
    analytical_research: str
    creative_research: str
    practical_research: str

def fan_out_to_researchers(state: State) -> list[Send]:
    return [
        Send("researcher_analytical", state),
        Send("researcher_creative", state),
        Send("researcher_practical", state),
    ]

def dispatcher(state: State) -> dict:
    last_message = state["messages"][-1] if state["messages"] else None
    topic = last_message.content if last_message else ""
    return {"topic": topic}

async def researcher_analytical(state: State) -> dict:
    response = await analytical_model.ainvoke([
        {"role": "system", "content": "您是分析研究专家。"},
        {"role": "user", "content": f"研究: {state['topic']}"},
    ])
    return {
        "analytical_research": response.content,
        "messages": [AIMessage(content=response.content, name="researcher_analytical")],
    }

# 类似的研究者节点...

workflow = StateGraph(State)
workflow.add_node("dispatcher", dispatcher)
workflow.add_node("researcher_analytical", researcher_analytical)
workflow.add_node("researcher_creative", researcher_creative)
workflow.add_node("researcher_practical", researcher_practical)
workflow.add_edge(START, "dispatcher")
workflow.add_conditional_edges("dispatcher", fan_out_to_researchers)
workflow.add_edge("researcher_analytical", END)
workflow.add_edge("researcher_creative", END)
workflow.add_edge("researcher_practical", END)

agent: CompiledStateGraph = workflow.compile()
```

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import type { AgentState } from "./types";
import { MessageBubble } from "./MessageBubble";

// 节点配置用于视觉显示
const NODE_CONFIG: Record<string, { label: string; color: string }> = {
  researcher_analytical: { label: "分析研究", color: "cyan" },
  researcher_creative: { label: "创意研究", color: "purple" },
  researcher_practical: { label: "实用研究", color: "emerald" },
};

function MultiAgentChat() {
  const stream = useStream<AgentState>({
    assistantId: "parallel-research",
    apiUrl: "http://localhost:2024",
  });

  return (
    <div className="flex flex-col gap-4">
      {stream.messages.map((message, idx) => {
        if (message.type !== "ai") {
          return <MessageBubble key={message.id ?? idx} message={message} />;
        }

        const metadata = stream.getMessagesMetadata?.(message);
        const nodeName =
          (metadata?.streamMetadata?.langgraph_node as string) ||
          (message as { name?: string }).name;

        const config = nodeName ? NODE_CONFIG[nodeName] : null;

        if (!config) {
          return <MessageBubble key={message.id ?? idx} message={message} />;
        }

        return (
          <div
            key={message.id ?? idx}
            className={`bg-${config.color}-950/30 border border-${config.color}-500/30 rounded-xl p-4`}
          >
            <div className={`text-sm font-semibold text-${config.color}-400 mb-2`}>
              {config.label}
            </div>
            <div className="text-neutral-200 whitespace-pre-wrap">
              {typeof message.content === "string" ? message.content : ""}
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

```typescript
import type { Message } from "@langchain/langgraph-sdk";

// 状态与您 Python 智能体的 State TypedDict 匹配
export interface AgentState {
  messages: Message[];
  topic: string;
  analytical_research: string;
  creative_research: string;
  practical_research: string;
}
```

查看 `parallel-research` 示例中带有三个并行研究者和独特视觉样式的多智能体流式传输的完整实现。

## 人工介入

当智能体需要人工批准工具执行时处理中断。在[如何处理中断](/oss/python/langgraph/interrupts#pause-using-interrupt)指南中了解更多信息。

```python
from langchain import create_agent, tool, human_in_the_loop_middleware
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver

model = ChatOpenAI(model="gpt-4o-mini")

@tool
def send_email(to: str, subject: str, body: str) -> dict:
    """发送电子邮件。需要人工批准。"""
    return {
        "status": "success",
        "content": f'向 {to} 发送主题为 "{subject}" 的电子邮件',
    }

@tool
def delete_file(path: str) -> dict:
    """删除文件。需要人工批准。"""
    return {"status": "success", "content": f'文件 "{path}" 已删除'}

@tool
def read_file(path: str) -> dict:
    """读取文件内容。无需批准。"""
    return {"status": "success", "content": f"{path} 的内容..."}

agent = create_agent(
    model=model,
    tools=[send_email, delete_file, read_file],
    middleware=[
        human_in_the_loop_middleware(
            interrupt_on={
                "send_email": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                    "description": "📧 发送前检查电子邮件",
                },
                "delete_file": {
                    "allowed_decisions": ["approve", "reject"],
                    "description": "🗑️ 确认文件删除",
                },
                "read_file": False,  # 安全 - 自动批准
            }
        ),
    ],
    checkpointer=MemorySaver(),
)
```

```tsx
import { useState } from "react";
import { useStream } from "@langchain/langgraph-sdk/react";
import type { AgentState, HITLRequest, HITLResponse } from "./types";
import { MessageBubble } from "./MessageBubble";

function HumanInTheLoopChat() {
  const stream = useStream<AgentState, { InterruptType: HITLRequest }>({
    assistantId: "human-in-the-loop",
    apiUrl: "http://localhost:2024",
  });

  const [isProcessing, setIsProcessing] = useState(false);
  const hitlRequest = stream.interrupt?.value as HITLRequest | undefined;

  const handleApprove = async (index: number) => {
    if (!hitlRequest) return;
    setIsProcessing(true);

    try {
      const decisions: HITLResponse["decisions"] =
        hitlRequest.actionRequests.map((_, i) =>
          i === index ? { type: "approve" } : { type: "approve" }
        );

      await stream.submit(null, {
        command: { resume: { decisions } as HITLResponse },
      });
    } finally {
      setIsProcessing(false);
    }
  };

  const handleReject = async (index: number, reason: string) => {
    if (!hitlRequest) return;
    setIsProcessing(true);

    try {
      const decisions: HITLResponse["decisions"] =
        hitlRequest.actionRequests.map((_, i) =>
          i === index
            ? { type: "reject", message: reason }
            : { type: "reject", message: "与其他操作一起拒绝" }
        );

      await stream.submit(null, {
        command: { resume: { decisions } as HITLResponse },
      });
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <div>
      {stream.messages.map((message, idx) => (
        <MessageBubble key={message.id ?? idx} message={message} />
      ))}

      {hitlRequest && hitlRequest.actionRequests.length > 0 && (
        <div className="bg-amber-900/20 border border-amber-500/30 rounded-xl p-4 mt-4">
          <h3 className="text-amber-400 font-semibold mb-4">
            操作需要批准
          </h3>

          {hitlRequest.actionRequests.map((action, idx) => (
            <div key={idx} className="bg-neutral-900 rounded-lg p-4 mb-4 last:mb-0">
              <div className="text-sm font-mono text-white mb-2">{action.name}</div>
              <pre className="text-xs bg-black rounded p-2 mb-3 overflow-x-auto">
                {JSON.stringify(action.args, null, 2)}
              </pre>
              <div className="flex gap-2">
                <button
                  onClick={() => handleApprove(idx)}
                  disabled={isProcessing}
                  className="px-3 py-1.5 bg-green-600 hover:bg-green-500 text-white text-sm rounded-lg"
                >
                  批准
                </button>
                <button
                  onClick={() => handleReject(idx, "用户拒绝")}
                  disabled={isProcessing}
                  className="px-3 py-1.5 bg-red-600 hover:bg-red-500 text-white text-sm rounded-lg"
                >
                  拒绝
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

```typescript
import type { Message } from "@langchain/langgraph-sdk";

// 工具调用类型匹配您的 Python 智能体
export type SendEmailToolCall = {
  name: "send_email";
  args: { to: string; subject: string; body: string };
  id?: string;
};

export type DeleteFileToolCall = {
  name: "delete_file";
  args: { path: string };
  id?: string;
};

export type ReadFileToolCall = {
  name: "read_file";
  args: { path: string };
  id?: string;
};

export type AgentToolCalls = SendEmailToolCall | DeleteFileToolCall | ReadFileToolCall;

export interface AgentState {
  messages: Message<AgentToolCalls>[];
}

// HITL 类型
export interface HITLRequest {
  actionRequests: Array<{
    name: string;
    args: Record<string, unknown>;
  }>;
}

export interface HITLResponse {
  decisions: Array<
    | { type: "approve" }
    | { type: "reject"; message: string }
    | { type: "edit"; newArgs: Record<string, unknown> }
  >;
}
```

查看 `human-in-the-loop` 示例中带有批准、拒绝和编辑操作的工作流批准的完整实现。

## 推理模型

<Warning>
  扩展推理/思考支持目前处于实验状态。推理令牌的流式传输接口因提供商而异（OpenAI vs. Anthropic），并且可能随着抽象的发展而变化。
</Warning>

当使用具有扩展推理能力的模型（如 OpenAI 的推理模型或 Anthropic 的扩展思考）时，思考过程嵌入在消息内容中。您需要单独提取和显示它。

```python
from langchain import create_agent
from langchain_openai import ChatOpenAI

# 使用支持推理的模型
# 对于 OpenAI: o1, o1-mini, o1-preview
# 对于 Anthropic: claude-sonnet-4-20250514 启用扩展思考
model = ChatOpenAI(model="o1-mini")

agent = create_agent(
    model=model,
    tools=[],  # 推理模型最适合复杂的推理任务
)
```

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import type { AgentState } from "./types";
import { getReasoningFromMessage, getTextContent } from "./utils";
import { MessageBubble } from "./MessageBubble";

function ReasoningChat() {
  const stream = useStream<AgentState>({
    assistantId: "reasoning-agent",
    apiUrl: "http://localhost:2024",
  });

  return (
    <div className="flex flex-col gap-4">
      {stream.messages.map((message, idx) => {
        if (message.type === "ai") {
          const reasoning = getReasoningFromMessage(message);
          const textContent = getTextContent(message);

          return (
            <div key={message.id ?? idx}>
              {reasoning && (
                <div className="mb-4">
                  <div className="text-xs font-medium text-amber-400/80 mb-2">
                    推理
                  </div>
                  <div className="bg-amber-950/50 border border-amber-500/20 rounded-2xl px-4 py-3">
                    <div className="text-sm text-amber-100/90 whitespace-pre-wrap">
                      {reasoning}
                    </div>
                  </div>
                </div>
              )}

              {textContent && (
                <div className="text-neutral-100 whitespace-pre-wrap">
                  {textContent}
                </div>
              )}
            </div>
          );
        }

        return <MessageBubble key={message.id ?? idx} message={message} />;
      })}

      {stream.isLoading && (
        <div className="flex items-center gap-2 text-amber-400/70">
          <span className="text-sm">思考中...</span>
        </div>
      )}
    </div>
  );
}
```

```typescript
import type { Message } from "@langchain/langgraph-sdk";

export interface AgentState {
  messages: Message[];
}
```

```typescript
import type { Message, AIMessage } from "@langchain/langgraph-sdk";

/**
 * 从 AI 消息中提取推理/思考内容。
 * 支持 OpenAI 推理和 Anthropic 扩展思考。
 */
export function getReasoningFromMessage(message: Message): string | undefined {
  type MessageWithExtras = AIMessage & {
    additional_kwargs?: {
      reasoning?: {
        summary?: Array<{ type: string; text: string }>;
      };
    };
    contentBlocks?: Array<{ type: string; thinking?: string }>;
  };

  const msg = message as MessageWithExtras;

  // 在 additional_kwargs 中检查 OpenAI 推理
  if (msg.additional_kwargs?.reasoning?.summary) {
    const content = msg.additional_kwargs.reasoning.summary
      .filter((item) => item.type === "summary_text")
      .map((item) => item.text)
      .join("");
    if (content.trim()) return content;
  }

  // 在 contentBlocks 中检查 Anthropic 思考
  if (msg.contentBlocks?.length) {
    const thinking = msg.contentBlocks
      .filter((b) => b.type === "thinking" && b.thinking)
      .map((b) => b.thinking)
      .join("\n");
    if (thinking) return thinking;
  }

  // 在 message.content 数组中检查思考
  if (Array.isArray(msg.content)) {
    const thinking = msg.content
      .filter((b): b is { type: "thinking"; thinking: string } =>
        typeof b === "object" && b?.type === "thinking" && "thinking" in b
      )
      .map((b) => b.thinking)
      .join("\n");
    if (thinking) return thinking;
  }

  return undefined;
}

/**
 * 从消息中提取文本内容。
 */
export function getTextContent(message: Message): string {
  if (typeof message.content === "string") return message.content;
  if (Array.isArray(message.content)) {
    return message.content
      .filter((c): c is { type: "text"; text: string } => c.type === "text")
      .map((c) => c.text)
      .join("");
  }
  return "";
}
```

查看 `reasoning-agent` 示例中带有 OpenAI 和 Anthropic 模型的推理令牌显示的完整实现。

## 自定义状态类型

对于自定义 LangGraph 应用程序，在状态的消息属性中嵌入您的工具调用类型。

## 自定义传输

对于自定义 API 端点或非标准部署，将 `transport` 选项与 `FetchStreamTransport` 一起使用以连接到任何流式传输 API。

## 相关内容

* [流式传输概述](overview.md) — 使用 LangChain 智能体的服务器端流式传输
* [useStream API 参考](https://reference.langchain.com/javascript/functions/_langchain_langgraph-sdk.react.useStream.html) — 完整 API 文档
* [智能体聊天 UI](https://python.langchain.com/ui) — LangChain 智能体的预构建聊天界面
* [人工介入](../advanced-usage/human-in-the-loop.md) — 配置人工审查的中断
* [多智能体系统](../advanced-usage/multi-agent/overview.md) — 使用多个 LLM 构建智能体

