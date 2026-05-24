# 第 5 章 @earendil-works/pi-coding-agent：交互式 AI 编程助手

## 5.1 整体架构

`@earendil-works/pi-coding-agent` 是 pi 项目的**旗舰包**，也是最终用户直接使用的 CLI 工具。它整合了其他 3 个包的所有能力，并添加了扩展系统、会话管理、设置管理等应用层功能。

### 5.1.1 依赖结构

```bash
@earendil-works/pi-coding-agent
├── @earendil-works/pi-tui        # 终端 UI
├── @earendil-works/pi-ai         # LLM 统一 API
├── @earendil-works/pi-agent-core # Agent 运行时
├── chalk            # 终端颜色
├── cross-spawn      # 跨平台子进程
├── diff             # 文件差异对比
├── highlight.js     # 代码语法高亮
├── minimatch        # 文件 glob 匹配
├── proper-lockfile  # 文件锁
├── typebox           # JSON Schema
├── undici           # HTTP 客户端
├── yaml             # YAML 解析
└── ...
```

### 5.1.2 三种运行模式

pi 支持三种运行模式：

```
pi                    → Interactive Mode (TUI 交互)
echo "hello" | pi     → Print Mode (批处理/管道)
pi --mode rpc         → RPC Mode (JSON-RPC)
```

**Interactive Mode**：全 TUI 交互，有编辑器、流式渲染、快捷键等。适合日常开发使用。

**Print Mode**：非交互模式。输入来自 CLI 参数或 stdin，直接输出 LLM 回复文本。适合脚本和管道操作：
```bash
echo "解释这段代码" | pi --model openai/gpt-4o
pi "列出当前目录的所有文件"
pi --model anthropic/claude-opus-4-5 "用 Python 写一个 web 服务器"
```

**RPC Mode**：JSON-RPC 协议，支持外部工具（如 VSCode 扩展）与 pi 通信：
```bash
pi --mode rpc
# stdin/stdout 传输 JSON-RPC 消息
# 支持：prompt, continue, listModels, listSessions 等命令
```

### 5.1.4 源代码结构

coding-agent 是 pi 四个包中最大的，有 **300+ 个源文件**，分布在以下目录：

```
src/
├── cli/                          # CLI 命令行解析
│   ├── args.ts                   # 参数解析
│   ├── file-processor.ts         # @file 参数处理
│   ├── initial-message.ts        # 初始消息构建
│   ├── list-models.ts            # --list-models
│   └── session-picker.ts         # 会话选择器
├── core/                         # 核心业务逻辑
│   ├── agent-session.ts          # Agent 会话管理 (核心类)
│   ├── agent-session-runtime.ts  # 运行时创建
│   ├── agent-session-services.ts # 服务创建
│   ├── auth-storage.ts           # 认证存储
│   ├── auth-guidance.ts          # 认证引导
│   ├── bash-executor.ts          # Bash 执行器
│   ├── compaction/               # 上下文压缩
│   ├── config.ts                 # 配置路径
│   ├── event-bus.ts              # 事件总线
│   ├── extensions/               # 扩展系统
│   ├── footer-data-provider.ts   # Footer 数据
│   ├── keybindings.ts            # 快捷键管理
│   ├── messages.ts               # 消息工具
│   ├── model-registry.ts         # 模型注册中心
│   ├── model-resolver.ts         # 模型解析
│   ├── package-manager.ts        # 包管理器
│   ├── provider-display-names.ts # Provider 显示名
│   ├── prompt-templates.ts       # Prompt 模板
│   ├── resource-loader.ts        # 资源加载
│   ├── sdk.ts                    # 程序化调用 SDK
│   ├── session-manager.ts        # 会话管理器
│   ├── settings-manager.ts       # 设置管理器
│   ├── skills.ts                 # 技能管理
│   ├── slash-commands.ts         # 斜杠命令
│   ├── system-prompt.ts          # 系统提示词构建
│   ├── tools/                    # 工具定义
│   │   ├── index.ts              # 工具工厂
│   │   ├── bash.ts               # Bash 工具
│   │   ├── edit.ts               # 编辑工具
│   │   ├── find.ts               # 搜索工具
│   │   ├── grep.ts               # Grep 工具
│   │   ├── ls.ts                 # 列表工具
│   │   ├── read.ts               # 读取工具
│   │   ├── write.ts              # 写入工具
│   │   └── truncate.ts           # 截断工具
│   └── export-html/              # HTML 导出
├── modes/                        # 运行模式
│   ├── index.ts                  # 统一导出
│   ├── interactive/              # 交互模式
│   │   ├── interactive-mode.ts   # 交互模式实现 (5000+ 行)
│   │   ├── theme/                # 主题系统
│   │   └── components/           # UI 组件
│   └── rpc/                      # RPC 模式
├── migrations.ts                 # 数据迁移
├── main.ts                       # CLI 入口
├── config.ts                     # 全局配置
├── package-manager-cli.ts        # 包管理命令
└── utils/                        # 工具函数
    ├── clipboard.ts
    ├── frontmatter.ts
    ├── image-resize.ts
    ├── paths.ts
    ├── shell.ts
    └── ...
```

## 5.2 启动流程

pi 的启动流程由 `main.ts` 中的 `main()` 函数驱动。下面是完整的启动流程：

```
main(args)
│
├── 1. 处理 --offline 标志
├── 2. 处理包管理命令 (pi package install ...)
├── 3. 处理配置命令 (pi config set ...)
│
├── 4. parseArgs(args)
│   ├── 解析 CLI 参数
│   ├── 检测冲突标志 (如 --fork + --session)
│   └── 返回 Args 对象
│
├── 5. resolveAppMode()
│   ├── 决定运行模式 (interactive / print / json / rpc)
│   └── 如果 stdin 不是 TTY 且有内容 → print 模式
│
├── 6. runMigrations() → 数据迁移
│
├── 7. createSessionManager()
│   ├── --no-session → InMemory
│   ├── --fork <session> → forkFrom(source)
│   ├── --session <id> → open(path)
│   ├── --resume → TUI 选择会话
│   ├── --continue → continueRecent()
│   └── 默认 → create(cwd)
│
├── 8. createAgentSessionRuntime()
│   ├── createAgentSessionServices()
│   │   ├── AuthStorage.create()
│   │   ├── ModelRegistry.create()
│   │   ├── SettingsManager.create()
│   │   ├── ResourceLoader → 加载扩展/技能/主题/模板
│   │   └── 启动 Extension Runner
│   │
│   ├── createAgentSessionFromServices()
│   │   ├── 解析模型 (--model / 保存的默认模型 / 作用域模型)
│   │   ├── 创建 Agent (streamFn, hooks, queue modes)
│   │   └── 构建 AgentSession
│   │
│   └── 返回 runtime = { session, services, diagnostics }
│
├── 9. 根据 AppMode 进入不同路径:
│   ├── rpc → runRpcMode(runtime)
│   ├── interactive → new InteractiveMode(runtime).run()
│   └── print → runPrintMode(runtime)
```

### 5.2.1 AgentSession 类解析

`AgentSession` 是 coding-agent 中最重要的类之一（约 3000 行）。它封装了 Agent 的生命周期，是所有运行模式（交互、打印、RPC）共享的核心抽象。

核心能力：

```typescript
class AgentSession {
    // Agent 状态访问
    get state(): AgentState;
    get model(): Model<any> | undefined;
    get thinkingLevel(): ThinkingLevel;

    // 消息管理
    async enqueueUserMessage(text: string, images?: ImageContent[]): Promise<void>;
    async runPrompt(): Promise<void>;
    async continue(): Promise<void>;

    // 事件订阅（自动持久化）
    subscribe(event: AgentSessionEvent, listener: Function): void;

    // 会话控制
    async switchToBranch(entries: SessionEntry[]): Promise<void>;
    async compact(): Promise<void>;

    // 模型控制
    async cycleModel(direction: "forward" | "backward"): Promise<void>;
    setThinkingLevel(level: ThinkingLevel): void;

    // Bash 执行
    async executeBash(command: string): Promise<BashResult>;
}
```

## 5.3 交互式模式的渲染通道

pi 最令人印象深刻的功能之一是**流式消息的实时渲染**。整个渲染通道是 pi-tui、pi-ai、pi-agent-core 三层协作的结果：

```
LLM API
  │
  ▼ SSE 响应流
pi-ai Provider
  │
  ▼ AssistantMessageEvent 流 (text_delta, thinking_delta, toolcall_delta, ...)
Agent Loop
  │
  ▼ AgentEvent (message_update)
InteractiveMode.handleSessionEvent()
  │
  ├── message_start → UserMessageComponent / AssistantMessageComponent 创建
  ├── message_update → AssistantMessageComponent.updateContent()
  │     │
  │     ▼ Markdown 组件重新渲染
  │     TUI.requestRender() → 差分渲染
  │
  ├── message_end → 最终消息保存
  │
  ├── tool_execution_start → ToolExecutionComponent 创建
  ├── tool_execution_update → ToolExecutionComponent.update()
  ├── tool_execution_end → 工具执行完成
  │
  └── turn_end → 检查是否需要继续
```

### 5.3.1 流式消息组件的渲染

`AssistantMessageComponent` 负责渲染 LLM 的流式回复。它根据事件更新内部的 Markdown 组件：

```typescript
class AssistantMessageComponent implements Component {
    private markdown: Markdown;
    private textBuffer = "";

    updateContent(event: AssistantMessageEvent) {
        switch (event.type) {
            case "text_delta":
                this.textBuffer += event.delta;
                this.markdown.setText(this.textBuffer);
                break;
            case "thinking_delta":
                // 在 Markdown 中插入思考块
                break;
            case "toolcall_delta":
                // 显示工具调用进度
                break;
        }
    }

    render(width: number): string[] {
        return this.markdown.render(width);
    }
}
```

每次 `text_delta` 事件到来时，Markdown 组件会重新渲染整个消息内容（包含语法高亮），然后通过 TUI 的差分渲染引擎计算差异，仅输出变化的部分到终端。

## 5.4 会话管理

### 5.4.1 JSONL 存储格式

pi 将会话持久化为 JSONL（JSON Lines）文件，每个会话一个文件。文件格式遵循严格的结构：

```
{"type": "session", "version": 3, "id": "...", "timestamp": "...", "cwd": "..."}  ← 会话头
{"type": "thinking_level_change", "id": "...", "parentId": null, "timestamp": "...", "thinkingLevel": "high"}
{"type": "model_change", "id": "...", "parentId": null, "timestamp": "...", "provider": "openai", "modelId": "gpt-4o"}
{"type": "message", "id": "...", "parentId": "...", "timestamp": "...", "message": {...}}  ← 对话条目
{"type": "compaction", "id": "...", "parentId": "...", "timestamp": "...", "summary": "...", "tokensBefore": 12345}
{"type": "message", "id": "...", "parentId": "...", "timestamp": "...", "message": {...}}
...
```

每个条目有一个 `id` 和 `parentId`，形成一棵**对话树**，支持分支操作。

### 5.4.2 SessionManager

`SessionManager` 负责会话的 CRUD 操作：

```typescript
class SessionManager {
    // 创建新会话
    static create(cwd: string, sessionDir?: string): SessionManager;

    // 打开已有会话
    static open(path: string, sessionDir?: string, cwd?: string): SessionManager;

    // 继续最近的会话
    static continueRecent(cwd: string, sessionDir?: string): SessionManager;

    // 分支：从一个已有会话创建新会话
    static forkFrom(sourcePath: string, cwd: string, sessionDir?: string): SessionManager;

    // 列出会话
    static list(cwd: string, sessionDir?: string): Promise<SessionInfo[]>;
    static listAll(): Promise<SessionInfo[]>;

    // 读取会话树
    getBranch(): SessionTreeEntry[];
    getBranchPaths(): SessionTreeNode[];

    // 追加条目
    appendEntry(entry: SessionEntry): void;

    // 切换分支
    switchToBranch(entries: SessionEntry[]): void;
}
```

### 5.4.3 分支（Fork）机制

分支是 pi 的一个独特功能。你可以从一个会话的某个节点分叉出去，创建新的对话线，而不会破坏原始对话：

```
原始会话:
  user: "写一个 Python web 服务器"
    assistant: "..."
      user: "用 Flask"
        assistant: "..."  ← 从这里分叉

分支会话:
  user: "写一个 Python web 服务器"
    assistant: "..."
      user: "用 FastAPI 替代"  ← 新的分支
        assistant: "..."
```

分叉会话是一个独立的 JSONL 文件，但保留了原始会话的上下文。

### 5.4.4 上下文压缩（Compaction）

当对话历史过长时，pi 会对其进行压缩——用摘要替代早期对话，以控制上下文窗口：

```typescript
async function compact(session: AgentSession) {
    // 1. 计算当前上下文 Token 数
    const tokens = estimateContextTokens(session);

    // 2. 判断是否需要压缩
    if (tokens < settings.compactionThreshold) return;

    // 3. 准备压缩（让 LLM 生成摘要）
    const { summary, cutPoint } = await prepareCompaction(session, model);

    // 4. 应用压缩
    applyCompaction(session, summary, cutPoint);

    // 5. 记录压缩条目到 JSONL
    session.addCompactionEntry(summary, tokens);
}
```

压缩后，对话历史变为：
```
compaction: "用户要求写一个 Python web 服务器，选择了 FastAPI..."
user: "再加一个数据库"
assistant: "..."
```

## 5.5 内置工具

pi 提供了 6 个内置工具，让 AI 代理可以直接操作文件系统：

| 工具名 | 用途 | 输入 | 特点 |
|--------|------|------|------|
| `read` | 读取文件内容 | 路径、偏移、行数限制 | 支持大文件分页读取 |
| `write` | 写入文件 | 路径、内容 | 自动创建父目录 |
| `edit` | 编辑文件 | 路径、新旧文本替换 | 精确匹配替换 |
| `bash` | 执行命令 | shell 命令 | 交互式进程、超时控制 |
| `grep` | 搜索文件内容 | 模式、路径、包含/排除 | 基于 ripgrep |
| `find` | 查找文件 | 路径、类型、名称模式 | 支持 .gitignore |
| `ls` | 列出目录 | 路径 | 同 ls -la |

这些工具的实现位于 `src/core/tools/` 下，每个工具都遵循 `AgentTool` 接口：

```typescript
// src/core/tools/index.ts — 工具工厂
export function createCodingTools(options: ToolsOptions): AgentTool<any>[] {
    return [
        createReadTool(options),
        createBashTool(options),
        createEditTool(options),
        createWriteTool(options),
        createGrepTool(options),
        createFindTool(options),
        createLsTool(options),
    ];
}
```

### Bash 工具的特殊性

Bash 工具是 pi 最强大的工具之一，它启动一个**持久的 shell 进程**：

```typescript
// src/core/tools/bash.ts
class BashTool {
    private shellProcess: ChildProcess;

    async execute(command: string): Promise<AgentToolResult> {
        // 1. 安全检查（防止危险命令）
        // 2. 写命令到 shell 的 stdin
        // 3. 异步读取 stdout/stderr
        // 4. 超时控制
        // 5. 如果输出过长，截断
        // 6. 返回结果
    }
}
```

## 5.6 SDK：程序化调用

pi 的一个重要设计是提供了完整的 SDK，允许其他程序以库的形式使用 pi 的能力，而不仅仅是作为一个 CLI 工具：

```typescript
import { createAgentSession } from "@earendil-works/pi-coding-agent";
import { getModel } from "@earendil-works/pi-ai";

// 最简用法
const { session } = await createAgentSession();

// 指定模型
const { session } = await createAgentSession({
    model: getModel("anthropic", "claude-opus-4-5"),
});

// 全控制
const { session, services } = await createAgentSession({
    model: myModel,
    tools: ["read", "bash"],
    thinkingLevel: "high",
    resourceLoader: myLoader,
    sessionManager: SessionManager.inMemory(),
});
```

SDK 的入口在 `src/core/sdk.ts` 中，`createAgentSession()` 函数整合了所有服务创建和配置逻辑。

## 5.7 章节小结

pi-coding-agent 是一个功能完备的 AI 编程助手 CLI，它的架构亮点包括：

1. **三层运行模式**：交互式（TUI）、打印（批处理）、RPC（外部集成）
2. **完整的启动流程**：从参数解析到运行时构建再到模式分发
3. **AgentSession 作为核心抽象**：封装了 Agent 运行时、事件系统、持久化和压缩
4. **流式渲染通道**：从 LLM SSE → EventStream → TUI 差分渲染的完整管线
5. **JSONL 会话存储**：支持分支、压缩、恢复
6. **6 个内置工具**：Read/Write/Edit/Bash/Grep/Find/Ls
7. **程序化 SDK**：允许将 pi 作为库使用
