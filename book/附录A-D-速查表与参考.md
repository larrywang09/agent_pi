# 附录 A 关键类型速查表

## A.1 @earendil-works/pi-tui

```typescript
// === 核心接口 ===
interface Component {
    render(width: number): string[];
    handleInput?(data: string): void;
    wantsKeyRelease?: boolean;
    invalidate(): void;
}

interface Focusable {
    focused: boolean;
}

interface Terminal {
    start(onInput: (data: string) => void, onResize: () => void): void;
    stop(): void;
    drainInput(maxMs?: number, idleMs?: number): Promise<void>;
    write(data: string): void;
    get columns(): number;
    get rows(): number;
    get kittyProtocolActive(): boolean;
    moveBy(lines: number): void;
    hideCursor(): void;
    showCursor(): void;
    clearLine(): void;
    clearFromCursor(): void;
    clearScreen(): void;
    setTitle(title: string): void;
    setProgress(active: boolean): void;
}

// === 覆盖层 ===
type OverlayAnchor = "center" | "top-left" | "top-right" | "bottom-left" | "bottom-right"
    | "top-center" | "bottom-center" | "left-center" | "right-center";

interface OverlayOptions {
    width?: SizeValue;
    minWidth?: number;
    maxHeight?: SizeValue;
    anchor?: OverlayAnchor;
    offsetX?: number;
    offsetY?: number;
    row?: SizeValue;
    col?: SizeValue;
    margin?: OverlayMargin | number;
    visible?: (termWidth: number, termHeight: number) => boolean;
    nonCapturing?: boolean;
}

interface OverlayHandle {
    hide(): void;
    setHidden(hidden: boolean): void;
    isHidden(): boolean;
    focus(): void;
    unfocus(): void;
    isFocused(): boolean;
}

// === 快捷键类型 ===
type Keybinding = keyof Keybindings;

interface KeybindingDefinition {
    defaultKeys: KeyId | KeyId[];
    description?: string;
}

type KeybindingsConfig = Record<string, KeyId | KeyId[] | undefined>;

// === 选择列表 ===
interface SelectItem {
    value: string;
    label: string;
    description?: string;
}

// === 设置项 ===
interface SettingItem {
    id: string;
    label: string;
    description?: string;
    currentValue: string;
    values?: string[];
    submenu?: (currentValue: string, done: (selectedValue?: string) => void) => Component;
}

// === 自动补全 ===
interface AutocompleteItem {
    value: string;
    label: string;
    description?: string;
}

interface SlashCommand {
    name: string;
    description: string;
    handler?: (args: string) => void;
}
```

## A.2 @earendil-works/pi-ai

```typescript
// === 核心类型 ===
type KnownApi = "openai-completions" | "mistral-conversations" | "openai-responses"
    | "azure-openai-responses" | "openai-codex-responses" | "anthropic-messages"
    | "bedrock-converse-stream" | "google-generative-ai" | "google-vertex";

type KnownProvider = "openai" | "anthropic" | "google" | "deepseek" | "github-copilot"
    | "openrouter" | "mistral" | "amazon-bedrock" | "fireworks" | "together"
    | "xai" | "groq" | "cerebras" | "minimax" | "moonshotai" | "huggingface"
    | "opencode" | "cloudflare-workers-ai" | ...;

// === 模型 ===
interface Model<TApi extends Api> {
    id: string;
    name: string;
    api: TApi;
    provider: Provider;
    baseUrl: string;
    reasoning: boolean;
    thinkingLevelMap?: ThinkingLevelMap;
    input: ("text" | "image")[];
    cost: { input: number; output: number; cacheRead: number; cacheWrite: number; };
    contextWindow: number;
    maxTokens: number;
    headers?: Record<string, string>;
    compat?: OpenAICompletionsCompat | OpenAIResponsesCompat | AnthropicMessagesCompat;
}

// === 消息 ===
interface UserMessage { role: "user"; content: string | (TextContent | ImageContent)[]; timestamp: number; }
interface AssistantMessage {
    role: "assistant";
    content: (TextContent | ThinkingContent | ToolCall)[];
    api: Api; provider: Provider; model: string;
    responseModel?: string; responseId?: string;
    usage: Usage;
    stopReason: StopReason;
    errorMessage?: string; timestamp: number;
}
interface ToolResultMessage { role: "toolResult"; toolCallId: string; toolName: string; content: (TextContent | ImageContent)[]; isError: boolean; timestamp: number; }

// === 内容块 ===
type TextContent = { type: "text"; text: string; textSignature?: string; };
type ThinkingContent = { type: "thinking"; thinking: string; thinkingSignature?: string; redacted?: boolean; };
type ImageContent = { type: "image"; data: string; mimeType: string; };
type ToolCall = { type: "toolCall"; id: string; name: string; arguments: Record<string, any>; };

// === 流式事件 ===
type AssistantMessageEvent =
    | { type: "start"; partial: AssistantMessage }
    | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
    | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
    | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
    | { type: "done"; reason: "stop"|"length"|"toolUse"; message: AssistantMessage }
    | { type: "error"; reason: "aborted"|"error"; error: AssistantMessage };

// === 工具 ===
interface Tool<TParameters extends TSchema> {
    name: string;
    description: string;
    parameters: TParameters;
}

// === 上下文 ===
interface Context {
    systemPrompt?: string;
    messages: Message[];
    tools?: Tool[];
}

// === Stream 函数 ===
type StreamFunction<TApi, TOptions> = (
    model: Model<TApi>,
    context: Context,
    options?: TOptions,
) => AssistantMessageEventStream;

// === 环境变量 ===
// Provider → 环境变量名映射
// openai:               OPENAI_API_KEY
// anthropic:            ANTHROPIC_API_KEY / ANTHROPIC_OAUTH_TOKEN
// google:               GEMINI_API_KEY
// deepseek:             DEEPSEEK_API_KEY
// openrouter:           OPENROUTER_API_KEY
// github-copilot:       COPILOT_GITHUB_TOKEN
// mistral:              MISTRAL_API_KEY
// fireworks:            FIREWORKS_API_KEY
// together:             TOGETHER_API_KEY
// groq:                 GROQ_API_KEY
// xai:                  XAI_API_KEY
// huggingface:          HF_TOKEN
```

## A.3 @earendil-works/pi-agent-core

```typescript
// === Agent 状态 ===
interface AgentState {
    systemPrompt: string;
    model: Model<any>;
    thinkingLevel: ThinkingLevel;
    set tools(tools: AgentTool<any>[]);
    get tools(): AgentTool<any>[];
    set messages(messages: AgentMessage[]);
    get messages(): AgentMessage[];
    readonly isStreaming: boolean;
    readonly streamingMessage?: AgentMessage;
    readonly pendingToolCalls: ReadonlySet<string>;
    readonly errorMessage?: string;
}

type QueueMode = "all" | "one-at-a-time";
type ToolExecutionMode = "sequential" | "parallel";
type ThinkingLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh";

// === Agent 事件 ===
type AgentEvent =
    | { type: "agent_start" }
    | { type: "agent_end"; messages: AgentMessage[] }
    | { type: "turn_start" }
    | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
    | { type: "message_start"; message: AgentMessage }
    | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
    | { type: "message_end"; message: AgentMessage }
    | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
    | { type: "tool_execution_update"; toolCallId: string; toolName: string; partialResult: any }
    | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };

// === Agent 工具 ===
interface AgentTool<TParameters extends TSchema, TDetails> extends Tool<TParameters> {
    label: string;
    prepareArguments?: (args: unknown) => Static<TParameters>;
    execute: (toolCallId: string, params: Static<TParameters>, signal?: AbortSignal,
        onUpdate?: AgentToolUpdateCallback<TDetails>) => Promise<AgentToolResult<TDetails>>;
    executionMode?: ToolExecutionMode;
}

// === Agent Loop 配置 ===
interface AgentLoopConfig extends SimpleStreamOptions {
    model: Model<any>;
    convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
    transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
    getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;
    shouldStopAfterTurn?: (context: ShouldStopAfterTurnContext) => boolean | Promise<boolean>;
    prepareNextTurn?: (context: PrepareNextTurnContext) => AgentLoopTurnUpdate | undefined | Promise<AgentLoopTurnUpdate | undefined>;
    getSteeringMessages?: () => Promise<AgentMessage[]>;
    getFollowUpMessages?: () => Promise<AgentMessage[]>;
    toolExecution?: ToolExecutionMode;
    beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult | undefined>;
    afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult | undefined>;
}
```

## A.4 @earendil-works/pi-coding-agent

```typescript
// === AgentSession 核心 ===
class AgentSession {
    get state(): AgentState;
    get model(): Model<any> | undefined;
    get thinkingLevel(): ThinkingLevel;
    async enqueueUserMessage(text: string, images?: ImageContent[]): Promise<void>;
    async runPrompt(): Promise<void>;
    async continue(): Promise<void>;
    subscribe(event: AgentSessionEvent, listener: Function): void;
    async switchToBranch(entries: SessionEntry[]): Promise<void>;
    async compact(): Promise<void>;
    async cycleModel(direction: "forward" | "backward"): Promise<void>;
    setThinkingLevel(level: ThinkingLevel): void;
    async executeBash(command: string): Promise<BashResult>;
}

// === SDK ===
interface CreateAgentSessionOptions {
    cwd?: string;
    agentDir?: string;
    authStorage?: AuthStorage;
    modelRegistry?: ModelRegistry;
    model?: Model<any>;
    thinkingLevel?: ThinkingLevel;
    scopedModels?: Array<{ model: Model<any>; thinkingLevel?: ThinkingLevel }>;
    noTools?: "all" | "builtin";
    tools?: string[];
    customTools?: ToolDefinition[];
    resourceLoader?: ResourceLoader;
    sessionManager?: SessionManager;
    settingsManager?: SettingsManager;
}

interface CreateAgentSessionResult {
    session: AgentSession;
    extensionsResult: LoadExtensionsResult;
    modelFallbackMessage?: string;
}

// === 扩展 API ===
interface ExtensionAPI {
    registerTool(tool: ToolDefinition): void;
    registerCommand(name: string, command: SlashCommand): void;
    registerShortcut(keys: string[], command: string): void;
    registerFlag(flag: ExtensionFlag): void;
    registerProvider(api: string, provider: ApiProvider): void;
    on(event: string, handler: (event: any, ctx: ExtensionContext) => void): void;
    registerMessageRenderer(renderer: MessageRenderer): void;
    setStatus(key: string, text: string | undefined): void;
    setWidget(key: string, content: ...): void;
    setFooter(factory: ...): void;
    setHeader(factory: ...): void;
    emit(eventType: string, data: unknown): void;
    getCommands(): SlashCommandInfo[];
    getAllTools(): ToolInfo[];
    getActiveTools(): string[];
    setActiveTools(toolNames: string[]): void;
    appendEntry<T>(customType: string, data: T): void;
}

// === Extension 上下文 ===
interface ExtensionContext {
    ui: ExtensionUIContext;
    hasUI: boolean;
    cwd: string;
    sessionManager: ReadonlySessionManager;
    modelRegistry: ModelRegistry;
    model: Model<any> | undefined;
    isIdle(): boolean;
    signal: AbortSignal | undefined;
    abort(): void;
    hasPendingMessages(): boolean;
    shutdown(): void;
    getContextUsage(): ContextUsage | undefined;
    compact(options?: CompactOptions): void;
    getSystemPrompt(): string;
}
```

---

# 附录 B 扩展 API 速查表

## B.1 扩展入口格式

每个扩展是一个 TypeScript 模块，默认导出一个函数：

```typescript
// 最简扩展
export default function (pi: ExtensionAPI) {
    pi.registerTool({ ... });
}

// 带配置的扩展
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
    // 注册工具
    pi.registerTool({ ... });

    // 注册命令
    pi.registerCommand("mycommand", {
        description: "My custom command",
        handler: async (args, ctx) => { ... },
    });

    // 监听事件
    pi.on("turn_start", async (event, ctx) => {
        console.log("Turn started");
    });
}
```

## B.2 安装位置

```bash
# 项目级别（推荐）
<project>/.pi/extensions/my-extension/index.ts

# 用户全局
~/.pi/agent/extensions/my-extension/index.ts

# 带依赖的扩展（需要 package.json）
<project>/.pi/extensions/my-extension/
├── index.ts
└── package.json
```

## B.3 事件速查

| 事件名 | 触发时机 | 可用操作 |
|--------|---------|---------|
| `session_start` | 会话加载/创建后 | 初始化状态 |
| `session_shutdown` | 会话关闭时 | 清理资源 |
| `session_before_compact` | 压缩前 | 修改压缩选项 |
| `session_compact` | 压缩后 | 重新计算状态 |
| `session_before_fork` | 分支前 | 阻止分支 |
| `session_before_switch` | 切换会话前 | 阻止切换 |
| `session_before_tree` | 导航树前 | 阻止导航 |
| `session_tree` | 导航树后 | 重建状态 |
| `input` | 用户输入后 | 修改/拦截输入 |
| `before_agent_start` | Agent 运行前 | 修改系统提示词 |
| `context` | LLM 请求前 | 查看/修改上下文 |
| `before_provider_request` | Provider 请求前 | 修改请求 |
| `agent_start` | Agent 开始运行 | — |
| `agent_end` | Agent 结束运行 | — |
| `turn_start` | 每个 turn 开始 | — |
| `turn_end` | 每个 turn 结束 | — |
| `message_start` | 消息开始 | — |
| `message_update` | 消息更新（流式） | — |
| `message_end` | 消息结束 | — |
| `tool_call` | 工具调用前 | 拦截/修改 |
| `tool_result` | 工具结果后 | 修改结果 |
| `user_bash` | 用户执行 Bash | 审计/拦截 |
| `resources_discover` | 资源发现 | 添加资源 |

## B.4 内置命令

| 命令 | 用途 |
|------|------|
| `/model` | 切换当前模型 |
| `/thinking` | 切换推理级别 |
| `/settings` | 打开设置面板 |
| `/session` | 选择/切换会话 |
| `/compact` | 手动触发压缩 |
| `/fork` | 从当前节点分支 |
| `/resume` | 恢复最近会话 |
| `/help` | 显示帮助信息 |
| `/clear` | 清除屏幕 |
| `/exit` | 退出 pi |

## B.5 内置快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+C` | 中断当前操作 |
| `Ctrl+D` | 清屏 |
| `Ctrl+O` | 打开模型选择器 |
| `Ctrl+P` | 循环下一个模型 |
| `Ctrl+Shift+P` | 循环上一个模型 |
| `Ctrl+L` | 切换思考级别 |
| `Ctrl+T` | 展开/折叠工具输出 |
| `Ctrl+E` | 打开外部编辑器 |
| `Escape` | 取消/关闭 |
| `Shift+Enter` | 换行（在编辑器中） |
| `Tab` | 自动补全 |
| `Shift+Ctrl+D` | 调试信息 |

---

# 附录 C Provider 开发指南

## C.1 Provider 注册流程

```typescript
// 1. 添加 API 标识符
// packages/ai/src/types.ts
export type KnownApi = "..." | "my-custom-api";

// 2. 创建 Provider 实现
// packages/ai/src/providers/my-provider.ts
import { AssistantMessageEventStream } from "../utils/event-stream.ts";

export const streamMyProvider: StreamFunction<"my-custom-api", MyOptions> = (model, context, options) => {
    const stream = new AssistantMessageEventStream();

    (async () => {
        try {
            // 消息格式转换
            const llmMessages = convertMessages(context.messages);

            // 发送 HTTP 请求
            const response = await fetch(`${model.baseUrl}/chat/completions`, {
                method: "POST",
                headers: { "Authorization": `Bearer ${apiKey}` },
                body: JSON.stringify({ messages: llmMessages, stream: true }),
            });

            // 流式解析
            const reader = response.body.getReader();
            stream.push({ type: "start", partial: output });

            while (true) {
                const { done, value } = await reader.read();
                if (done) break;
                // 解析 chunk → 发射事件
                for (const event of parseChunk(value, output)) {
                    stream.push(event);
                }
            }

            // 结束
            stream.push({ type: "done", reason: "stop", message: output });
        } catch (error) {
            stream.push({ type: "error", reason: "error", error: createErrorMessage(error) });
        }
    })();

    return stream;
};

// 3. 注册到 Registry
// packages/ai/src/providers/register-builtins.ts
import { streamMyProvider, streamSimpleMyProvider } from "./my-provider.ts";

registerApiProvider({
    api: "my-custom-api",
    stream: streamMyProvider,
    streamSimple: streamSimpleMyProvider,
});

// 4. 添加模型数据
// packages/ai/scripts/generate-models.ts

// 5. 添加环境变量支持
// packages/ai/src/env-api-keys.ts
```

## C.2 通过扩展注册 Provider

```typescript
// 扩展方式，无需修改 pi 源码
export default function (pi: ExtensionAPI) {
    pi.registerProvider("my-custom-api", {
        api: "my-custom-api",

        stream(model, context, options) {
            const stream = new AssistantMessageEventStream();
            // ... 实现
            return stream;
        },

        streamSimple(model, context, options) {
            return this.stream(model, context, options);
        },
    });
}
```

## C.3 Provider 响应事件规范

Provider 必须发射以下事件序列：

```
必须:
  start → {text|thinking|toolcall}_delta* → done

可选:
  text_start/end     (文本块开始/结束边界)
  thinking_start/end (思考块开始/结束边界)
  toolcall_start/end (工具调用块开始/结束边界)

错误:
  任何位置 → error → 终止
```

---

# 附录 D 参考资源与链接

## D.1 官方资源

| 资源 | 链接 |
|------|------|
| 项目官网 | [https://pi.dev](https://pi.dev) |
| 在线文档 | [https://pi.dev/docs/latest](https://pi.dev/docs/latest) |
| GitHub 仓库 | [https://github.com/earendil-works/pi-mono](https://github.com/earendil-works/pi-mono) |
| Discord 社区 | [https://discord.com/invite/3cU7Bz4UPx](https://discord.com/invite/3cU7Bz4UPx) |
| HF 会话数据集 | [https://huggingface.co/datasets/badlogicgames/pi-mono](https://huggingface.co/datasets/badlogicgames/pi-mono) |

## D.2 代码库关键路径

| 路径 | 说明 |
|------|------|
| `packages/tui/src/` | TUI 框架源码 |
| `packages/ai/src/` | LLM 统一 API 源码 |
| `packages/ai/src/providers/` | 所有 Provider 实现 |
| `packages/agent/src/` | Agent 运行时源码 |
| `packages/coding-agent/src/` | CLI 应用源码 |
| `packages/coding-agent/src/core/extensions/` | 扩展系统源码 |
| `packages/coding-agent/src/modes/interactive/` | 交互模式源码 |
| `packages/coding-agent/examples/extensions/` | 95+ 扩展示例 |
| `packages/coding-agent/examples/sdk/` | SDK 使用示例 |

## D.3 外部依赖

| 包名 | 用途 | 文档链接 |
|------|------|---------|
| `typebox` | JSON Schema 生成 | [https://github.com/sinclairzx81/typebox](https://github.com/sinclairzx81/typebox) |
| `openai` | OpenAI SDK | [https://github.com/openai/openai-node](https://github.com/openai/openai-node) |
| `@anthropic-ai/sdk` | Anthropic SDK | [https://github.com/anthropics/anthropic-sdk-typescript](https://github.com/anthropics/anthropic-sdk-typescript) |
| `@google/genai` | Google Gemini SDK | [https://github.com/googleapis/nodejs-genai](https://github.com/googleapis/nodejs-genai) |
| `marked` | Markdown 解析 | [https://marked.js.org/](https://marked.js.org/) |
| `chalk` | 终端颜色 | [https://github.com/chalk/chalk](https://github.com/chalk/chalk) |
| `highlight.js` | 代码高亮 | [https://highlightjs.org/](https://highlightjs.org/) |
| `Biome` | 格式化 + Lint | [https://biomejs.dev/](https://biomejs.dev/) |
| `Vitest` | 测试框架 | [https://vitest.dev/](https://vitest.dev/) |

## D.4 相关技术标准

| 标准 | 说明 | 链接 |
|------|------|------|
| ANSI Escape Codes | 终端转义序列标准 | [https://en.wikipedia.org/wiki/ANSI_escape_code](https://en.wikipedia.org/wiki/ANSI_escape_code) |
| CSI 2026 | 同步输出协议 | [https://gist.github.com/christianparpart/d8a62cc1ab659194337d73e399004036](https://gist.github.com/christianparpart/d8a62cc1ab659194337d73e399004036) |
| Kitty Keyboard Protocol | 增强键盘协议 | [https://sw.kovidgoyal.net/kitty/keyboard-protocol/](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) |
| Kitty Graphics Protocol | 终端图片协议 | [https://sw.kovidgoyal.net/kitty/graphics-protocol/](https://sw.kovidgoyal.net/kitty/graphics-protocol/) |
| iTerm2 Images | 内联图片协议 | [https://iterm2.com/documentation-images.html](https://iterm2.com/documentation-images.html) |
| Server-Sent Events (SSE) | 服务端推送事件 | [https://html.spec.whatwg.org/multipage/server-sent-events.html](https://html.spec.whatwg.org/multipage/server-sent-events.html) |
| OpenAPI Tool Calling | 工具调用规范 | [https://platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling) |
| Anthropic Tool Use | Tool Use API | [https://docs.anthropic.com/en/docs/build-with-claude/tool-use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) |
| JSONL | JSON Lines 格式 | [https://jsonlines.org/](https://jsonlines.org/) |

## D.5 推荐阅读

| 主题 | 资源 | 作者/来源 |
|------|------|----------|
| Network Programming with ANSI Escape Codes | 系列博客 | [https://www.youtube.com/watch?v=eUcLcIoyCxY](https://www.youtube.com/watch?v=eUcLcIoyCxY) |
| Designing Terminal UIs | 书籍 | Update / No Starch Press |
| Building LLM Applications | 指南 | OpenAI Cookbook |
| Agent Design Patterns | 论文 | Anthropic / Google Research |
| TypeScript Erasable Syntax | TypeScript 5.8 新特性 | Dev Blog |
