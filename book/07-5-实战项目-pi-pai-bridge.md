# 实战项目五：pi-pai-bridge — pi 作为 PAI 的多模型编码引擎

## 项目概述

[Personal AI Infrastructure (PAI)](https://github.com/danielmiessler/Personal_AI_Infrastructure) 是一个 **Life Operating System** — 它捕获你是谁、你在意什么、你要去哪里，然后用 AI 帮助你达成理想状态。PAI 原生构建在 **Claude Code** 之上。

而 **pi** 是一个多 Provider 的 AI 编程代理。这个项目的目标是将两者深度整合：

```
PAI (Life OS)                          pi (Coding Agent)
═══════════════                        ════════════════
  PAI/USER/TELOS/     ───context──▶   注入 pi 的系统提示词
  PAI/MEMORY/         ◀──写入─────   编码会话后记录学习
  PULSE (localhost:31337) ◀──推送───   pi RPC 发送会话统计
  PAI/skills/         ───加载─────▶   注册为 pi 命令/工具
  DA (Digital Assistant) ◀──路由───   通过 pi 调用任意模型
```

### 为什么要做这个集成？

| PAI 的优势 | pi 的补充 |
|-----------|----------|
| 丰富的个人上下文（TELOS、记忆） | 多 Provider 支持（不限于 Claude） |
| Pulse 生活仪表盘 | 扩展系统 + RPC 模式 |
| 技能系统（SKILL.md） | 工具执行引擎 + 会话管理 |
| 算法框架（ISA） | TUI 交互体验 |
| DA 数字助理人格 | SDK 程序化调用 |

集成后：**PAI 提供「你是谁 + 你要去哪」，pi 提供「用哪个模型 + 怎么执行」**。

## 项目结构

```
pi-pai-bridge/
├── src/
│   ├── index.ts                          # 扩展入口
│   ├── pai-context-loader.ts             # 读取 PAI 上下文
│   ├── pai-memory-tool.ts                # 读写 PAI 记忆的工具
│   ├── pai-skills-loader.ts              # 加载 PAI 技能
│   ├── pai-pulse-bridge.ts              # 推送到 Pulse 仪表盘
│   └── pai-sdk-runner.ts                # SDK 批处理封装
├── package.json
└── README.md
```

## PAI 目录结构速览

```
~/.claude/PAI/                     ← PAI 的根目录
├── ALGORITHM/                     ← 算法定义（v6.3.0，7 阶段循环）
│   └── v6.3.0.md
├── USER/
│   ├── TELOS/                     ← 使命、目标、信念、智慧
│   │   ├── MISSION.md
│   │   ├── GOALS.md
│   │   ├── BELIEFS.md
│   │   └── WISDOM.md
│   ├── DA_IDENTITY.md             ← 数字助理人格定义
│   └── PREFERENCES.md             ← 偏好设置
├── MEMORY/                        ← 三层记忆系统
│   ├── WORK/{slug}/               ← 工作记忆
│   ├── KNOWLEDGE/                 ← 知识记忆
│   └── LEARNING/                  ← 学习记忆
├── skills/                        ← 技能定义
│   ├── ISA/                       ← 理想状态制品技能
│   ├── Fabric/                    ← fabric patterns
│   └── ...
├── PULSE/                         ← 生活仪表盘
│   ├── server.ts                  ← Pulse 服务
│   └── ...
└── bin/                           ← 工具脚本
    ├── inference.ts               ← 推理封装
    └── ...
```

## 第一步：PAI 上下文加载器

pi 扩展的核心功能：读取 PAI 的个人上下文文件，注入到 pi 的系统提示词中。

```typescript
// src/pai-context-loader.ts
import * as fs from "node:fs";
import * as path from "node:path";

export interface PaiContext {
    telos: {
        mission: string;
        goals: string;
        beliefs: string;
        wisdom: string;
    };
    daIdentity: string;
    preferences: string;
    recentMemory: string[];
}

const PAI_DIR = path.join(process.env.HOME || "~", ".claude", "PAI");

/**
 * 读取 PAI 的 TELOS 目录
 */
function readTelosDir(telosDir: string): PaiContext["telos"] {
    const telos: PaiContext["telos"] = {
        mission: "",
        goals: "",
        beliefs: "",
        wisdom: "",
    };

    try {
        const files = fs.readdirSync(telosDir);
        for (const file of files) {
            const content = fs.readFileSync(path.join(telosDir, file), "utf-8");
            const name = path.basename(file, ".md").toLowerCase();
            switch (name) {
                case "mission": telos.mission = content; break;
                case "goals": telos.goals = content; break;
                case "beliefs": telos.beliefs = content; break;
                case "wisdom": telos.wisdom = content; break;
            }
        }
    } catch {}

    return telos;
}

/**
 * 读取最近的 MEMORY 条目
 */
function readRecentMemory(memoryDir: string, maxEntries = 5): string[] {
    const memories: string[] = [];

    try {
        // 读取 WORK 目录下的最近工作记录
        const workDir = path.join(memoryDir, "WORK");
        if (fs.existsSync(workDir)) {
            const entries = fs.readdirSync(workDir)
                .map(name => ({ name, time: fs.statSync(path.join(workDir, name)).mtimeMs }))
                .sort((a, b) => b.time - a.time)
                .slice(0, maxEntries);

            for (const entry of entries) {
                const entryDir = path.join(workDir, entry.name);
                const summaryFile = path.join(entryDir, "summary.md");
                if (fs.existsSync(summaryFile)) {
                    memories.push(`[WORK: ${entry.name}]\n${fs.readFileSync(summaryFile, "utf-8")}`);
                }
            }
        }

        // 读取 LEARNING 中的经验教训
        const learningDir = path.join(memoryDir, "LEARNING");
        if (fs.existsSync(learningDir)) {
            const learnings = fs.readdirSync(learningDir)
                .sort()
                .slice(-maxEntries);

            for (const file of learnings) {
                const content = fs.readFileSync(path.join(learningDir, file), "utf-8");
                memories.push(`[LEARNING: ${file}]\n${content}`);
            }
        }
    } catch {}

    return memories;
}

/**
 * 加载完整的 PAI 上下文
 */
export function loadPaiContext(): PaiContext | null {
    if (!fs.existsSync(PAI_DIR)) {
        return null;
    }

    const userDir = path.join(PAI_DIR, "USER");
    const memoryDir = path.join(PAI_DIR, "MEMORY");

    try {
        return {
            telos: readTelosDir(path.join(userDir, "TELOS")),
            daIdentity: fs.readFileSync(path.join(userDir, "DA_IDENTITY.md"), "utf-8"),
            preferences: fs.readFileSync(path.join(userDir, "PREFERENCES.md"), "utf-8"),
            recentMemory: readRecentMemory(memoryDir),
        };
    } catch {
        return null;
    }
}

/**
 * 将 PAI 上下文格式化为系统提示词片段
 */
export function formatPaiContextForPrompt(ctx: PaiContext): string {
    const sections: string[] = [];

    sections.push(`<pai_context>`);

    // TELOS
    if (ctx.telos.mission) {
        sections.push(`<telos_mission>${ctx.telos.mission}</telos_mission>`);
    }
    if (ctx.telos.goals) {
        sections.push(`<telos_goals>${ctx.telos.goals}</telos_goals>`);
    }

    // DA 身份
    if (ctx.daIdentity) {
        // 提取 DA 的名字和人格描述
        const nameMatch = ctx.daIdentity.match(/^#\s+(.+)$/m);
        const name = nameMatch ? nameMatch[1].trim() : "DA";
        const personalitySection = ctx.daIdentity.split(/^##\s+/m)
            .find(s => s.startsWith("PERSONALITY") || s.startsWith("Personality"));
        const personality = personalitySection
            ? personalitySection.split("\n").slice(1).join("\n").trim()
            : "";

        sections.push(`<da_identity name="${name}">${personality}</da_identity>`);
    }

    // 最近记忆
    if (ctx.recentMemory.length > 0) {
        sections.push(`<recent_memory>`);
        for (const mem of ctx.recentMemory) {
            sections.push(mem);
        }
        sections.push(`</recent_memory>`);
    }

    sections.push(`</pai_context>`);

    // 生成系统提示词指令
    sections.push(`
## PAI Integration Instructions

You are operating within the user's Personal AI Infrastructure (PAI) Life OS.
The context above contains the user's mission, goals, and recent work memory.

When working on tasks:
1. Consider the user's TELOS (mission/goals) when making decisions
2. Reference relevant past work from MEMORY to avoid repeating mistakes
3. After completing significant work, record learnings back to PAI memory
4. Use the pai_memory tool to read/write PAI's memory system
`);

    return sections.join("\n");
}
```

## 第二步：PAI 记忆工具

注册一个 pi 工具，让 LLM 可以读取和写入 PAI 的记忆系统：

```typescript
// src/pai-memory-tool.ts
import * as fs from "node:fs";
import * as path from "node:path";
import { StringEnum, Type } from "@earendil-works/pi-ai";
import { defineTool } from "@earendil-works/pi-coding-agent";

const PAI_MEMORY_DIR = path.join(process.env.HOME || "~", ".claude", "PAI", "MEMORY");

function ensureDir(dir: string): void {
    if (!fs.existsSync(dir)) {
        fs.mkdirSync(dir, { recursive: true });
    }
}

function slugify(text: string): string {
    return text.toLowerCase()
        .replace(/[^a-z0-9]+/g, "-")
        .replace(/^-|-$/g, "")
        .slice(0, 80);
}

export const paiMemoryTool = defineTool({
    name: "pai_memory",
    label: "PAI Memory",
    description: `Read from or write to PAI's three-tier memory system.
    - WORK: records of completed work sessions
    - KNOWLEDGE: persistent knowledge and reference material
    - LEARNING: lessons learned, insights, and improvements

    Use this tool to persist important context across sessions.
    PAI memory lives at ~/.claude/PAI/MEMORY/`,
    parameters: Type.Object({
        action: StringEnum(["read", "write", "search"] as const),
        tier: StringEnum(["WORK", "KNOWLEDGE", "LEARNING"] as const),
        slug: Type.Optional(Type.String({
            description: "Unique identifier for the memory entry. Auto-generated from title if not provided.",
        })),
        title: Type.Optional(Type.String({
            description: "Title for write operations. Used to generate slug if slug is not provided.",
        })),
        content: Type.Optional(Type.String({
            description: "Content to write (for write action). Markdown format.",
        })),
        query: Type.Optional(Type.String({
            description: "Search query (for search action). Uses ripgrep-style matching.",
        })),
    }),

    async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
        const tierDir = path.join(PAI_MEMORY_DIR, params.tier);

        switch (params.action) {
            case "write": {
                ensureDir(tierDir);
                const slug = params.slug || slugify(params.title || "untitled");
                const entryDir = path.join(tierDir, slug);
                ensureDir(entryDir);

                const timestamp = new Date().toISOString();
                const frontmatter = `---
title: ${params.title || slug}
date: ${timestamp}
type: ${params.tier}
---

`;
                const filePath = path.join(entryDir, "entry.md");
                fs.writeFileSync(filePath, frontmatter + (params.content || ""));

                return {
                    content: [{ type: "text", text: `Written to PAI memory: ${params.tier}/${slug}` }],
                    details: { path: filePath, tier: params.tier, slug },
                };
            }

            case "read": {
                if (!params.slug) {
                    // 列出该 tier 下所有条目
                    ensureDir(tierDir);
                    const entries = fs.readdirSync(tierDir);
                    const summary = entries.length > 0
                        ? `Entries in ${params.tier}:\n` + entries.map(e => `  - ${e}`).join("\n")
                        : `No entries in ${params.tier}`;
                    return {
                        content: [{ type: "text", text: summary }],
                        details: { entries },
                    };
                }

                const entryPath = path.join(tierDir, params.slug, "entry.md");
                if (!fs.existsSync(entryPath)) {
                    return {
                        content: [{ type: "text", text: `Memory entry not found: ${params.tier}/${params.slug}` }],
                        details: { error: "not_found" },
                    };
                }

                const content = fs.readFileSync(entryPath, "utf-8");
                return {
                    content: [{ type: "text", text: `# ${params.tier}/${params.slug}\n\n${content}` }],
                    details: { path: entryPath, size: content.length },
                };
            }

            case "search": {
                if (!params.query) {
                    return {
                        content: [{ type: "text", text: "Query required for search action" }],
                        details: { error: "query_required" },
                    };
                }

                ensureDir(tierDir);
                const results: Array<{ slug: string; match: string }> = [];
                const entries = fs.readdirSync(tierDir);

                for (const entry of entries) {
                    const filePath = path.join(tierDir, entry, "entry.md");
                    if (!fs.existsSync(filePath)) continue;
                    const content = fs.readFileSync(filePath, "utf-8");
                    if (content.toLowerCase().includes(params.query.toLowerCase())) {
                        // 提取匹配内容的上下文
                        const lines = content.split("\n");
                        const matchLines = lines.filter(l =>
                            l.toLowerCase().includes(params.query!.toLowerCase())
                        );
                        results.push({
                            slug: entry,
                            match: matchLines.slice(0, 3).join("\n"),
                        });
                    }
                }

                if (results.length === 0) {
                    return {
                        content: [{ type: "text", text: `No matches in ${params.tier} for "${params.query}"` }],
                        details: { results: [] },
                    };
                }

                const output = `Found ${results.length} result(s) in ${params.tier}:\n\n` +
                    results.map(r => `### ${r.slug}\n${r.match}`).join("\n\n");

                return {
                    content: [{ type: "text", text: output }],
                    details: { results },
                };
            }

            default:
                return {
                    content: [{ type: "text", text: `Unknown action: ${params.action}` }],
                    details: { error: "unknown_action" },
                };
        }
    },
});
```

## 第三步：PAI 技能加载器

PAI 包含大量技能定义（SKILL.md）。pi 可以加载这些技能，注册为 pi 命令：

```typescript
// src/pai-skills-loader.ts
import * as fs from "node:fs";
import * as path from "node:path";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

interface PaiSkill {
    name: string;
    description: string;
    filePath: string;
    content: string;
}

const PAI_SKILLS_DIRS = [
    path.join(process.env.HOME || "~", ".claude", "skills"),
    path.join(process.env.HOME || "~", ".claude", "PAI", "skills"),
];

/**
 * 扫描 PAI 技能目录，加载所有 SKILL.md
 */
export function discoverPaiSkills(): PaiSkill[] {
    const skills: PaiSkill[] = [];

    for (const skillsDir of PAI_SKILLS_DIRS) {
        if (!fs.existsSync(skillsDir)) continue;

        try {
            const entries = fs.readdirSync(skillsDir, { withFileTypes: true });
            for (const entry of entries) {
                if (!entry.isDirectory()) continue;

                const skillFile = path.join(skillsDir, entry.name, "SKILL.md");
                if (!fs.existsSync(skillFile)) {
                    // 也检查 index.ts
                    const indexFile = path.join(skillsDir, entry.name, "index.ts");
                    if (fs.existsSync(indexFile)) continue;
                    // 无 SKILL.md 也不是扩展，跳过
                    continue;
                }

                const content = fs.readFileSync(skillFile, "utf-8");
                // 解析 frontmatter
                const nameMatch = content.match(/^name:\s*(.+)$/m);
                const descMatch = content.match(/^description:\s*(.+)$/m);

                skills.push({
                    name: nameMatch?.[1]?.trim() || entry.name,
                    description: descMatch?.[1]?.trim() || `PAI skill: ${entry.name}`,
                    filePath: skillFile,
                    content,
                });
            }
        } catch {}
    }

    return skills;
}

/**
 * 将 PAI 技能注册为 pi 斜杠命令
 */
export function registerPaiSkills(pi: ExtensionAPI, skills: PaiSkill[]): void {
    for (const skill of skills) {
        pi.registerCommand(`pai:${skill.name.toLowerCase().replace(/\s+/g, "-")}`, {
            description: `[PAI] ${skill.description}`,
            handler: async (args, ctx) => {
                // 在交互模式下显示技能内容
                if (ctx.hasUI) {
                    ctx.ui.notify(`Loaded PAI skill: ${skill.name}`, "info");

                    // 将技能内容作为系统提示注入，然后让用户输入
                    const userInput = await ctx.ui.input(
                        `PAI Skill: ${skill.name}`,
                        `Enter your input for this skill...`,
                    );

                    if (userInput) {
                        // 技能 + 用户输入作为新消息发送给 Agent
                        ctx.sendUserMessage(
                            `I want to use the "${skill.name}" skill.\n\nSkill instructions:\n${skill.content}\n\nMy input:\n${userInput}`,
                            { deliverAs: "steer" },
                        );
                    }
                } else {
                    ctx.ui.notify(`/${skill.name} requires interactive mode`, "error");
                }
            },
        });
    }
}

/**
 * 将 PAI 技能注入系统提示词（让 LLM 知道这些技能的存在）
 */
export function formatSkillsForPrompt(skills: PaiSkill[]): string {
    if (skills.length === 0) return "";

    const skillList = skills.map(s =>
        `  - ${s.name}: ${s.description}`
    ).join("\n");

    return `
## Available PAI Skills

The following PAI skills are available. You can invoke them by describing what you need.

${skillList}

To use a skill, describe the task and reference the skill name.
`;
}
```

## 第四步：Pulse 仪表盘桥接

PAI 的 Pulse 是一个 Life Dashboard，运行在 `localhost:31337`。pi 可以将会话统计数据推送到 Pulse：

```typescript
// src/pai-pulse-bridge.ts
const PULSE_API = "http://localhost:31337/api/pulse";

interface PiSessionSummary {
    sessionId: string;
    provider: string;
    model: string;
    messageCount: number;
    totalInputTokens: number;
    totalOutputTokens: number;
    totalCost: number;
    toolsUsed: Record<string, number>;
    startTime: string;
    endTime: string;
    summary: string;
}

/**
 * 将会话摘要推送到 PAI Pulse
 */
export async function pushToPulse(summary: PiSessionSummary): Promise<boolean> {
    try {
        const response = await fetch(`${PULSE_API}/activity`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                source: "pi-coding-agent",
                type: "coding_session",
                timestamp: new Date().toISOString(),
                data: summary,
            }),
        });
        return response.ok;
    } catch {
        // Pulse 可能未运行，静默失败
        return false;
    }
}

/**
 * 从 pi 的会话管理器收集统计数据，格式化为 Pulse 格式
 */
export async function collectSessionSummary(
    ctx: { sessionManager: any; model?: { provider: string; id: string } },
): Promise<PiSessionSummary> {
    const branch = ctx.sessionManager.getBranch();
    const messages = branch.filter((e: any) => e.type === "message").map((e: any) => e.message);

    const toolFrequency: Record<string, number> = {};
    let totalInput = 0;
    let totalOutput = 0;
    let totalCost = 0;

    for (const msg of messages) {
        if (msg.role === "assistant" && msg.usage) {
            totalInput += msg.usage.input || 0;
            totalOutput += msg.usage.output || 0;
            totalCost += msg.usage.cost?.total || 0;

            const toolCalls = msg.content?.filter((c: any) => c.type === "toolCall") || [];
            for (const tc of toolCalls) {
                toolFrequency[tc.name] = (toolFrequency[tc.name] || 0) + 1;
            }
        }
    }

    const startTime = messages.length > 0
        ? new Date(messages[0].timestamp).toISOString()
        : new Date().toISOString();
    const endTime = messages.length > 0
        ? new Date(messages[messages.length - 1].timestamp).toISOString()
        : new Date().toISOString();

    return {
        sessionId: branch[0]?.id || "unknown",
        provider: ctx.model?.provider || "unknown",
        model: ctx.model?.id || "unknown",
        messageCount: messages.length,
        totalInputTokens: totalInput,
        totalOutputTokens: totalOutput,
        totalCost,
        toolsUsed: toolFrequency,
        startTime,
        endTime,
        summary: `${messages.length} messages, ${totalInput + totalOutput} tokens, $${totalCost.toFixed(6)} cost`,
    };
}
```

## 第五步：扩展入口

将所有模块组装为完整的 pi 扩展：

```typescript
// src/index.ts
import type { ExtensionAPI, ExtensionContext } from "@earendil-works/pi-coding-agent";
import { loadPaiContext, formatPaiContextForPrompt } from "./pai-context-loader.ts";
import { paiMemoryTool } from "./pai-memory-tool.ts";
import { discoverPaiSkills, registerPaiSkills, formatSkillsForPrompt } from "./pai-skills-loader.ts";
import { pushToPulse, collectSessionSummary } from "./pai-pulse-bridge.ts";

export default function (pi: ExtensionAPI) {
    // ====================================================================
    // 1. 加载 PAI 上下文
    // ====================================================================

    const paiCtx = loadPaiContext();
    if (!paiCtx) {
        console.warn("PAI context not found. Install PAI first: curl -sSL https://ourpai.ai/install.sh | bash");
    } else {
        console.log(`PAI context loaded: ${paiCtx.daIdentity ? "DA identified" : "no DA"}`);
    }

    // ====================================================================
    // 2. 注册 PAI 工具
    // ====================================================================

    pi.registerTool(paiMemoryTool);

    // ====================================================================
    // 3. 加载 PAI 技能
    // ====================================================================

    const paiSkills = paiCtx ? discoverPaiSkills() : [];
    if (paiSkills.length > 0) {
        registerPaiSkills(pi, paiSkills);
        console.log(`Loaded ${paiSkills.length} PAI skills as commands`);
    }

    // ====================================================================
    // 4. 注入 PAI 上下文到系统提示词
    // ====================================================================

    pi.on("before_agent_start", async (event, ctx) => {
        if (!paiCtx) return;

        // 将 PAI 上下文注入系统提示词
        const paiPrompt = formatPaiContextForPrompt(paiCtx);

        // 如果有技能，也注入技能列表
        if (paiSkills.length > 0) {
            event.actions.appendSystemPrompt(
                paiPrompt + "\n" + formatSkillsForPrompt(paiSkills)
            );
        } else {
            event.actions.appendSystemPrompt(paiPrompt);
        }
    });

    // ====================================================================
    // 5. 在每个 turn 结束时做记忆回写 + Pulse 推送
    // ====================================================================

    pi.on("turn_end", async (event, ctx) => {
        if (!paiCtx) return;

        const msg = event.message;
        if (msg.role !== "assistant" || msg.stopReason === "error") return;

        // 检查是否有工具调用（表示有实质性的工作）
        const hasToolCalls = msg.content.some((c: any) => c.type === "toolCall");
        if (!hasToolCalls) return;

        // 提取 LLM 回复中的总结性内容
        const textContent = msg.content
            .filter((c: any) => c.type === "text")
            .map((c: any) => c.text)
            .join("\n");

        // 如果有实质性的工作成果，写入 PAI 的 WORK 记忆
        if (textContent.length > 100) {
            const slug = `pi-session-${Date.now()}`;
            const summary = textContent.slice(0, 2000);
            const entryDir = path.join(
                process.env.HOME || "~",
                ".claude", "PAI", "MEMORY", "WORK", slug
            );
            fs.mkdirSync(entryDir, { recursive: true });
            fs.writeFileSync(
                path.join(entryDir, "summary.md"),
                `# Coding Session Summary\n\nDate: ${new Date().toISOString()}\n\n${summary}`
            );
        }
    });

    // ====================================================================
    // 6. 会话结束时推送到 Pulse
    // ====================================================================

    pi.on("agent_end", async (_event, ctx) => {
        if (!paiCtx) return;

        try {
            const summary = await collectSessionSummary(ctx);
            const pushed = await pushToPulse(summary);
            if (pushed) {
                console.log("Session stats pushed to PAI Pulse dashboard");
            }
        } catch {}
    });

    // ====================================================================
    // 7. /pai 命令 — 查看 PAI 状态
    // ====================================================================

    pi.registerCommand("pai", {
        description: "Show PAI integration status",
        handler: async (_args, ctx) => {
            if (!ctx.hasUI) return;

            if (!paiCtx) {
                ctx.ui.notify(
                    "PAI not found. Install: curl -sSL https://ourpai.ai/install.sh | bash",
                    "warning",
                );
                return;
            }

            const lines: string[] = [];
            lines.push("=== PAI Integration Status ===");
            lines.push("");
            lines.push(`TELOS: ${paiCtx.telos.mission ? "loaded" : "not found"}`);
            lines.push(`DA Identity: ${paiCtx.daIdentity ? "loaded" : "not found"}`);
            lines.push(`Recent Memories: ${paiCtx.recentMemory.length}`);
            lines.push(`Skills Loaded: ${paiSkills.length}`);
            lines.push("");

            // 检查 Pulse 是否运行
            try {
                const response = await fetch("http://localhost:31337/api/pulse/health");
                if (response.ok) {
                    lines.push("Pulse Dashboard: running at http://localhost:31337");
                }
            } catch {
                lines.push("Pulse Dashboard: not detected");
            }

            ctx.ui.notify(lines.join("\n"), "info");
        },
    });
}
```

## SDK 封装：pi run with PAI context

通过 SDK 在脚本中使用 pi + PAI：

```typescript
// src/pai-sdk-runner.ts
import { createAgentSession } from "@earendil-works/pi-coding-agent";
import { loadPaiContext, formatPaiContextForPrompt } from "./pai-context-loader.ts";
import { discoverPaiSkills, formatSkillsForPrompt } from "./pai-skills-loader.ts";

export interface PaiRunOptions {
    provider?: string;
    modelId?: string;
    thinkingLevel?: "off" | "low" | "medium" | "high";
    systemPrompt?: string;
}

/**
 * 以 PAI 上下文增强的方式运行 pi 会话
 */
export async function paiRun(
    prompt: string,
    options: PaiRunOptions = {},
) {
    // 1. 加载 PAI 上下文
    const paiCtx = loadPaiContext();
    const paiSkills = paiCtx ? discoverPaiSkills() : [];

    // 2. 构建增强的系统提示词
    let systemPrompt = options.systemPrompt || "";

    if (paiCtx) {
        systemPrompt += "\n\n" + formatPaiContextForPrompt(paiCtx);
    }
    if (paiSkills.length > 0) {
        systemPrompt += "\n\n" + formatSkillsForPrompt(paiSkills);
    }

    // 3. 创建 pi 会话
    const { session } = await createAgentSession({
        model: options.provider
            ? { id: options.modelId || "default", provider: options.provider, /* ... */ } as any
            : undefined,
        thinkingLevel: options.thinkingLevel,
    });

    // 4. 设置系统提示词
    if (systemPrompt) {
        session.state.systemPrompt = systemPrompt;
    }

    // 5. 执行
    console.log(`\npi + PAI: processing with ${paiCtx ? "PAI context" : "no PAI context"}...\n`);

    const messages: string[] = [];
    session.subscribe("message_end", (event: any) => {
        if (event.message.role === "assistant") {
            const text = event.message.content
                .filter((c: any) => c.type === "text")
                .map((c: any) => c.text)
                .join("");
            if (text) messages.push(text);
        }
    });

    await session.enqueueUserMessage(prompt);
    await session.runPrompt();

    return { messages, paiContext: !!paiCtx, skillCount: paiSkills.length };
}

// CLI 入口
if (import.meta.main) {
    const prompt = process.argv[2];
    if (!prompt) {
        console.error("Usage: npx tsx pai-sdk-runner.ts <prompt>");
        process.exit(1);
    }

    paiRun(prompt, {
        provider: process.env.PI_PROVIDER,
        modelId: process.env.PI_MODEL,
    }).then((result) => {
        console.log("\n=== Response ===");
        for (const msg of result.messages) {
            console.log(msg);
        }
        console.log(`\n--- PAI: ${result.paiContext}, Skills: ${result.skillCount} ---`);
    });
}
```

## 安装与使用

```bash
# 1. 先安装 PAI（如果还没装）
curl -sSL https://ourpai.ai/install.sh | bash

# 2. 安装 pi-pai-bridge 扩展
mkdir -p ~/.pi/agent/extensions/pi-pai-bridge
# 将本项目复制到该目录

# 3. 启动 pi，自动加载 PAI 上下文
pi

# 4. 查看集成状态
# /pai

# 5. 使用 PAI 技能
# /pai:isa

# 6. 使用 SDK 批处理模式
PI_PROVIDER=openai PI_MODEL=gpt-4o npx tsx pai-sdk-runner.ts \
  "根据我的 TELOS 目标，规划本周的编码工作"
```

## PAI 与 pi 的双向数据流

```
                    ┌─────────────────────────┐
                    │   PAI Life OS           │
                    │   ~/.claude/PAI/        │
                    │                         │
                    │  ┌─────────────────┐    │
                    │  │ TELOS (使命/目标) │◀──┼─── 注入系统提示词
                    │  └─────────────────┘    │
                    │  ┌─────────────────┐    │
                    │  │ MEMORY (记忆)    │◀──┼─── 读写编码会话记忆
                    │  └─────────────────┘    │
                    │  ┌─────────────────┐    │
                    │  │ skills/ (技能)   │◀──┼─── 注册为 pi 命令
                    │  └─────────────────┘    │
                    │  ┌─────────────────┐    │
                    │  │ PULSE (仪表盘)   │◀──┼─── 推送会话统计
                    │  └─────────────────┘    │
                    └──────────┬──────────────┘
                               │
                               ▼
                    ┌─────────────────────────┐
                    │   pi Coding Agent       │
                    │                         │
                    │  ├── 多 Provider 支持    │
                    │  ├── TUI 交互界面        │
                    │  ├── 工具执行引擎        │
                    │  ├── 会话管理（分支/压缩) │
                    │  └── RPC 外部集成        │
                    └─────────────────────────┘
```

## 技术覆盖对照

| 技术领域 | 本项目覆盖程度 | 说明 |
|---------|:-------------:|------|
| pi Extension 入口 | ★★★★★ | 多事件监听：`before_agent_start`、`turn_end`、`agent_end` |
| pi 自定义工具 | ★★★★★ | `pai_memory` 工具：读写搜索三层记忆 |
| pi 命令注册 | ★★★★★ | `/pai` 命令 + 自动注册 PAI 技能为 `/pai:*` 命令 |
| pi 系统提示词注入 | ★★★★★ | `event.actions.appendSystemPrompt()` 注入 TELOS/DA/Memory |
| pi SDK 程序化调用 | ★★★★★ | `pai-sdk-runner.ts` 封装 |
| pi RPC 数据推送 | ★★★★ | HTTP 推送会话统计到 Pulse |
| PAI TELOS 解析 | ★★★★★ | 读取 MISSION/GOALS/BELIEFS/WISDOM |
| PAI MEMORY 读写 | ★★★★★ | WORK/KNOWLEDGE/LEARNING 三层 |
| PAI SKILL.md 加载 | ★★★★★ | 发现、解析 frontmatter、注册命令 |
| PAI Pulse 集成 | ★★★★ | `api/pulse/activity` 端点推送 |
| 跨系统上下文传递 | ★★★★★ | PAI → pi 系统提示词 → LLM → 工具调用 → PAI 记忆回写 |

## 章节小结

pi-pai-bridge 是两个项目的深度整合，展示了：

1. **PAI 作为 Life OS** 提供"你是谁、你要去哪"的元上下文
2. **pi 作为多模型编码引擎** 提供"用哪个模型、怎么执行"的执行层
3. **双向数据流**：PAI 上下文 → pi 系统提示词 → LLM 推理 → 工具执行 → PAI 记忆回写 → Pulse 仪表盘
4. **技能互通**：PAI 的 SKILL.md 技能文件可被 pi 发现并注册为命令
5. **多模型灵活性**：pi 支持 30+ Provider，PAI 用户的编码不再限于 Claude

这个项目让 pi 超越了"编程助手"的定位，成为 **PAI Life OS 的通用 AI 执行引擎**。用户只需维护一份 TELOS 和记忆，就可以在任意模型（OpenAI、Anthropic、Google、DeepSeek...）间切换，始终获得个性化的 AI 体验。
