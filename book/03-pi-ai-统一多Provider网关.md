# 第 3 章 @earendil-works/pi-ai：统一多 Provider LLM 网关

## 3.1 设计目标

pi-ai 要解决的核心问题是：**如何用一个统一的接口对接所有 LLM Provider？**

每个 LLM Provider 都有自己的一套 API：

| Provider | API 风格 | 流式方式 | 消息格式 |
|----------|---------|---------|---------|
| OpenAI | ChatGPT Completions API | Server-Sent Events (SSE) | system/user/assistant/tool |
| OpenAI (new) | Responses API | SSE | developer/user/assistant/tool |
| Anthropic | Messages API | SSE | user/assistant (含 system 参数) |
| Google | Generative AI API | SSE | user/model (含 system_instruction) |
| AWS Bedrock | Converse Stream API | WebSocket-like | user/assistant |
| Mistral | Conversations API | SSE | system/user/assistant/tool |

pi-ai 的目标就是将这些完全不同的 API 统一为**一个通用的流式事件协议**（`AssistantMessageEvent`），让上层代码（Agent 运行时、UI）只需处理一种消息格式。

## 3.2 类型体系

pi-ai 的类型系统是其架构的基石，也是整个 pi 项目最值得仔细阅读的部分。核心类型位于 `src/types.ts`。

### 3.2.1 Provider 与 API 的区分

pi-ai 对 **Provider**（服务商）和 **API**（接口协议）做了清晰区分：

```typescript
// Provider = 服务提供商
export type KnownProvider =
    | "openai" | "anthropic" | "google" | "deepseek"
    | "github-copilot" | "openrouter" | "mistral"
    | "amazon-bedrock" | "fireworks" | "together"
    | "xai" | "groq" | "cerebras" | "minimax" | "moonshotai"
    | ...; // 支持 30+ 个 Provider

// API = 接口协议
export type KnownApi =
    | "openai-completions"   // 旧版 OpenAI Completions API
    | "openai-responses"     // 新版 OpenAI Responses API
    | "openai-codex-responses" // OpenAI Codex Responses API
    | "anthropic-messages"   // Anthropic Messages API
    | "bedrock-converse-stream" // AWS Bedrock
    | "google-generative-ai" // Google Gemini
    | "google-vertex"        // Google Vertex AI
    | "mistral-conversations"; // Mistral API
```

**为什么需要区分 Provider 和 API？**
- 同一个 Provider 可能支持多个 API（例如 OpenAI 有 Completions 和 Responses 两种 API）
- 同一个 API 协议可能被多个 Provider 复用（例如 DeepSeek 使用 OpenAI-compatible API）
- Provider 决定了定价、模型列表、认证方式；API 决定了请求格式和协议细节

这种设计在 `Model` 类型中得到体现：

```typescript
export interface Model<TApi extends Api> {
    id: string;           // 模型 ID，如 "gpt-4o"
    name: string;         // 人类可读的名称
    api: TApi;            // API 协议类型
    provider: Provider;   // 服务商
    baseUrl: string;      // API 端点 URL
    reasoning: boolean;   // 是否支持推理/思考
    thinkingLevelMap?: ThinkingLevelMap; // 思考级别映射
    input: ("text" | "image")[];  // 支持的输入类型
    cost: {               // 成本（$/百万 tokens）
        input: number;
        output: number;
        cacheRead: number;
        cacheWrite: number;
    };
    contextWindow: number;
    maxTokens: number;
    headers?: Record<string, string>; // 自定义 HTTP 头
    compat?: ...;  // API 兼容性配置
}
```

### 3.2.2 消息类型体系

pi-ai 定义了 4 种消息角色和 3 种内容块类型：

**消息角色**：
```typescript
// 用户消息
interface UserMessage { role: "user"; content: string | ContentBlock[]; timestamp: number; }

// 助手消息（LLM 回复）
interface AssistantMessage {
    role: "assistant";
    content: (TextContent | ThinkingContent | ToolCall)[];
    api: Api; provider: Provider; model: string;
    responseModel?: string;
    responseId?: string;
    usage: Usage;
    stopReason: StopReason;  // "stop" | "length" | "toolUse" | "error" | "aborted"
    errorMessage?: string;
    timestamp: number;
}

// 工具结果消息
interface ToolResultMessage {
    role: "toolResult";
    toolCallId: string; toolName: string;
    content: (TextContent | ImageContent)[];
    isError: boolean;
    timestamp: number;
}
```

**内容块类型**：
```typescript
type TextContent = { type: "text"; text: string; textSignature?: string; };
type ThinkingContent = { type: "thinking"; thinking: string; thinkingSignature?: string; redacted?: boolean; };
type ImageContent = { type: "image"; data: string; mimeType: string; };
type ToolCall = { type: "toolCall"; id: string; name: string; arguments: Record<string, any>; };
```

### 3.2.3 流式事件协议

这是 pi-ai 与上层通信的核心协议。每个事件都包含一个 `partial` 字段，表示截至当前时刻的完整消息状态：

```typescript
export type AssistantMessageEvent =
    // 消息开始
    | { type: "start"; partial: AssistantMessage }

    // 文本内容
    | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }

    // 思考/推理内容
    | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }

    // 工具调用
    | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }

    // 结束
    | { type: "done"; reason: "stop"|"length"|"toolUse"; message: AssistantMessage }
    | { type: "error"; reason: "aborted"|"error"; error: AssistantMessage };
```

**设计要点**：
- `contentIndex` 字段标识这是消息内容列表中的第几个块，让 UI 可以独立更新每个块的渲染
- `partial` 字段包含整个消息的当前状态，方便 UI 做整体刷新
- `start` 事件在第一个内容到来之前发出，让 UI 立即显示 Loading 状态
- `done` / `error` 事件携带最终完整消息，可用于一次性获取

## 3.3 API Registry：插件式 Provider 注册机制

API Registry 是 pi-ai 的核心调度层，支持运行时的 Provider 动态注册和查找。

### 3.3.1 注册与查找

```typescript
// src/api-registry.ts

// 注册表
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

// 注册 Provider
export function registerApiProvider<TApi extends Api, TOptions extends StreamOptions>(
    provider: ApiProvider<TApi, TOptions>,
    sourceId?: string,  // 用于批量卸载
): void {
    apiProviderRegistry.set(provider.api, {
        provider: {
            api: provider.api,
            stream: wrapStream(provider.api, provider.stream),
            streamSimple: wrapStreamSimple(provider.api, provider.streamSimple),
        },
        sourceId,
    });
}

// 查找 Provider
export function getApiProvider(api: Api): ApiProviderInternal | undefined {
    return apiProviderRegistry.get(api)?.provider;
}

// 批量卸载（用于扩展系统的动态 Provider）
export function unregisterApiProviders(sourceId: string): void {
    for (const [api, entry] of apiProviderRegistry.entries()) {
        if (entry.sourceId === sourceId) {
            apiProviderRegistry.delete(api);
        }
    }
}
```

### 3.3.2 Provider 接口

每个 Provider 需要实现两个函数：

```typescript
export interface ApiProvider<TApi extends Api, TOptions extends StreamOptions> {
    api: TApi;
    stream: StreamFunction<TApi, TOptions>;
    streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}
```

其中 `stream()` 是底层 API，接收 Provider 特定的选项；`streamSimple()` 是简化接口，统一接收 `SimpleStreamOptions`（只包含 `thinkingLevel` 和 `thinkingBudgets` 等通用选项）。

### 3.3.3 stream() 函数签名

```typescript
type StreamFunction<TApi extends Api, TOptions extends StreamOptions> = (
    model: Model<TApi>,
    context: Context,
    options?: TOptions,
) => AssistantMessageEventStream;
```

**关键契约**：
- 必须返回 `AssistantMessageEventStream`（一个 `EventStream<AssistantMessageEvent, AssistantMessage>`）
- 绝不抛出异常 — 错误被编码为 `type: "error"` 的流事件
- 异步操作通过 `EventStream` 的 `push()` 方法逐步推送

## 3.4 EventStream 实现

`EventStream` 是 pi-ai 中连接 Provider → Agent → UI 的核心数据结构。它实现了 `AsyncIterable` 接口，既可以被 `for await...of` 消费，也提供了 `result()` Promise 用于获取最终结果。

```typescript
// src/utils/event-stream.ts
export class EventStream<T, R = T> implements AsyncIterable<T> {
    private queue: T[] = [];
    private waiting: ((value: IteratorResult<T>) => void)[] = [];
    private done = false;
    private finalResultPromise: Promise<R>;

    constructor(
        isComplete: (event: T) => boolean,
        extractResult: (event: T) => R,
    ) { ... }

    push(event: T): void {
        if (this.done) return;
        if (this.isComplete(event)) {
            this.done = true;
            this.resolveFinalResult(this.extractResult(event));
        }
        // 优先推送给等待的消费者，否则入队
        const waiter = this.waiting.shift();
        if (waiter) { waiter({ value: event, done: false }); }
        else { this.queue.push(event); }
    }

    async *[Symbol.asyncIterator](): AsyncIterator<T> {
        // 有可用数据时弹出，否则等待 push()
    }

    result(): Promise<R> { return this.finalResultPromise; }
}

// 专用于 LLM 流的子类
export class AssistantMessageEventStream
    extends EventStream<AssistantMessageEvent, AssistantMessage> {
    constructor() {
        super(
            (event) => event.type === "done" || event.type === "error",
            (event) => event.type === "done" ? event.message : event.error,
        );
    }
}
```

**设计亮点**：
1. **双通道读取**：`for await...of` 用于流式消费（UI 逐帧更新），`result()` 用于一次性获取（Agent 获取最终消息）
2. **背压处理**：如果消费者消费速度慢于推送速度，事件会入队；如果推送速度慢，消费者会等待
3. **事件即终态**：`push(done)` 会立即触发 `resolveFinalResult`，两条读取路径一致

## 3.5 Provider 实现解剖：以 OpenAI Completions 为例

### 3.5.1 整体模式

所有 Provider 的实现遵循相同的模板：

```
AsyncIIFE pattern:
streamOpenAICompletions(model, context, options) {
    const stream = new AssistantMessageEventStream();

    (async () => {
        try {
            // 1. 获取 API Key
            // 2. 创建 HTTP 客户端
            // 3. 转换消息格式 (transformMessages)
            // 4. 构建请求参数
            // 5. 发送请求
            // 6. 解析响应流
            // 7. 发射事件：start → {text|thinking|toolcall}_{start|delta|end} → done
        } catch (error) {
            stream.push({ type: "error", ... });
        }
    })();

    return stream;
}
```

### 3.5.2 消息转换（transformMessages）

这是 Provider 实现中最核心的部分之一。pi-ai 定义了统一的 `Message[]` 类型，但每个 Provider 需要自己的消息格式。`transformMessages` 函数负责转换：

```typescript
// 在 openai-completions.ts 中
import { transformMessages } from "./transform-messages.ts";

// transformMessages 将统一的 Message[] 转换为 Provider 特定的格式
const llmMessages = transformMessages(context.messages, model, compat);
// 返回 ChatCompletionMessageParam[] （OpenAI SDK 的格式）
```

### 3.5.3 流式响应的解析

以 OpenAI Completions 为例，流式响应的解析核心是处理 `chat.completions.create()` 返回的 SSE 流：

```typescript
for await (const chunk of openaiStream) {
    // 每个 chunk 是 ChatCompletionChunk 类型
    const choice = chunk.choices?.[0];

    if (choice?.delta?.content) {
        // 文本增量
        const block = ensureTextBlock();
        block.text += choice.delta.content;
        stream.push({
            type: "text_delta",
            contentIndex: getContentIndex(block),
            delta: choice.delta.content,
            partial: output,
        });
    }

    if (choice?.delta?.tool_calls) {
        // 工具调用增量（流式工具调用，Tool Calling Streaming）
        for (const toolCall of choice.delta.tool_calls) {
            const block = ensureToolCallBlock(toolCall);
            block.partialArgs += toolCall.function?.arguments;
            block.arguments = parseStreamingJson(block.partialArgs);
            stream.push({
                type: "toolcall_delta",
                contentIndex: getContentIndex(block),
                delta: toolCall.function?.arguments,
                partial: output,
            });
        }
    }

    if (choice?.finish_reason) {
        output.stopReason = mapStopReason(choice.finish_reason);
    }
}
```

**流式工具调用的挑战**：LLM 的工具调用参数是通过 SSE 流式传输的 JSON 片段。pi-ai 使用 `partial-json` 库实时解析部分 JSON，使得 UI 可以在工具调用参数完全到达之前就开始渲染。

### 3.5.4 思考/推理内容的处理

OpenAI 兼容的 API 使用不同的字段名来传递推理内容：

```typescript
const reasoningFields = ["reasoning_content", "reasoning", "reasoning_text"];
for (const field of reasoningFields) {
    const value = deltaFields[field];
    if (typeof value === "string" && value.length > 0) {
        // 找到推理内容字段
        const block = ensureThinkingBlock(field);
        block.thinking += value;
        stream.push({ type: "thinking_delta", ... });
    }
}
```

## 3.6 懒加载 Provider 注册

pi-ai 的一个关键设计是 **懒加载**（Lazy Loading）。Provider 模块不会在应用启动时静态导入，而是通过 `register-builtins.ts` 中的包装器按需加载：

```typescript
// src/providers/register-builtins.ts

function createLazyStream<TApi, TOptions, TSimpleOptions>(
    loadModule: () => Promise<LazyProviderModule<TApi, TOptions, TSimpleOptions>>
): StreamFunction<TApi, TOptions> {
    return (model, context, options) => {
        const outer = new AssistantMessageEventStream();

        loadModule().then((module) => {
            const inner = module.stream(model, context, options);
            forwardStream(outer, inner);
        }).catch((error) => {
            const message = createLazyLoadErrorMessage(model, error);
            outer.push({ type: "error", ... });
            outer.end(message);
        });

        return outer;
    };
}

// 每个 Provider 都有自己的加载函数和缓存
let anthropicProviderModulePromise: Promise<...> | undefined;

function loadAnthropicProviderModule() {
    anthropicProviderModulePromise ||= import("./anthropic.ts").then((module) => ({
        stream: module.streamAnthropic,
        streamSimple: module.streamSimpleAnthropic,
    }));
    return anthropicProviderModulePromise;
}

// 注册到 Registry
registerApiProvider({
    api: "anthropic-messages",
    stream: createLazyStream(loadAnthropicProviderModule),
    streamSimple: createLazySimpleStream(loadAnthropicProviderModule),
});
```

**这样设计的好处**：
1. **启动速度快**：不需要在启动时加载所有 Provider 的依赖（`openai`、`@anthropic-ai/sdk`、`@google/genai` 等大型 SDK）
2. **内存占用低**：只有真正用到的 Provider 才会被加载到内存
3. **可扩展性**：扩展系统可以通过 `registerApiProvider()` 在运行时注册自定义 Provider
4. **错误隔离**：如果某个 Provider 模块加载失败，不会影响其他 Provider

## 3.7 环境变量与认证

pi-ai 的 `env-api-keys.ts` 处理从环境变量中读取 API Key 的逻辑。它支持 30+ 个 Provider 的认证方式：

```typescript
// 通过环境变量名查找
const envMap: Record<string, string> = {
    openai: "OPENAI_API_KEY",
    deepseek: "DEEPSEEK_API_KEY",
    google: "GEMINI_API_KEY",
    anthropic: "ANTHROPIC_API_KEY",  // 或 ANTHROPIC_OAUTH_TOKEN
    // ...
};

// 非标准认证（AWS Bedrock 和 Google Vertex）
if (provider === "amazon-bedrock") {
    // 检查 AWS_PROFILE, AWS_ACCESS_KEY_ID+AWS_SECRET_ACCESS_KEY, 等
}
if (provider === "google-vertex") {
    // 检查 GOOGLE_APPLICATION_CREDENTIALS 或默认 ADC 路径
}
```

## 3.8 模型注册表

`models.generated.ts` 包含所有已知模型的元数据，由 `scripts/generate-models.ts` 自动生成：

```typescript
// models.ts — 运行时加载 models.generated.ts
import { MODELS } from "./models.generated.ts";

const modelRegistry: Map<string, Map<string, Model<Api>>> = new Map();

// 初始化
for (const [provider, models] of Object.entries(MODELS)) {
    const providerModels = new Map<string, Model<Api>>();
    for (const [id, model] of Object.entries(models)) {
        providerModels.set(id, model as Model<Api>);
    }
    modelRegistry.set(provider, providerModels);
}

// 查找模型
export function getModel(provider, modelId) {
    return modelRegistry.get(provider)?.get(modelId);
}
```

这种方式的好处是模型数据与代码分离，新增 Provider 时只需更新生成脚本，而不需要修改运行时代码。

## 3.9 认证与 OAuth 支持

除了 API Key 认证，pi-ai 还支持 OAuth 认证，主要用于 GitHub Copilot：

```typescript
// src/oauth.ts — OAuth 登录流程
export interface OAuthProvider {
    id: OAuthProviderId;
    name: string;
    startLogin(): Promise<{ url: string; code: string }>;
    pollForToken(deviceCode: string): Promise<OAuthCredentials>;
}

// 支持 OAuth 的 Provider
export const OAUTH_PROVIDERS: OAuthProvider[] = [
    { id: "github-copilot", name: "GitHub Copilot", ... },
    // future: 其他 OAuth Provider
];
```

OAuth 流程在 `src/utils/oauth/` 目录下有完整的实现，包括设备授权码流程（Device Authorization Grant）。

## 3.10 章节小结

pi-ai 的设计很好地演示了如何构建一个可扩展的、统一的 LLM API 网关：

1. **三层次抽象**：`Provider`（服务商）→ `Api`（协议）→ `Model`（具体模型）
2. **插件式注册**：`api-registry.ts` 支持运行时动态注册/卸载 Provider
3. **懒加载**：Provider 模块按需加载，降低启动开销
4. **统一事件协议**：`AssistantMessageEvent` 的 14 种事件类型覆盖了所有 LLM 交互场景
5. **EventStream**：基于 AsyncIterable 的通用流式数据结构，支持推拉两种模式
6. **消息转换**：统一的 `Message[] ↔ Provider 特定格式` 转换层
7. **模型数据与代码分离**：`models.generated.ts` 自动生成
