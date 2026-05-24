# 实战项目三：TUI 文件管理器 Widget

## 项目概述

本次项目将构建一个**文件管理器 Widget**，在 pi 的交互模式下提供图形化的文件浏览能力。这是对 pi-tui 的 `Component` 接口、覆盖层系统、键盘处理能力的深度实践。

通过这个项目，你将掌握：
- 自定义 `Component` 接口实现（render/handleInput/invalidate）
- `Focusable` 接口与 IME 支持
- 覆盖层系统（`OverlayOptions`、`OverlayHandle`）
- pi-tui 的 `SelectList`、`Text`、`TruncatedText` 等组件的组合使用
- 扩展系统的 `setWidget` / `setEditorComponent` / `registerCommand`
- 文件系统操作的集成

## 项目结构

```
pi-file-manager/
├── src/
│   ├── index.ts              # 扩展入口
│   ├── file-tree.ts          # 文件树核心组件
│   ├── file-preview.ts       # 文件预览组件
│   ├── file-ops.ts           # 文件操作（CRUD）
│   └── keybindings.ts        # 快捷键绑定
├── package.json
└── README.md
```

## 核心实现：FileTree 组件

```typescript
// src/file-tree.ts
import * as fs from "node:fs";
import * as path from "node:path";
import {
    Text,
    TruncatedText,
    truncateToWidth,
    visibleWidth,
    matchesKey,
} from "@earendil-works/pi-tui";
import type { Component, Theme } from "@earendil-works/pi-coding-agent";

// ============================================================================
// 文件树节点
// ============================================================================

interface FileNode {
    name: string;
    path: string;
    type: "file" | "dir";
    expanded: boolean;
    children?: FileNode[];
    depth: number;
}

// ============================================================================
// 文件树组件
// ============================================================================

export class FileTreeComponent implements Component {
    private root: FileNode[];
    private scrollOffset = 0;
    private selectedIndex = 0;
    private flatList: FlatEntry[] = [];
    private cwd: string;
    private theme: Theme;
    private showDotfiles = false;
    private filterPattern = "";
    private cachedWidth = 0;
    private cachedLines: string[] = [];

    // 回调
    public onSelectFile?: (filePath: string) => void;
    public onClose?: () => void;
    public onNavigate?: (filePath: string) => void;

    constructor(cwd: string, theme: Theme) {
        this.cwd = cwd;
        this.theme = theme;
        this.root = [];
        this.loadDirectory(cwd);
    }

    get currentPath(): string {
        const entry = this.flatList[this.selectedIndex];
        return entry ? entry.node.path : this.cwd;
    }

    // ====================================================================
    // 目录加载
    // ====================================================================

    private loadDirectory(dirPath: string): void {
        try {
            const entries = fs.readdirSync(dirPath, { withFileTypes: true });
            const nodes: FileNode[] = [];

            for (const entry of entries) {
                if (!this.showDotfiles && entry.name.startsWith(".")) continue;

                if (this.filterPattern && !entry.name.toLowerCase().includes(this.filterPattern.toLowerCase())) {
                    continue;
                }

                nodes.push({
                    name: entry.name,
                    path: path.join(dirPath, entry.name),
                    type: entry.isDirectory() ? "dir" : "file",
                    expanded: false,
                    depth: 0,
                });
            }

            // 排序：目录优先，按名称排序
            nodes.sort((a, b) => {
                if (a.type !== b.type) return a.type === "dir" ? -1 : 1;
                return a.name.localeCompare(b.name);
            });

            this.root = nodes;
            this.flattenTree();
        } catch (error) {
            this.root = [];
            this.flatList = [];
        }
    }

    // ====================================================================
    // 树扁平化（用于渲染）
    // ====================================================================

    private flattenTree(): void {
        this.flatList = [];
        this.flattenNodes(this.root, 0);
    }

    private flattenNodes(nodes: FileNode[], depth: number): void {
        for (const node of nodes) {
            this.flatList.push({ node, depth });
            if (node.type === "dir" && node.expanded && node.children) {
                this.flattenNodes(node.children, depth + 1);
            }
        }
    }

    // ====================================================================
    // 展开/折叠目录
    // ====================================================================

    private toggleExpand(index: number): void {
        const entry = this.flatList[index];
        if (!entry || entry.node.type !== "dir") return;

        entry.node.expanded = !entry.node.expanded;

        if (entry.node.expanded) {
            // 加载子目录
            entry.node.children = this.loadChildren(entry.node.path, entry.node.depth + 1);
        }

        this.flattenTree();
        this.invalidate();
    }

    private loadChildren(dirPath: string, depth: number): FileNode[] {
        try {
            const entries = fs.readdirSync(dirPath, { withFileTypes: true });
            const nodes: FileNode[] = [];

            for (const entry of entries) {
                if (!this.showDotfiles && entry.name.startsWith(".")) continue;
                nodes.push({
                    name: entry.name,
                    path: path.join(dirPath, entry.name),
                    type: entry.isDirectory() ? "dir" : "file",
                    expanded: false,
                    depth,
                });
            }

            nodes.sort((a, b) => {
                if (a.type !== b.type) return a.type === "dir" ? -1 : 1;
                return a.name.localeCompare(b.name);
            });

            return nodes;
        } catch {
            return [];
        }
    }

    // ====================================================================
    // 过滤
    // ====================================================================

    setFilter(pattern: string): void {
        this.filterPattern = pattern;
        this.loadDirectory(this.cwd);
        this.selectedIndex = 0;
        this.scrollOffset = 0;
    }

    setShowDotfiles(show: boolean): void {
        this.showDotfiles = show;
        this.loadDirectory(this.cwd);
        this.selectedIndex = 0;
        this.scrollOffset = 0;
    }

    // ====================================================================
    // Component 接口
    // ====================================================================

    handleInput(data: string): void {
        if (matchesKey(data, "up") || matchesKey(data, "ctrl+k")) {
            this.selectedIndex = Math.max(0, this.selectedIndex - 1);
            this.ensureVisible();
        } else if (matchesKey(data, "down") || matchesKey(data, "ctrl+j")) {
            this.selectedIndex = Math.min(this.flatList.length - 1, this.selectedIndex + 1);
            this.ensureVisible();
        } else if (matchesKey(data, "space") || matchesKey(data, "enter")) {
            this.toggleExpand(this.selectedIndex);
        } else if (matchesKey(data, "ctrl+o")) {
            // 打开文件
            const entry = this.flatList[this.selectedIndex];
            if (entry && entry.node.type === "file") {
                this.onSelectFile?.(entry.node.path);
            }
        } else if (matchesKey(data, "left") || matchesKey(data, "ctrl+[")) {
            // 折叠当前目录或返回上一级
            const entry = this.flatList[this.selectedIndex];
            if (entry?.node.type === "dir" && entry.node.expanded) {
                this.toggleExpand(this.selectedIndex);
            } else {
                // 上一级目录
                const parent = path.dirname(this.cwd);
                if (parent !== this.cwd) {
                    this.cwd = parent;
                    this.loadDirectory(this.cwd);
                    this.selectedIndex = 0;
                    this.scrollOffset = 0;
                }
            }
        } else if (matchesKey(data, "right")) {
            // 展开目录
            const entry = this.flatList[this.selectedIndex];
            if (entry?.node.type === "dir") {
                this.toggleExpand(this.selectedIndex);
            }
        } else if (matchesKey(data, "home")) {
            this.selectedIndex = 0;
            this.scrollOffset = 0;
        } else if (matchesKey(data, "end")) {
            this.selectedIndex = this.flatList.length - 1;
            this.ensureVisible();
        } else if (matchesKey(data, "pageUp")) {
            this.selectedIndex = Math.max(0, this.selectedIndex - 20);
            this.ensureVisible();
        } else if (matchesKey(data, "pageDown")) {
            this.selectedIndex = Math.min(this.flatList.length - 1, this.selectedIndex + 20);
            this.ensureVisible();
        } else if (matchesKey(data, "escape")) {
            this.onClose?.();
        }
    }

    render(width: number): string[] {
        if (this.cachedWidth === width && this.cachedLines.length > 0) {
            return this.cachedLines;
        }

        const th = this.theme;
        const lines: string[] = [];
        const maxVisible = 20;

        // 标题栏
        const header = ` ${th.fg("accent", th.bold("File Browser"))}  ${th.fg("dim", this.cwd)} `;
        lines.push(truncateToWidth(header, width));
        lines.push(th.fg("border", `─${"─".repeat(Math.max(0, width - 2))}─`));

        // 文件列表
        const start = this.scrollOffset;
        const end = Math.min(start + maxVisible, this.flatList.length);

        for (let i = start; i < end; i++) {
            const entry = this.flatList[i];
            const isSelected = i === this.selectedIndex;
            const prefix = isSelected ? th.fg("accent", "▸") : " ";
            const indent = "  ".repeat(entry.node.depth);

            let icon: string;
            let color: (s: string) => string;

            if (entry.node.type === "dir") {
                icon = entry.node.expanded ? "▼" : "▶";
                color = th.fg("accent");
            } else {
                icon = "○";
                color = th.fg("muted");
            }

            const line = `${prefix} ${color(`${indent}${icon} ${entry.node.name}`)}`;
            lines.push(isSelected
                ? truncateToWidth(th.bg("accent", line), width)
                : truncateToWidth(line, width));
        }

        // 底部状态栏
        lines.push(th.fg("border", `─${"─".repeat(Math.max(0, width - 2))}─`));
        const status = ` ${this.flatList.length} items | ↑↓ navigate | Space toggle | Ctrl+O open | Esc close `;
        lines.push(truncateToWidth(th.fg("dim", status), width));

        this.cachedWidth = width;
        this.cachedLines = lines;
        return lines;
    }

    invalidate(): void {
        this.cachedWidth = 0;
        this.cachedLines = [];
    }

    private ensureVisible(): void {
        const maxVisible = 20;
        if (this.selectedIndex < this.scrollOffset) {
            this.scrollOffset = this.selectedIndex;
        } else if (this.selectedIndex >= this.scrollOffset + maxVisible) {
            this.scrollOffset = this.selectedIndex - maxVisible + 1;
        }
    }
}

interface FlatEntry {
    node: FileNode;
    depth: number;
}
```

## 文件预览组件

```typescript
// src/file-preview.ts
import * as fs from "node:fs";
import { Text, truncateToWidth } from "@earendil-works/pi-tui";
import type { Component, Theme } from "@earendil-works/pi-coding-agent";

export class FilePreviewComponent implements Component {
    private filePath: string;
    private content: string;
    private theme: Theme;
    private cachedWidth = 0;
    private cachedLines: string[] = [];

    constructor(filePath: string, theme: Theme) {
        this.filePath = filePath;
        this.theme = theme;
        this.content = this.readFile();
    }

    private readFile(): string {
        try {
            const stat = fs.statSync(this.filePath);
            if (stat.size > 1024 * 100) { // >100KB 截断
                const buffer = Buffer.alloc(1024 * 100);
                const fd = fs.openSync(this.filePath, "r");
                fs.readSync(fd, buffer, 0, buffer.length, 0);
                fs.closeSync(fd);
                return buffer.toString("utf-8") + "\n\n... (file truncated, 100KB max)";
            }
            return fs.readFileSync(this.filePath, "utf-8");
        } catch {
            return `Error reading file: ${this.filePath}`;
        }
    }

    render(width: number): string[] {
        if (this.cachedWidth === width && this.cachedLines.length > 0) {
            return this.cachedLines;
        }

        const th = this.theme;
        const lines: string[] = [];
        const contentLines = this.content.split("\n");
        const maxPreviewLines = 30;

        // 标题
        lines.push(th.fg("accent", th.bold(` Preview: ${this.filePath} `)));
        lines.push(th.fg("border", `─${"─".repeat(Math.max(0, width - 2))}─`));

        // 文件内容（截断）
        const previewLines = contentLines.slice(0, maxPreviewLines);
        for (const line of previewLines) {
            lines.push(truncateToWidth(th.fg("text", line), width));
        }

        if (contentLines.length > maxPreviewLines) {
            lines.push(th.fg("dim", `... ${contentLines.length - maxPreviewLines} more lines`));
        }

        this.cachedWidth = width;
        this.cachedLines = lines;
        return lines;
    }

    invalidate(): void {
        this.cachedWidth = 0;
        this.cachedLines = [];
    }
}
```

## 扩展入口

```typescript
// src/index.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { FileTreeComponent } from "./file-tree.ts";
import { FilePreviewComponent } from "./file-preview.ts";

export default function (pi: ExtensionAPI) {
    // /files 命令 — 打开文件管理器
    pi.registerCommand("files", {
        description: "Browse files with an interactive file tree",
        handler: async (_args, ctx) => {
            if (!ctx.hasUI) {
                ctx.ui.notify("/files requires interactive mode", "error");
                return;
            }

            await ctx.ui.custom<void>((tui, theme, _kb, done) => {
                const tree = new FileTreeComponent(ctx.cwd, theme);

                tree.onClose = () => done();
                tree.onSelectFile = (filePath) => {
                    // 用预览覆盖层显示文件
                    const preview = new FilePreviewComponent(filePath, theme);
                    tui.showOverlay(preview, {
                        width: "70%",
                        maxHeight: "80%",
                        anchor: "center",
                    });
                };

                return tree;
            }, {
                overlay: true,
                overlayOptions: {
                    width: "60%",
                    maxHeight: "90%",
                    anchor: "left-center",
                    margin: 1,
                },
            });
        },
    });

    // 注册快捷键 Alt+F
    pi.registerShortcut(["alt+f"], "app.fileBrowser");
}
```

## 使用效果

```
┌─ File Browser ──────────────────────────────────── /Users/user/project ─┐
 ▸ ▼ src
      ▶ components/
      ○ index.ts
      ○ app.ts
   ▶ tests/
   ○ package.json
   ○ README.md
   ○ tsconfig.json
│  12 items | ↑↓ navigate | Space toggle | Ctrl+O open | Esc close        │
└──────────────────────────────────────────────────────────────────────────┘
```
