# 实战项目二：自定义 LLM Provider — 接入 DeepSeek

## 项目概述

前面的项目中我们通过扩展系统使用现成的 Provider。这次我们将**深入 pi-ai 内部**，实现一个完整的自定义 LLM Provider，将 DeepSeek（深度求索）的原生 API 接入 pi。

通过这个项目，你将掌握：
- pi-ai 的 Provider 注册机制（`api-registry.ts`）
- 消息格式转换（`Message[]` ↔ Provider 格式）
- 流式响应的解析（SSE 事件流 → `AssistantMessageEvent`）
- Thinking/Reasoning 内容处理
- Tool Calling 实现
- 模型元数据注册

## 项目结构

```
pi-provider-deepseek/
├── src/
│   ├── index.ts              # 扩展入口（注册 Provider）
│   ├── deepseek-provider.ts  # Provider 核心实现
│   ├── convert-messages.ts   # 消息格式转换
│   ├── parse-stream.ts       # SSE 流解析
│   └── models.ts             # 模型定义
├── package.json
└── README.md
```

## DeepSeek API 简介

DeepSeek 提供了与 OpenAI 兼容的 Chat Completions API，但有自己独特的字段：

```http
POST https://api.deepseek.com/chat/completions
Authorization: Bearer <sk-xxx>
Content-Type: application/json

{
    "model": "deepseek-chat",
    "messages": [...],
    "stream": true,
    "thinking": { "type": "enabled" },  ← DeepSeek 特有：推理模式
    "reasoning_effort": "high"
}
```

DeepSeek 的 SSE 响应流与 OpenAI 兼容，但在 `delta` 中包含额外的 `reasoning_content` 字段：

```json
{
    "choices": [{
        "delta": {
            "reasoning_content": "思考过程...",  ← DeepSeek 特有
            "content": "最终回复..."
        }
    }]
}
```

## Provider 核心实现

```typescript
// src/deepseek-provider.ts
import OpenAI from "openai";
import type { ChatCompletionChunk } from "openai/resources/chat/completions.js";
import { getEnvApiKey } from "@earendil-works/pi-ai";
import {
    AssistantMessageEventStream,
    calculateCost,
} from "@earendil-works/pi-ai";
import type {
    Api,
    AssistantMessage,
    Context,
    Model,
    SimpleStreamOptions,
    StreamFunction,
    StreamOptions,
    TextContent,
    ThinkingContent,
    ToolCall,
    ToolResultMessage,
    Usage,
} from "@earendil-works/pi-ai";
import { convertMessages } from "./convert-messages.ts";

// ============================================================================
// Provider 特定选项
// ============================================================================

export interface DeepSeekOptions extends StreamOptions {
    /** 推理级别 (DeepSeek 原生支持) */
    reasoningEffort?: "minimal" | "low" | "medium" | "high" | "xhigh";
    /** 是否启用 Thinking 模式 */
    enableThinking?: boolean;
    /** 最大推理 tokens */
    maxThinkingTokens?: number;
    /** Tool choice */
    toolChoice?: "auto" | "none" | "required" | { type: "function"; function: { name: string } };
}

// ============================================================================
// 默认模型
// ============================================================================

const DEEPSEEK_DEFAULT_MODEL: Model<"deepseek-chat"> = {
    id: "deepseek-chat",
    name: "DeepSeek V3",
    api: "deepseek-chat",
    provider: "deepseek",
    baseUrl: "https://api.deepseek.com",
    reasoning: true,
    thinkingLevelMap: {
        off: null,
        minimal: "minimal",
        low: "low",
        medium: "medium",
        high: "high",
        xhigh: "xhigh",
    },
    input: ["text", "image"] as ("text" | "image")[],
    cost: { input: 0.27, output: 1.10, cacheRead: 0.07, cacheWrite: 0.27 },
    contextWindow: 64_000,
    maxTokens: 8_192,
};

// ============================================================================
// 核心 Stream 函数
// ============================================================================

export const streamDeepSeek: StreamFunction<"deepseek-chat", DeepSeekOptions> = (
    model: Model<"deepseek-chat">,
    context: Context,
    options?: DeepSeekOptions,
): AssistantMessageEventStream => {
    const stream = new AssistantMessageEventStream();

    (async () => {
        const output: AssistantMessage = createEmptyMessage(model);

        try {
            const apiKey = options?.apiKey || getEnvApiKey("deepseek") || "";
            const client = createClient(model, apiKey, options);

            // 构建请求参数
            const params = buildParams(model, context, options);

            // 发送请求并获得流式响应
            const { data: openaiStream, response } = await client.chat.completions
                .create(params, { signal: options?.signal })
                .withResponse();

            await options?.onResponse?.({
                status: response.status,
                headers: Object.fromEntries(response.headers.entries()),
            }, model);

            // 发射 start 事件
            stream.push({ type: "start", partial: output });

            // 处理流式响应
            await processStream(openaiStream, output, stream, options);

            // 完成
            stream.push({ type: "done", reason: output.stopReason as any, message: output });
            stream.end();

        } catch (error: any) {
            handleStreamError(error, output, stream, options);
        }
    })();

    return stream;
};

// ============================================================================
// streamSimple — 统一接口
// ============================================================================

export const streamSimpleDeepSeek: StreamFunction<"deepseek-chat", SimpleStreamOptions> = (
    model: Model<"deepseek-chat">,
    context: Context,
    options?: SimpleStreamOptions,
): AssistantMessageEventStream => {
    const apiKey = options?.apiKey || getEnvApiKey("deepseek");
    if (!apiKey) {
        throw new Error("No API key for DeepSeek. Set DEEPSEEK_API_KEY environment variable.");
    }

    return streamDeepSeek(model, context, {
        ...options,
        apiKey,
        enableThinking: options?.reasoning ? options.reasoning !== "off" : false,
        reasoningEffort: options?.reasoning && options.reasoning !== "off"
            ? options.reasoning : undefined,
    } satisfies DeepSeekOptions);
};

// ============================================================================
// 辅助函数
// ============================================================================

function createEmptyMessage(model: Model<any>): AssistantMessage {
    return {
        role: "assistant",
        content: [],
        api: model.api,
        provider: model.provider,
        model: model.id,
        usage: {
            input: 0, output: 0, cacheRead: 0, cacheWrite: 0,
            totalTokens: 0,
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
        },
        stopReason: "stop",
        timestamp: Date.now(),
    };
}

function createClient(model: Model<any>, apiKey: string, options?: DeepSeekOptions): OpenAI {
    return new OpenAI({
        apiKey,
        baseURL: model.baseUrl,
        dangerouslyAllowBrowser: true,
        timeout: options?.timeoutMs ?? 600_000,
        maxRetries: options?.maxRetries ?? 2,
    });
}

function buildParams(
    model: Model<"deepseek-chat">,
    context: Context,
    options?: DeepSeekOptions,
): OpenAI.Chat.Completions.ChatCompletionCreateParamsStreaming {
    const messages = convertMessages(context.messages);
    const params: OpenAI.Chat.Completions.ChatCompletionCreateParamsStreaming = {
        model: model.id,
        messages,
        stream: true,
        stream_options: { include_usage: true },
    };

    if (options?.maxTokens) {
        params.max_tokens = options.maxTokens;
    }
    if (options?.temperature !== undefined) {
        params.temperature = options.temperature;
    }

    // DeepSeek 特有的 Thinking 模式 — 使用 thinking 字段
    if (options?.enableThinking) {
        (params as any).thinking = { type: "enabled" };
        if (options?.reasoningEffort) {
            (params as any).reasoning_effort =
                model.thinkingLevelMap?.[options.reasoningEffort] ?? options.reasoningEffort;
        }
        if (options?.maxThinkingTokens) {
            (params as any).thinking.max_tokens = options.maxThinkingTokens;
        }
    }

    // 工具调用
    if (context.tools && context.tools.length > 0) {
        params.tools = context.tools.map(t => ({
            type: "function" as const,
            function: {
                name: t.name,
                description: t.description,
                parameters: t.parameters as any,
                strict: true,
            },
        }));
    }

    if (options?.toolChoice) {
        params.tool_choice = options.toolChoice;
    }

    return params;
}

async function processStream(
    openaiStream: AsyncIterable<ChatCompletionChunk>,
    output: AssistantMessage,
    stream: AssistantMessageEventStream,
    options?: DeepSeekOptions,
): Promise<void> {
    type StreamingToolCall = ToolCall & { partialArgs?: string; streamIndex?: number };
    let textBlock: TextContent | null = null;
    let thinkingBlock: ThinkingContent | null = null;
    const toolCallBlocks = new Map<number, StreamingToolCall>();

    const getContentIndex = (block: any) => output.content.indexOf(block);

    const ensureTextBlock = () => {
        if (!textBlock) {
            textBlock = { type: "text", text: "" };
            output.content.push(textBlock);
            stream.push({
                type: "text_start",
                contentIndex: getContentIndex(textBlock),
                partial: output,
            });
        }
        return textBlock;
    };

    const ensureThinkingBlock = () => {
        if (!thinkingBlock) {
            thinkingBlock = { type: "thinking", thinking: "" };
            output.content.push(thinkingBlock);
            stream.push({
                type: "thinking_start",
                contentIndex: getContentIndex(thinkingBlock),
                partial: output,
            });
        }
        return thinkingBlock;
    };

    for await (const chunk of openaiStream) {
        if (!chunk || typeof chunk !== "object") continue;

        // 收集 response ID 和 usage
        output.responseId ||= chunk.id;
        if (chunk.usage) {
            output.usage = {
                input: chunk.usage.prompt_tokens || 0,
                output: chunk.usage.completion_tokens || 0,
                cacheRead: 0,
                cacheWrite: 0,
                totalTokens: chunk.usage.total_tokens || 0,
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
            };
            // 计算成本
            output.usage.cost = calculateCost(
                { cost: { input: 0.27, output: 1.10, cacheRead: 0.07, cacheWrite: 0.27 } } as any,
                output.usage,
            );
        }

        const choice = chunk.choices?.[0];
        if (!choice) continue;

        if (choice.finish_reason) {
            output.stopReason = mapFinishReason(choice.finish_reason);
        }

        const delta = choice.delta;
        if (!delta) continue;

        // 处理推理内容 (DeepSeek 特有的 reasoning_content)
        const deltaAny = delta as any;
        if (deltaAny.reasoning_content && typeof deltaAny.reasoning_content === "string") {
            const block = ensureThinkingBlock();
            block.thinking += deltaAny.reasoning_content;
            stream.push({
                type: "thinking_delta",
                contentIndex: getContentIndex(block),
                delta: deltaAny.reasoning_content,
                partial: output,
            });
        }

        // 处理文本内容
        if (delta.content) {
            const block = ensureTextBlock();
            block.text += delta.content;
            stream.push({
                type: "text_delta",
                contentIndex: getContentIndex(block),
                delta: delta.content,
                partial: output,
            });
        }

        // 处理工具调用
        if (delta.tool_calls) {
            for (const tc of delta.tool_calls) {
                const idx = tc.index ?? 0;
                let block = toolCallBlocks.get(idx);
                if (!block) {
                    block = {
                        type: "toolCall",
                        id: tc.id || "",
                        name: tc.function?.name || "",
                        arguments: {},
                        partialArgs: "",
                        streamIndex: idx,
                    };
                    toolCallBlocks.set(idx, block);
                    output.content.push(block);
                    stream.push({
                        type: "toolcall_start",
                        contentIndex: getContentIndex(block),
                        partial: output,
                    });
                }
                if (tc.id) block.id = tc.id;
                if (tc.function?.name) block.name = tc.function.name;
                if (tc.function?.arguments) {
                    block.partialArgs += tc.function.arguments;
                    try {
                        block.arguments = JSON.parse(block.partialArgs!);
                    } catch {
                        // 部分 JSON，继续等待
                    }
                    stream.push({
                        type: "toolcall_delta",
                        contentIndex: getContentIndex(block),
                        delta: tc.function.arguments,
                        partial: output,
                    });
                }
            }
        }
    }

    // 结束所有块
    for (const block of output.content) {
        const idx = output.content.indexOf(block);
        if (block.type === "text") {
            stream.push({ type: "text_end", contentIndex: idx, content: block.text, partial: output });
        } else if (block.type === "thinking") {
            stream.push({ type: "thinking_end", contentIndex: idx, content: block.thinking, partial: output });
        } else if (block.type === "toolCall") {
            const tc = block as StreamingToolCall;
            if (tc.partialArgs) {
                try { tc.arguments = JSON.parse(tc.partialArgs); } catch {}
                delete tc.partialArgs;
                delete tc.streamIndex;
            }
            stream.push({ type: "toolcall_end", contentIndex: idx, toolCall: block, partial: output });
        }
    }

    if (options?.signal?.aborted) {
        throw new Error("Request was aborted");
    }
}

function handleStreamError(
    error: any,
    output: AssistantMessage,
    stream: AssistantMessageEventStream,
    options?: DeepSeekOptions,
): void {
    output.stopReason = options?.signal?.aborted ? "aborted" : "error";
    output.errorMessage = error instanceof Error ? error.message : String(error);
    stream.push({ type: "error", reason: output.stopReason as any, error: output });
    stream.end();
}

function mapFinishReason(reason: string): "stop" | "length" | "toolUse" | "error" {
    switch (reason) {
        case "stop": return "stop";
        case "length": return "length";
        case "tool_calls": return "toolUse";
        default: return "error";
    }
}
```

## 消息格式转换

```typescript
// src/convert-messages.ts
import type { ChatCompletionMessageParam } from "openai/resources/chat/completions.js";
import type { Message } from "@earendil-works/pi-ai";

export function convertMessages(messages: Message[]): ChatCompletionMessageParam[] {
    return messages.map(msg => {
        switch (msg.role) {
            case "user": {
                const content = typeof msg.content === "string"
                    ? msg.content
                    : msg.content.map(block => {
                        if (block.type === "text") {
                            return { type: "text", text: block.text };
                        }
                        if (block.type === "image") {
                            return {
                                type: "image_url",
                                image_url: {
                                    url: `data:${block.mimeType};base64,${block.data}`,
                                },
                            };
                        }
                        return { type: "text", text: "" };
                    });
                return { role: "user", content } as ChatCompletionMessageParam;
            }

            case "assistant": {
                const result: any = { role: "assistant" };
                const textParts = msg.content.filter(c => c.type === "text");
                if (textParts.length > 0) {
                    result.content = textParts.map(c => c.text).join("");
                }
                const toolCalls = msg.content.filter(c => c.type === "toolCall");
                if (toolCalls.length > 0) {
                    result.tool_calls = toolCalls.map(tc => ({
                        id: tc.id,
                        type: "function",
                        function: {
                            name: tc.name,
                            arguments: JSON.stringify(tc.arguments),
                        },
                    }));
                }
                return result as ChatCompletionMessageParam;
            }

            case "toolResult": {
                const textContent = msg.content
                    .filter(c => c.type === "text")
                    .map(c => c.text)
                    .join("\n");
                return {
                    role: "tool",
                    tool_call_id: msg.toolCallId,
                    content: textContent,
                } as ChatCompletionMessageParam;
            }

            default:
                return { role: "user", content: "" } as ChatCompletionMessageParam;
        }
    });
}
```

## 扩展注册入口

```typescript
// src/index.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { streamDeepSeek, streamSimpleDeepSeek } from "./deepseek-provider.ts";

export default function (pi: ExtensionAPI) {
    pi.registerProvider("deepseek-chat", {
        api: "deepseek-chat",
        stream: streamDeepSeek,
        streamSimple: streamSimpleDeepSeek,
    });
}
```

## 环境变量

```bash
export DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
```

## 使用

```bash
# 直接在命令行指定模型
pi --model deepseek/deepseek-chat "用 Python 写一个快速排序"

# 或在交互模式中使用 /model 切换
# /model deepseek/deepseek-chat
```

## 测试

利用 pi-ai 的 Faux Provider 类比照编写测试：

```typescript
// test/deepseek-provider.test.ts
import { describe, it, expect } from "vitest";
import { streamDeepSeek, streamSimpleDeepSeek } from "../src/deepseek-provider.ts";

describe("DeepSeek Provider", () => {
    it("should convert messages correctly", () => {
        const { convertMessages } = require("../src/convert-messages.ts");
        const result = convertMessages([
            {
                role: "user",
                content: "Hello",
                timestamp: Date.now(),
            },
        ]);
        expect(result[0].role).toBe("user");
        expect((result[0] as any).content).toBe("Hello");
    });

    it("should handle tool calls in messages", () => {
        const { convertMessages } = require("../src/convert-messages.ts");
        const result = convertMessages([
            {
                role: "assistant",
                content: [
                    { type: "text", text: "I'll search for that." },
                    {
                        type: "toolCall",
                        id: "call_123",
                        name: "web_search",
                        arguments: { query: "hello" },
                    },
                ],
                api: "deepseek-chat",
                provider: "deepseek",
                model: "deepseek-chat",
                usage: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, totalTokens: 0,
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 } },
                stopReason: "toolUse",
                timestamp: Date.now(),
            },
        ]);
        expect((result[0] as any).tool_calls).toBeDefined();
        expect((result[0] as any).tool_calls[0].function.name).toBe("web_search");
    });
});
```
