# 第 4 章 @earendil-works/pi-agent-core：Agent 运行时引擎

## 4.1 从 LLM 到 Agent 的飞跃

`@earendil-works/pi-ai` 只解决了"如何调用 LLM"的问题。但一个真正的 AI 编程代理需要做得更多：

1. **多轮对话**：管理对话历史，在多次 LLM 调用之间保持上下文
2. **工具调用**：解析 LLM 发出的工具调用请求，执行工具，将结果返回给 LLM
3. **循环**：LLM 回复 → 工具调用 → 工具结果 → LLM 再次回复 → … 直到完成任务
4. **消息队列**：支持在运行中注入消息（Steer）或在完成后发送消息（FollowUp）
5. **Hook 链**：在工具调用前后插入自定义逻辑（权限检查、结果修改等）
6. **事件**：将 Agent 内部状态以事件形式发布，供 UI 或日志系统消费

这就是 `@earendil-works/pi-agent-core` 要做的事情。它建立在 `pi-ai` 之上，提供了一个完整的、可扩展的 Agent 运行时。

## 4.2 架构概览

pi-agent-core 只有 **7 个源文件**，却实现了完整的 Agent 运行时：

```
src/
├── index.ts          # 统一导出
├── types.ts          # 核心类型：AgentEvent, AgentTool, AgentState, QueueMode...
├── agent.ts          # Agent 类（状态机 + 队列 + 事件系统）
├── agent-loop.ts     # Agent Loop（LLM 调用 → 工具执行 循环逻辑）
├── proxy.ts          # 代理工具
├── node.ts           # Node.js 特定功能
└── harness/          # 高级封装（Agent Harness）
    ├── types.ts
    ├── agent-harness.ts
    ├── messages.ts
    ├── prompt-templates.ts
    ├── system-prompt.ts
    ├── skills.ts
    ├── compaction/
    │   ├── compaction.ts
    │   └── branch-summarization.ts
    └── session/
        ├── session.ts
        ├── memory-repo.ts
        ├── jsonl-repo.ts
        └── repo-utils.ts
```

## 4.3 Agent 状态机

`Agent` 类封装了 Agent 的所有状态和行为。它的状态机设计如下：

```
                    ┌─────────┐
                    │  Idle   │
                    └────┬────┘
                         │ prompt() / continue()
                         ▼
                    ┌──────────┐
                    │ Streaming│  ← isStreaming = true
                    └────┬─────┘
                         │ Agent Loop 完成
                         ▼
                    ┌─────────┐
                    │  Idle   │  ← isStreaming = false
                    └─────────┘
```

状态变量全部包含在 `AgentState` 接口中：

```typescript
export interface AgentState {
    systemPrompt: string;              // 系统提示词
    model: Model<any>;                 // 当前使用的模型
    thinkingLevel: ThinkingLevel;       // 推理级别
    set tools(tools: AgentTool<any>[]); // 可用工具
    set messages(messages: AgentMessage[]); // 对话历史
    readonly isStreaming: boolean;      // 是否正在运行
    readonly streamingMessage?: AgentMessage; // 当前流式消息
    readonly pendingToolCalls: ReadonlySet<string>; // 正在执行的工具调用
    readonly errorMessage?: string;     // 最近的错误消息
}
```

### 4.3.1 核心方法

```typescript
export class Agent {
    // 提交一个新的 Prompt
    async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]): Promise<void>;

    // 从当前对话继续（最后一条消息必须是 user 或 toolResult）
    async continue(): Promise<void>;

    // 注册事件监听
    subscribe(listener: (event: AgentEvent) => void): () => void;

    // 注入消息（在当前工具执行完成后、下一轮 LLM 调用之前）
    steer(message: AgentMessage): void;

    // 追加消息（在当前轮次全部完成后）
    followUp(message: AgentMessage): void;

    // 中断当前运行
    abort(): void;

    // 等待当前运行结束
    waitForIdle(): Promise<void>;

    // 重置所有状态
    reset(): void;
}
```

### 4.3.2 消息队列

Agent 实现了两级队列：**Steer** 和 **FollowUp**，每个队列都有两种模式：

```typescript
// 队列模式
type QueueMode = "all" | "one-at-a-time";

class PendingMessageQueue {
    public mode: QueueMode;
    private messages: AgentMessage[] = [];

    enqueue(message: AgentMessage): void;
    drain(): AgentMessage[] {
        if (this.mode === "all") {
            const drained = this.messages.slice();
            this.messages = [];
            return drained;
        }
        // "one-at-a-time": 只弹出最早的一条
        const first = this.messages[0];
        this.messages = this.messages.slice(1);
        return [first];
    }
}
```

**Steer vs FollowUp 的区别**：

```
Steer 队列：
  助理回复 → 工具执行 → [检查 Steer 队列] → 有消息则立即开始新一轮
                                                      ↓
  无消息 → 检查 FollowUp 队列 → 有消息则开始 → 否则结束

FollowUp 队列：
  只在 Agent 即将停止时检查（没有更多工具调用、没有 Steer 消息）
```

这个设计让扩展系统可以在 Agent 运行时"插入"消息——比如用户正在输入新提示时，Agent 可以继续处理当前工具调用，然后处理用户的新输入。

## 4.4 Agent Loop：智能体的核心循环

Agent Loop 位于 `agent-loop.ts` 中，是整个 pi 项目最关键的算法。它的结构如下：

```
runAgentLoop(prompts, context, config, emit, signal)
    │
    ├── emit agent_start
    │
    └── runLoop(currentContext, newMessages, config, signal, emit)
         │
         ├── 外层循环 (while true):
         │    │
         │    ├── 内层循环:
         │    │    │
         │    │    ├── 1. 处理挂起消息（Steer / 初始 Prompt）
         │    │    │
         │    │    ├── 2. streamAssistantResponse():
         │    │    │    ├── transformContext() → 裁剪上下文
         │    │    │    ├── convertToLlm() → AgentMessage → Message
         │    │    │    ├── streamSimple() → 调用 LLM
         │    │    │    ├── 流式接收事件 → emit message_update
         │    │    │    └── 返回 AssistantMessage
         │    │    │
         │    │    ├── 3. 检查 stopReason:
         │    │    │    ├── "error"/"aborted" → 结束
         │    │    │    └── 其他 → 继续
         │    │    │
         │    │    ├── 4. 检查 Tool Calls:
         │    │    │    ├── 有 → executeToolCalls()
         │    │    │    │    ├── 并行执行（默认）
         │    │    │    │    └── 顺序执行（回退）
         │    │    │    └── 无 → hasMoreToolCalls = false
         │    │    │
         │    │    ├── 5. emit turn_end
         │    │    │
         │    │    ├── 6. prepareNextTurn() → 更新上下文/模型
         │    │    │
         │    │    ├── 7. shouldStopAfterTurn() → 如果返回 true, 结束
         │    │    │
         │    │    └── 8. 检查 Steer 队列 → 继续内层循环
         │    │
         │    ├── 检查 FollowUp 队列:
         │    │    ├── 有消息 → 设为 pending → 继续外层循环
         │    │    └── 无消息 → 退出外层循环
         │    │
         │    └── emit agent_end
         │
         └── 返回 newMessages[]
```

### 4.4.1 关键的流式消息处理

`streamAssistantResponse()` 是 LLM 调用与事件系统的桥梁。它通过 `for await...of` 消费 `EventStream`，并将事件转换为 `AgentEvent`：

```typescript
async function streamAssistantResponse(context, config, signal, emit, streamFn) {
    // 1. 转换消息
    let messages = context.messages;
    if (config.transformContext) {
        messages = await config.transformContext(messages, signal);
    }
    const llmMessages = await config.convertToLlm(messages);

    // 2. 构建 LLM 调用上下文
    const llmContext: Context = {
        systemPrompt: context.systemPrompt,
        messages: llmMessages,
        tools: context.tools,
    };

    // 3. 调用 LLM（可替换的 streamFn）
    const response = await streamFn(config.model, llmContext, { ...config, signal });

    // 4. 流式处理事件
    let partialMessage: AssistantMessage | null = null;
    for await (const event of response) {
        switch (event.type) {
            case "start":
                partialMessage = event.partial;
                context.messages.push(partialMessage);
                await emit({ type: "message_start", message: partialMessage });
                break;

            case "text_delta":
            case "thinking_delta":
            case "toolcall_delta":
                // 更新消息内容并发射 update 事件
                partialMessage = event.partial;
                context.messages[context.messages.length - 1] = partialMessage;
                await emit({ type: "message_update", message: partialMessage, assistantMessageEvent: event });
                break;

            case "done":
            case "error":
                // 返回最终消息
                return finalMessage;
        }
    }
}
```

### 4.4.2 工具执行引擎

工具执行支持两种模式：

#### 并行模式（默认）

```typescript
async function executeToolCallsParallel(currentContext, assistantMessage, toolCalls, config, signal, emit) {
    // 阶段 1：准备所有工具调用（顺序执行，可以阻塞）
    for (const toolCall of toolCalls) {
        await emit({ type: "tool_execution_start", ... });
        const preparation = await prepareToolCall(...);
        // 如果被 beforeToolCall 阻止了，立即发出结果
        if (preparation.kind === "immediate") {
            finalizedCalls.push(finalized);
            continue;
        }
        // 否则加入并行执行队列
        finalizedCalls.push(async () => {
            const executed = await executePreparedToolCall(preparation, ...);
            return await finalizeExecutedToolCall(preparation, executed, ...);
        });
    }

    // 阶段 2：并行执行所有准备好的工具
    const orderedFinalizedCalls = await Promise.all(
        finalizedCalls.map(entry => typeof entry === "function" ? entry() : entry)
    );

    // 阶段 3：按原始顺序生成 ToolResultMessage
    const messages = orderedFinalizedCalls.map(createToolResultMessage);
    return { messages, terminate: shouldTerminateToolBatch(orderedFinalizedCalls) };
}
```

#### 顺序模式

```typescript
async function executeToolCallsSequential(...) {
    for (const toolCall of toolCalls) {
        // 逐个执行，等待每个完成后再开始下一个
        const preparation = await prepareToolCall(...);
        if (preparation.kind === "prepared") {
            const executed = await executePreparedToolCall(preparation, ...);
            finalized = await finalizeExecutedToolCall(preparation, executed, ...);
        }
        // 发射结果
        await emitToolExecutionEnd(finalized, emit);
        messages.push(createToolResultMessage(finalized));
    }
    return { messages, terminate: shouldTerminateToolBatch(finalizedCalls) };
}
```

**何时使用并行/顺序？**
- **默认并行**：LLM 可能会同时发出多个独立的工具调用（如同时 `read` 多个文件）
- **含顺序标记的工具**：如果某个工具设置了 `executionMode: "sequential"`（如 `edit` 操作），所有工具都会回退到顺序模式
- **通过配置强制顺序**：`config.toolExecution = "sequential"`

### 4.4.3 Hook 链：BeforeToolCall / AfterToolCall

Agent Loop 在工具执行前后提供了钩子（Hook）：

```typescript
interface AgentLoopConfig {
    // 工具执行前调用（可以阻止执行）
    beforeToolCall?: (
        context: BeforeToolCallContext,  // assistantMessage, toolCall, args
        signal?: AbortSignal,
    ) => Promise<BeforeToolCallResult | undefined>;
    // 返回 { block: true, reason: "..." } 阻止工具执行

    // 工具执行后调用（可以修改结果）
    afterToolCall?: (
        context: AfterToolCallContext,  // assistantMessage, toolCall, args, result
        signal?: AbortSignal,
    ) => Promise<AfterToolCallResult | undefined>;
    // 返回 { content: [...], isError: true, terminate: true } 覆盖结果

    // 每个 turn 结束后调用（决定是否继续）
    shouldStopAfterTurn?: (
        context: ShouldStopAfterTurnContext,
    ) => boolean | Promise<boolean>;

    // 每个 turn 结束后调用（可以更换模型/上下文）
    prepareNextTurn?: (
        context: PrepareNextTurnContext,
    ) => AgentLoopTurnUpdate | undefined | Promise<AgentLoopTurnUpdate | undefined>;
}
```

这些 Hook 是 pi 扩展系统的核心机制之一，扩展可以通过它们实现权限控制、结果修改、上下文注入等。

## 4.5 事件系统

Agent 运行时发出的事件构成了与 UI 通信的完整协议：

```
Agent 事件时间线:

agent_start
  │
  ├── turn_start
  │     │
  │     ├── message_start (用户消息)
  │     ├── message_end   (用户消息)
  │     │
  │     ├── message_start (助理消息，流式开始)
  │     ├── message_update (助理消息，流式更新)  ← 高频，每段文本/思考/工具调用增量
  │     ├── message_end   (助理消息，流式结束)
  │     │
  │     ├── tool_execution_start (工具#1)
  │     ├── tool_execution_update (工具#1 进度更新)
  │     ├── tool_execution_end   (工具#1)
  │     ├── tool_execution_start (工具#2)
  │     ├── tool_execution_end   (工具#2)
  │     │
  │     └── turn_end
  │
  ├── (下一轮 turn_start...)
  │
  └── agent_end
```

`AgentEvent` 类型的完整定义：

```typescript
export type AgentEvent =
    // Agent 生命周期
    | { type: "agent_start" }
    | { type: "agent_end"; messages: AgentMessage[] }

    // Turn 生命周期
    | { type: "turn_start" }
    | { type: "turn_end"; message: AssistantMessage; toolResults: ToolResultMessage[] }

    // 消息生命周期
    | { type: "message_start"; message: AgentMessage }
    | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
    | { type: "message_end"; message: AgentMessage }

    // 工具执行生命周期
    | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
    | { type: "tool_execution_update"; toolCallId: string; toolName: string; partialResult: any }
    | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

## 4.6 AgentHarness：高级封装

pi-agent-core 还提供了一个更高级的封装 `AgentHarness`，位于 `src/harness/` 下。它为 Agent 添加了以下功能：

1. **会话管理**（`Session`）— 持久化对话历史到 JSONL 文件
2. **技能系统**（`Skills`）— 加载 SKILL.md 文件并注入到系统提示中
3. **Prompt 模板**（`PromptTemplates`）— 可复用的提示模板
4. **上下文压缩**（`Compaction`）— 摘要历史消息以控制上下文窗口
5. **分支摘要**（`BranchSummarization`）— 为会话分支生成摘要

### 上下文压缩（Compaction）

当对话上下文超过模型的上下文窗口时，AgentHarness 可以对历史进行压缩：

```typescript
async function compact(session: Session, settings: CompactionSettings) {
    // 1. 检查是否需要压缩
    if (!shouldCompact(session, settings)) return;

    // 2. 生成摘要
    const summary = await generateSummary(session, model);

    // 3. 查找切割点
    const cutPoint = findCutPoint(session, settings);

    // 4. 序列化压缩后的对话
    const compacted = serializeConversation(session, cutPoint);

    // 5. 写入压缩条目
    session.addEntry({
        type: "compaction",
        summary,
        tokensBefore: estimateTokens(session),
        // ...
    });
}
```

## 4.7 StreamFn：可替换的 LLM 调用函数

Agent 运行时不直接调用 pi-ai 的 `streamSimple`，而是通过一个可配置的 `streamFn`：

```typescript
// types.ts
export type StreamFn = (
    ...args: Parameters<typeof streamSimple>
) => ReturnType<typeof streamSimple> | Promise<ReturnType<typeof streamSimple>>;
```

这允许上层应用替换 LLM 调用行为。在 `Agent` 构造函数中，默认使用 `streamSimple`：

```typescript
class Agent {
    public streamFn: StreamFn;

    constructor(options: AgentOptions = {}) {
        this.streamFn = options.streamFn ?? streamSimple;
        // ...
    }
}
```

通过这个设计，你可以：
- 在测试中使用模拟 LLM 代替真实 API
- 添加请求/响应日志
- 实现重试逻辑
- 修改请求参数

## 4.8 章节小结

pi-agent-core 的设计展示了如何构建一个健壮且可扩展的 AI Agent 运行时：

1. **状态机设计**：清晰的 Idle/Streaming 状态，支持 prompt/continue/abort 操作
2. **两级消息队列**：Steer（中途插入）和 FollowUp（会后触发）的区分
3. **Agent Loop 算法**：外层（FollowUp 驱动）和内层（Tool Call 驱动）的双层循环
4. **工具执行引擎**：并行执行（默认）和顺序执行（按需）两种模式
5. **Hook 链**：4 个关键点的可扩展 Hook
6. **事件系统**：完整的 Agent 生命周期事件，从 `agent_start` 到 `agent_end`
7. **可替换的 LLM 调用**：通过 `StreamFn` 抽象
8. **高级封装**：AgentHarness 提供了会话管理、技能、压缩等生产级功能

Agent Loop 的代码虽然只有 400 多行，但由于其高度递归和事件驱动的性质，是所有 pi 源代码中最需要仔细阅读的部分之一。
