# 第 7 章 实战项目：构建 AI Code Review Assistant

## 7.1 项目概述

本章将通过一个完整的实战项目，综合运用前面 6 章学到的所有知识。我们将构建一个 **pi-code-review-assistant**，它是一个 pi 扩展，能够：

1. 添加 `review_pr` 工具，让 LLM 可以直接调用 GitHub PR Review API
2. 添加 `list_comments` 和 `post_review` 工具，形成完整的 Review 工作流
3. 添加 `/pr-review` 用户命令，手动触发 Review
4. 注册 `context` 事件监听，自动注入 PR 上下文
5. 自定义 TUI 渲染，显示评分和差异对比
6. 通过 SDK 提供批处理模式

### 项目文件结构

```
pi-code-review-assistant/
├── src/
│   ├── index.ts              # 扩展入口
│   ├── tools/
│   │   ├── review-pr.ts      # review_pr 工具
│   │   ├── list-comments.ts  # list_comments 工具
│   │   └── post-review.ts    # post_review 工具
│   ├── components/
│   │   └── review-result.ts  # 自定义渲染组件
│   ├── github-api.ts         # GitHub API 封装
│   └── sdk-example.ts        # SDK 批处理示例
├── package.json
└── README.md
```

## 7.2 安装与配置

扩展需要放置在 pi 的扩展加载路径中：

```bash
# 项目级别
mkdir -p .pi/extensions/pi-code-review-assistant
ln -s /path/to/pi-code-review-assistant/src/index.ts .pi/extensions/pi-code-review-assistant/index.ts

# 或用户全局
mkdir -p ~/.pi/agent/extensions/pi-code-review-assistant
```

需要设置 GitHub API Token：

```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```

## 7.3 GitHub API 封装

首先实现与 GitHub 通信的基础模块：

```typescript
// src/github-api.ts
export interface PRInfo {
    owner: string;
    repo: string;
    prNumber: number;
}

export interface PRFile {
    filename: string;
    status: string;
    additions: number;
    deletions: number;
    changes: number;
    patch?: string;
    contents_url: string;
}

export interface PRReview {
    id: number;
    body: string;
    state: "APPROVED" | "CHANGES_REQUESTED" | "COMMENTED";
    user: { login: string };
    submitted_at: string;
}

export interface PRReviewComment {
    id: number;
    body: string;
    path: string;
    position: number;
    line: number;
    user: { login: string };
}

export class GitHubAPI {
    private token: string;
    private baseUrl = "https://api.github.com";

    constructor(token?: string) {
        this.token = token || process.env.GITHUB_TOKEN || "";
        if (!this.token) {
            throw new Error("GITHUB_TOKEN is required. Set it via environment variable.");
        }
    }

    private async request<T>(path: string, options: RequestInit = {}): Promise<T> {
        const url = `${this.baseUrl}${path}`;
        const response = await fetch(url, {
            ...options,
            headers: {
                "Authorization": `Bearer ${this.token}`,
                "Accept": "application/vnd.github.v3+json",
                "Content-Type": "application/json",
                ...options.headers,
            },
        });

        if (!response.ok) {
            const error = await response.text();
            throw new Error(`GitHub API error (${response.status}): ${error}`);
        }

        return response.json();
    }

    // 解析 PR URL 或 "owner/repo#number" 格式
    parsePRInput(input: string): PRInfo {
        // 支持格式:
        // "owner/repo#123"
        // "https://github.com/owner/repo/pull/123"
        // "--owner owner --repo repo --pr 123"

        const urlMatch = input.match(/github\.com\/([^/]+)\/([^/]+)\/pull\/(\d+)/);
        if (urlMatch) {
            return { owner: urlMatch[1], repo: urlMatch[2], prNumber: parseInt(urlMatch[3]) };
        }

        const shortMatch = input.match(/^([^/]+)\/([^#]+)#(\d+)$/);
        if (shortMatch) {
            return { owner: shortMatch[1], repo: shortMatch[2], prNumber: parseInt(shortMatch[3]) };
        }

        throw new Error(`Unable to parse PR input: "${input}". Use format "owner/repo#123" or full URL.`);
    }

    // 获取 PR 详情
    async getPR(info: PRInfo): Promise<any> {
        return this.request(`/repos/${info.owner}/${info.repo}/pulls/${info.prNumber}`);
    }

    // 获取 PR 变更的文件列表
    async getPRFiles(info: PRInfo): Promise<PRFile[]> {
        return this.request(`/repos/${info.owner}/${info.repo}/pulls/${info.prNumber}/files`);
    }

    // 获取 PR 的 Review 评论
    async getPRReviews(info: PRInfo): Promise<PRReview[]> {
        return this.request(`/repos/${info.owner}/${info.repo}/pulls/${info.prNumber}/reviews`);
    }

    // 获取 PR 的 Review Comments
    async getPRComments(info: PRInfo): Promise<PRReviewComment[]> {
        return this.request(`/repos/${info.owner}/${info.repo}/pulls/${info.prNumber}/comments`);
    }

    // 对 PR 提交 Review
    async createReview(
        info: PRInfo,
        body: string,
        event: "APPROVE" | "REQUEST_CHANGES" | "COMMENT",
        comments: Array<{ path: string; body: string; line?: number }> = [],
    ): Promise<PRReview> {
        return this.request(`/repos/${info.owner}/${info.repo}/pulls/${info.prNumber}/reviews`, {
            method: "POST",
            body: JSON.stringify({
                body,
                event,
                comments,
            }),
        });
    }
}
```

## 7.4 工具定义

### 7.4.1 review_pr 工具

这个工具允许 LLM 获取一个 PR 的详细信息并进行代码审查：

```typescript
// src/tools/review-pr.ts
import { Type } from "@earendil-works/pi-ai";
import { defineTool } from "@earendil-works/pi-coding-agent";
import { GitHubAPI, type PRFile } from "../github-api.ts";

export const reviewPRTool = defineTool({
    name: "review_pr",
    label: "Review PR",
    description: `Fetch a GitHub Pull Request and review its code changes.
Returns PR metadata, file diffs, and existing reviews.
Use this tool to analyze code quality, suggest improvements, and identify issues.

The tool will automatically:
1. Fetch PR title, description, and metadata
2. Get all changed files with their diffs
3. Check existing reviews and comments
4. Provide a structured analysis`,
    parameters: Type.Object({
        pr: Type.String({
            description: "Pull request identifier. Format: 'owner/repo#123' or full GitHub URL",
        }),
        focus: Type.Optional(
            Type.String({
                description: "Optional area to focus the review on. E.g., 'security', 'performance', 'api-design'",
            }),
        ),
        maxFiles: Type.Optional(
            Type.Number({
                description: "Maximum number of files to review (default: 10, use fewer for large PRs)",
                default: 10,
            }),
        ),
    }),
    async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
        const api = new GitHubAPI();
        const info = api.parsePRInput(params.pr);
        const maxFiles = params.maxFiles || 10;

        // 获取 PR 信息
        const pr = await api.getPR(info);
        const files = await api.getPRFiles(info);
        const reviews = await api.getPRReviews(info);
        const comments = await api.getPRComments(info);

        // 限制文件数量
        const filesToReview = files.slice(0, maxFiles);

        // 构造 Review 上下文
        let reviewContext = `# Pull Request Review\n\n`;
        reviewContext += `## PR #${pr.number}: ${pr.title}\n\n`;
        reviewContext += `${pr.body || "*No description provided*"}\n\n`;

        reviewContext += `## Changes Overview\n`;
        reviewContext += `- Total files changed: ${files.length}\n`;
        reviewContext += `- Additions: ${pr.additions}, Deletions: ${pr.deletions}\n`;
        reviewContext += `- Status: ${pr.merged ? "Merged" : pr.draft ? "Draft" : "Open"}\n`;
        reviewContext += `- Branch: ${pr.head.label} → ${pr.base.label}\n\n`;

        if (params.focus) {
            reviewContext += `## Review Focus\n`;
            reviewContext += `Please focus on: **${params.focus}**\n\n`;
        }

        reviewContext += `## Files Changed\n`;
        for (const file of filesToReview) {
            reviewContext += `### ${file.filename} (${file.status})\n`;
            reviewContext += `+${file.additions} -${file.deletions} (${file.changes} changes)\n\n`;
            if (file.patch) {
                reviewContext += "```diff\n" + file.patch + "\n```\n\n";
            }
        }

        if (filesToReview.length < files.length) {
            reviewContext += `> *${files.length - filesToReview.length} more files not shown*`;
        }

        return {
            content: [{ type: "text", text: reviewContext }],
            details: {
                pr: { owner: info.owner, repo: info.repo, number: info.prNumber },
                title: pr.title,
                totalFiles: files.length,
                reviewedFiles: filesToReview.length,
                additions: pr.additions,
                deletions: pr.deletions,
                existingReviews: reviews.length,
                existingComments: comments.length,
            },
        };
    },
});
```

### 7.4.2 list_comments 工具

```typescript
// src/tools/list-comments.ts
import { Type } from "@earendil-works/pi-ai";
import { defineTool } from "@earendil-works/pi-coding-agent";
import { GitHubAPI } from "../github-api.ts";

export const listCommentsTool = defineTool({
    name: "list_comments",
    label: "List PR Comments",
    description: "List all review comments on a GitHub Pull Request.",
    parameters: Type.Object({
        pr: Type.String({
            description: "PR identifier. Format: 'owner/repo#123'",
        }),
    }),
    async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
        const api = new GitHubAPI();
        const info = api.parsePRInput(params.pr);
        const comments = await api.getPRComments(info);
        const reviews = await api.getPRReviews(info);

        let output = `# Comments on PR #${info.prNumber}\n\n`;

        if (reviews.length > 0) {
            output += `## Reviews (${reviews.length})\n`;
            for (const review of reviews) {
                output += `### ${review.user.login} — ${review.state}\n`;
                if (review.body) {
                    output += `${review.body}\n`;
                }
                output += `> Submitted: ${review.submitted_at}\n\n`;
            }
        }

        if (comments.length > 0) {
            output += `## Inline Comments (${comments.length})\n`;
            for (const comment of comments) {
                output += `- **${comment.user.login}** on \`${comment.path}\``;
                if (comment.line) output += ` line ${comment.line}`;
                output += `:\n  > ${comment.body}\n\n`;
            }
        }

        if (reviews.length === 0 && comments.length === 0) {
            output += "*No reviews or comments yet.*\n";
        }

        return {
            content: [{ type: "text", text: output }],
            details: { commentCount: comments.length, reviewCount: reviews.length },
        };
    },
});
```

### 7.4.3 post_review 工具

```typescript
// src/tools/post-review.ts
import { StringEnum } from "@earendil-works/pi-ai";
import { Type } from "@earendil-works/pi-ai";
import { defineTool } from "@earendil-works/pi-coding-agent";
import { GitHubAPI } from "../github-api.ts";

export const postReviewTool = defineTool({
    name: "post_review",
    label: "Post PR Review",
    description: "Submit a review on a GitHub Pull Request with comments and a conclusion.",
    parameters: Type.Object({
        pr: Type.String({
            description: "PR identifier. Format: 'owner/repo#123'",
        }),
        body: Type.String({
            description: "The main review body text with your analysis and suggestions",
        }),
        conclusion: StringEnum(["COMMENT", "APPROVE", "REQUEST_CHANGES"] as const),
        inlineComments: Type.Optional(
            Type.Array(
                Type.Object({
                    path: Type.String({ description: "File path" }),
                    body: Type.String({ description: "Comment text" }),
                    line: Type.Optional(
                        Type.Number({ description: "Line number to comment on" }),
                    ),
                }),
                { description: "Optional inline comments on specific lines" },
            ),
        ),
    }),
    async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
        const api = new GitHubAPI();
        const info = api.parsePRInput(params.pr);

        const review = await api.createReview(
            info,
            params.body,
            params.conclusion,
            params.inlineComments || [],
        );

        return {
            content: [{
                type: "text",
                text: `Review submitted successfully on ${info.owner}/${info.repo}#${info.prNumber}

Conclusion: ${params.conclusion}

${params.inlineComments?.length ? `\n${params.inlineComments.length} inline comment(s) posted.` : ""}`,
            }],
            details: {
                reviewId: review.id,
                conclusion: params.conclusion,
                inlineCommentCount: params.inlineComments?.length || 0,
            },
        };
    },
});
```

## 7.5 自定义渲染组件

让工具结果在 TUI 中显示更美观：

```typescript
// src/components/review-result.ts
import { Text, truncateToWidth } from "@earendil-works/pi-tui";
import type { Theme } from "@earendil-works/pi-coding-agent";

export function renderReviewResult(
    details: {
        totalFiles: number;
        reviewedFiles: number;
        additions: number;
        deletions: number;
        existingReviews: number;
        existingComments: number;
    },
    theme: Theme,
): Text {
    const lines: string[] = [];

    // 评分
    const score = calculateScore(details);
    const scoreColor = score >= 80 ? "success" : score >= 60 ? "warning" : "error";
    lines.push(theme.fg(scoreColor, theme.bold(`Quality Score: ${score}/100`)));

    lines.push("");
    lines.push(theme.fg("muted", "Changes Overview:"));
    lines.push(`  Files: ${theme.fg("accent", `${details.reviewedFiles}/${details.totalFiles}`)}`);
    lines.push(`  Additions: ${theme.fg("success", `+${details.additions}`)}`);
    lines.push(`  Deletions: ${theme.fg("error", `-${details.deletions}`)}`);

    lines.push("");
    lines.push(theme.fg("muted", "Existing Feedback:"));
    lines.push(`  Reviews: ${theme.fg("accent", String(details.existingReviews))}`);
    lines.push(`  Comments: ${theme.fg("accent", String(details.existingComments))}`);

    return new Text(lines.join("\n"), 1, 0);
}

function calculateScore(details: {
    totalFiles: number;
    additions: number;
    deletions: number;
}): number {
    // 简化的评分逻辑
    let score = 80; // 基础分

    // 大量变更扣分
    if (details.additions > 500) score -= 15;
    else if (details.additions > 200) score -= 5;

    // 大量文件扣分
    if (details.totalFiles > 20) score -= 10;
    else if (details.totalFiles > 10) score -= 5;

    return Math.max(0, Math.min(100, score));
}
```

## 7.6 扩展入口与注册

将所有内容组合到扩展入口文件中：

```typescript
// src/index.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { reviewPRTool } from "./tools/review-pr.ts";
import { listCommentsTool } from "./tools/list-comments.ts";
import { postReviewTool } from "./tools/post-review.ts";
import { renderReviewResult } from "./components/review-result.ts";

export default function (pi: ExtensionAPI) {
    // 注册三个工具
    pi.registerTool(reviewPRTool);
    pi.registerTool(listCommentsTool);
    pi.registerTool(postReviewTool);

    // 注册 /pr-review 命令
    pi.registerCommand("pr-review", {
        description: "Review a GitHub Pull Request",
        handler: async (_args, ctx) => {
            // 让用户输入 PR 标识
            const prInput = await ctx.ui.input(
                "Pull Request",
                "owner/repo#123 or full GitHub URL",
            );
            if (!prInput) {
                ctx.ui.notify("Review cancelled", "info");
                return;
            }

            // 通过 Agent 发送用户消息触发 Review
            ctx.sendUserMessage(
                `Please review PR ${prInput}. Use the review_pr tool to get the PR details, then provide a thorough code review covering code quality, potential bugs, security issues, and suggestions for improvement.`,
                { deliverAs: "steer" },
            );
        },
    });

    // 注册自定义工具渲染器
    reviewPRTool.renderResult = (result, options, theme, ctx) => {
        if (result.details) {
            return renderReviewResult(result.details, theme);
        }
        return undefined;
    };

    // 自动注入 PR 上下文（可选功能）
    pi.on("context", async (event, ctx) => {
        // 检查最近的用户消息是否包含 PR 引用
        const lastUserMsg = event.messages
            .filter((m: any) => m.role === "user")
            .pop();

        if (lastUserMsg) {
            const content = typeof lastUserMsg.content === "string"
                ? lastUserMsg.content
                : lastUserMsg.content.map((c: any) => c.text).join(" ");

            // 如果消息包含 PR 引用，注入 review 指南
            if (content.match(/(?:review|check|audit)\s+(?:PR|pull request)/i)) {
                event.appendSystemPrompt(`
## Code Review Guidelines

When reviewing code, focus on:
1. **Correctness**: Does the code work as intended?
2. **Security**: Are there any security vulnerabilities?
3. **Performance**: Are there performance issues?
4. **Maintainability**: Is the code clean and well-structured?
5. **Best Practices**: Does it follow community standards?

Provide specific, actionable feedback with code examples where possible.`);
            }
        }
    });
}
```

## 7.7 SDK 批处理模式

通过 pi SDK 在脚本中使用 Code Review 功能，无需交互式界面：

```typescript
// src/sdk-example.ts
import { createAgentSession } from "@earendil-works/pi-coding-agent";
import { getModel } from "@earendil-works/pi-ai";

async function batchReviewPR(prIdentifier: string) {
    console.log(`Reviewing PR: ${prIdentifier}`);

    const { session } = await createAgentSession({
        model: getModel("openai", "gpt-4o"),
        tools: ["review_pr", "list_comments", "post_review"],
        sessionManager: SessionManager.inMemory(),
    });

    // 订阅事件以捕获输出
    const messages: string[] = [];
    session.subscribe("message_end", (event: any) => {
        if (event.message.role === "assistant") {
            const text = event.message.content
                .filter((c: any) => c.type === "text")
                .map((c: any) => c.text)
                .join("");
            messages.push(text);
        }
    });

    // 执行 Review
    await session.enqueueUserMessage(
        `Review PR ${prIdentifier}. Focus on security and performance.`
    );
    await session.runPrompt();

    // 输出结果
    console.log("=== Review Results ===");
    for (const msg of messages) {
        console.log(msg);
    }
}

// 运行
const pr = process.argv[2];
if (!pr) {
    console.error("Usage: npx tsx sdk-example.ts owner/repo#123");
    process.exit(1);
}
batchReviewPR(pr).catch(console.error);
```

## 7.8 测试

### 单元测试（使用 Faux Provider）

pi 的测试基础设施使用 **Faux Provider**（模拟 LLM）进行测试，无需真实 API 调用：

```typescript
// test/review-pr.test.ts
import { describe, it, expect, vi } from "vitest";
import { type AssistantMessage, createFauxProvider } from "@earendil-works/pi-ai";
import { createAgentSession } from "@earendil-works/pi-coding-agent";

describe("review_pr tool", () => {
    it("should parse GitHub PR URLs correctly", async () => {
        const { default: reviewExtension } = await import("../src/index.ts");

        // 创建会话，使用 Faux Provider
        const { session } = await createAgentSession({
            extensionFactories: [(pi) => reviewExtension(pi)],
        });

        // 模拟 GitHub API
        const mockFetch = vi.fn();
        globalThis.fetch = mockFetch;

        mockFetch.mockResolvedValueOnce({
            ok: true,
            json: async () => ({
                number: 42,
                title: "Fix login bug",
                body: "This PR fixes a login validation issue",
                additions: 100,
                deletions: 20,
                merged: false,
                draft: false,
                head: { label: "user:fix-login" },
                base: { label: "main" },
            }),
        });

        mockFetch.mockResolvedValueOnce({
            ok: true,
            json: async () => ([{
                filename: "src/auth.ts",
                status: "modified",
                additions: 50,
                deletions: 10,
                changes: 60,
                patch: "@@ -1,5 +1,7 @@\n+import { validateInput } from './validation';\n function login() {\n-  const input = getInput();\n+  const input = sanitizeInput(getInput());\n   if (validate(input)) {\n     authenticate(input);\n   }\n }",
            }]),
        });

        mockFetch.mockResolvedValueOnce({
            ok: true,
            json: async () => ([]),
        });

        mockFetch.mockResolvedValueOnce({
            ok: true,
            json: async () => ([]),
        });

        // 发送用户消息触发 review
        // 注意：这个测试验证的是工具定义本身，而不是完整的 LLM 调用
        const tool = reviewExtension.getTool("review_pr");
        expect(tool.name).toBe("review_pr");
        expect(tool.label).toBe("Review PR");

        const result = await tool.execute(
            "test-call-id",
            { pr: "owner/repo#42", focus: "security" },
            undefined,
            () => {},
            {} as any,
        );

        expect(result.content[0].type).toBe("text");
        expect(result.content[0].text).toContain("PR #42");
        expect(result.content[0].text).toContain("Fix login bug");
        expect(result.content[0].text).toContain("src/auth.ts");
    });
});
```

### 集成测试

通过 Faux Provider 测试完整的 Review 工作流：

```typescript
// test/review-workflow.test.ts
import { describe, it, expect, beforeAll } from "vitest";

describe("PR Review Workflow", () => {
    it("complete review cycle: fetch -> analyze -> post", async () => {
        // 使用 Faux Provider 模拟 LLM 回复
        const fauxProvider = createFauxProvider({
            response: {
                role: "assistant",
                content: [
                    {
                        type: "toolCall",
                        id: "call_1",
                        name: "review_pr",
                        arguments: { pr: "owner/repo#42" },
                    },
                    {
                        type: "text",
                        text: "I've reviewed the PR. The code quality is good overall. I'll post my review comments.",
                    },
                    {
                        type: "toolCall",
                        id: "call_2",
                        name: "post_review",
                        arguments: {
                            pr: "owner/repo#42",
                            body: "## Review Summary\n\nCode quality: Good\n\n**Issues Found:**\n1. Input validation should be centralized\n2. Add error handling for edge cases\n\nOverall: Needs minor changes.",
                            conclusion: "COMMENT",
                        },
                    },
                ],
                stopReason: "toolUse",
                usage: { input: 100, output: 200, cacheRead: 0, cacheWrite: 0, totalTokens: 300 },
            },
        });

        const { session } = await createAgentSession({
            // 使用 Faux Provider
            streamFn: fauxProvider.stream,
            // 注册扩展
            extensionFactories: [(await import("../src/index.ts")).default],
        });

        // 模拟 GitHub API
        globalThis.fetch = async (url: string) => {
            if (url.includes("/pulls/42")) {
                return { ok: true, json: async () => ({ number: 42, title: "Test PR", body: "Test", additions: 10, deletions: 5, merged: false, draft: false, head: { label: "test" }, base: { label: "main" } }) };
            }
            if (url.includes("/files")) {
                return { ok: true, json: async () => ([{ filename: "test.ts", status: "modified", additions: 5, deletions: 3, changes: 8, patch: "@@ -1 +1,5 @@\n+new code" }]) };
            }
            if (url.includes("/reviews") && options.method === "POST") {
                return { ok: true, json: async () => ({ id: 1, body: "Review body", state: options.body ? JSON.parse(options.body).event : "COMMENTED" }) };
            }
            return { ok: true, json: async () => ([]) };
        };

        // 执行
        await session.enqueueUserMessage("Review PR owner/repo#42");
        const messages = await session.runPrompt();

        // 验证工具被正确调用
        const toolCalls = messages.filter(
            (m: any) => m.role === "assistant" && m.content.some((c: any) => c.type === "toolCall"),
        );
        expect(toolCalls.length).toBeGreaterThan(0);
    });
});
```

## 7.9 知识点回顾

这个实战项目覆盖了本书前 6 章的主要知识点：

| 知识点 | 所属章节 | 在项目中的体现 |
|--------|---------|--------------|
| Component 接口 | 第 2 章 pi-tui | `renderReviewResult()` 自定义渲染 |
| EventStream 协议 | 第 3 章 pi-ai | 扩展监听 `context` 事件注入提示词 |
| Provider Registry | 第 3 章 pi-ai | 扩展可以通过 `registerProvider` 添加自定义 Provider |
| Agent Loop | 第 4 章 agent-core | `review_pr` → `post_review` 的多步工作流 |
| Tool 执行引擎 | 第 4 章 agent-core | 工具定义遵循 `AgentTool` 接口 |
| AgentSession 管理 | 第 5 章 coding-agent | `createAgentSession()` SDK 调用 |
| 扩展系统 | 第 6 章 扩展系统 | `registerTool`、`registerCommand`、`on("context")` |
| 自定义渲染 | 第 6 章 扩展系统 | `renderResult` 方法 |
| Faux Provider 测试 | 第 8 章 工程实践 | 使用 `createFauxProvider` 进行集成测试 |

## 7.10 章节小结

通过构建 pi-code-review-assistant 扩展，我们实践了 pi 的完整扩展开发流程：

1. **工具定义**：使用 `defineTool` 和 typebox Schema 注册三个互相关联的工具
2. **扩展入口**：在 `index.ts` 中注册工具、命令和事件监听器
3. **UI 渲染**：自定义工具结果的 TUI 渲染
4. **SDK 集成**：通过 `createAgentSession` 在脚本中批处理使用
5. **测试**：使用 Faux Provider 进行单元测试和集成测试

这个项目展示了如何利用 pi 的扩展系统，将外部 API（GitHub）无缝集成到 AI Agent 的工作流中，实现端到端的代码审查自动化。
