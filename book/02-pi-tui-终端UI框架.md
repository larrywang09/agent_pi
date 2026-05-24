# 第 2 章 @earendil-works/pi-tui：自己动手写终端 UI 框架

## 2.1 终端渲染基础

在深入 pi-tui 的源码之前，我们需要了解终端渲染的一些基础知识。这些知识对于理解 pi-tui 的设计至关重要。

### 2.1.1 ANSI 转义序列

终端通过 **ANSI 转义序列**（ANSI Escape Sequences）来控制文本样式、光标位置和屏幕内容。这些序列以 `\x1b[`（ESC + `[`，即 CSI — Control Sequence Introducer）开头：

```typescript
// 设置文本颜色（前景色）
"\x1b[31m"  // 红色
"\x1b[32m"  // 绿色
"\x1b[33m"  // 黄色

// 设置背景色
"\x1b[41m"  // 红色背景

// 样式
"\x1b[1m"   // 粗体
"\x1b[4m"   // 下划线
"\x1b[7m"   // 反转（前景色和背景色互换）

// 重置所有样式
"\x1b[0m"

// 光标操作
"\x1b[2J"   // 清屏
"\x1b[H"    // 光标归位 (0,0)
"\x1b[6;1H" // 将光标移动到第 6 行第 1 列
"\x1b[A"    // 光标上移一行
"\x1b[2K"   // 清除当前行
```

pi-tui 大量使用这些序列来实现差分渲染和光标定位。

### 2.1.2 同步输出（Synchronized Output）

终端输出存在一个经典问题：当程序逐行写入屏幕时，用户可能看到中间状态（行被逐个刷出，产生闪烁）。为了解决这个问题，现代终端（如 Kitty、iTerm2、Ghostty）支持 **CSI 2026 同步输出协议**：

```typescript
"\x1b[?2026h"  // 开始同步输出 — 终端缓存所有写入，直到结束标记
// ... 所有写入内容 ...
"\x1b[?2026l"  // 结束同步输出 — 终端一次性刷新所有内容
```

pi-tui 在所有渲染操作中都用这个协议包裹，实现无闪烁渲染。

### 2.1.3 Kitty 键盘协议

传统终端键盘处理有一个严重局限：无法区分 `Ctrl+C` 和单独的 `C` 键按下，也无法区分按键按下和释放。**Kitty 键盘协议**（Kitty Keyboard Protocol）解决了这个问题：

- 每个按键事件包含完整的修饰符信息（Ctrl/Alt/Shift/Super）
- 支持按键按下和释放事件的区分
- 支持功能键（F1-F12）和更多特殊键

pi-tui 的 `keys.ts` 模块实现了 Kitty 协议的探测和解析，同时也向下兼容传统序列。

### 2.1.4 终端图片渲染

pi-tui 支持在终端中显示图片，通过两种协议：

- **Kitty 图片协议**：将图片编码为 base64，通过 APC（Application Program Command）序列发送给终端，由终端解码并渲染
- **iTerm2 内联图片协议**：使用 OSC 1337 序列

```typescript
// Kitty 图片协议示例（简化）
"\x1b_Ga=T,f=100,i=1;base64编码数据...\x1b\\"
```

pi-tui 的 `terminal-image.ts` 封装了所有这些细节。

## 2.2 整体架构

pi-tui 的源码结构非常清晰，位于 `packages/tui/src/` 目录下：

```
src/
├── index.ts              # 统一导出
├── tui.ts                # 核心：TUI 类（差分渲染、组件管理、覆盖层）
├── terminal.ts           # 终端抽象层（ProcessTerminal + VirtualTerminal）
├── terminal-image.ts     # 终端图片渲染
├── keys.ts               # 键盘事件解析（Kitty 协议）
├── keybindings.ts        # 快捷键绑定系统
├── kill-ring.ts          # 编辑器的 kill ring（类似 Emacs）
├── editor-component.ts   # 编辑器组件接口
├── stdin-buffer.ts       # 输入缓冲（批量分割）
├── autocomplete.ts       # 自动补全（文件路径 + 命令）
├── fuzzy.ts              # 模糊搜索
├── undo-stack.ts         # 撤销栈
├── utils.ts              # 宽度计算、文本裁剪、ANSI 处理
└── components/           # 内置组件
    ├── box.ts
    ├── cancellable-loader.ts
    ├── editor.ts
    ├── image.ts
    ├── input.ts
    ├── loader.ts
    ├── markdown.ts
    ├── select-list.ts
    ├── settings-list.ts
    ├── spacer.ts
    ├── text.ts
    └── truncated-text.ts
```

pi-tui 只有 **2 个运行时依赖**：

| 依赖 | 用途 |
|------|------|
| `get-east-asian-width` | 计算东亚字符（CJK）的显示宽度 |
| `marked` | Markdown 解析（用于 Markdown 组件） |

以及一个**可选的** `koffi`（FFI 库，用于某些平台特定的终端操作）。

## 2.3 三种渲染策略详解

pi-tui 的差分渲染引擎是它最核心的设计。它的渲染策略分为三种情况，由 `doRender()` 方法实现：

### 策略一：首次渲染（First Render）

**触发条件**：`previousLines` 数组为空。

**行为**：将所有新行直接输出到屏幕，不执行任何清除操作。假设终端处于干净的初始状态。

```typescript
// tui.ts — 简化后的首次渲染逻辑
if (this.previousLines.length === 0 && !widthChanged && !heightChanged) {
    fullRender(false);  // false = 不清屏
    return;
}
```

`fullRender(false)` 会输出所有行，但不清除屏幕，从而避免闪屏。

### 策略二：宽度变化或视口上方内容变化 — 全量重绘

**触发条件**：

1. **终端宽度变化**（`widthChanged`）：文本换行位置会全部改变，必须重新渲染。
2. **终端高度变化**（`heightChanged`）：视口大小改变，需要重新计算可见内容。
3. **内容在视口之上发生变化**：无法通过增量更新来修复。
4. **内容缩小且启用了 `clearOnShrink`**：清除多余的空白行。

**行为**：清除屏幕和滚动缓冲区，然后从头输出所有行。

```typescript
// 策略二：全量重绘
if (widthChanged) {
    logRedraw(`terminal width changed (${this.previousWidth} -> ${width})`);
    fullRender(true);  // true = 清屏
    return;
}
```

### 策略三：增量更新（Differential Update）

**触发条件**：内容发生变化，但变化区域在当前视口内。

**行为**：计算发生变化的第一行和最后一行，然后：

1. 移动光标到变化的第一行
2. 清除该行及其后的变化行
3. 输出变化行的新内容
4. 如果内容增加了，继续输出新增行

```typescript
// 策略三的简化逻辑
let buffer = "\x1b[?2026h";  // 开始同步输出
buffer += this.deleteChangedKittyImages(firstChanged, lastChanged);
buffer += moveCursorTo(firstChanged);

for (let i = firstChanged; i <= renderEnd; i++) {
    if (i > firstChanged) buffer += "\r\n";
    buffer += "\x1b[2K";         // 清除当前行
    buffer += newLines[i];       // 输出新内容
}
buffer += "\x1b[?2026l";        // 结束同步输出
this.terminal.write(buffer);
```

这种策略最常被触发——比如流式输出中每次 `text_delta` 事件更新 Markdown 内容时。

### 三种策略的决策流程

```
doRender()
│
├─ previousLines 为空 → 首次渲染（不清屏全输出）
│
├─ 宽度变化 → 清屏全量重绘
│
├─ 高度变化（非 Termux）→ 清屏全量重绘
│
├─ 内容缩小且 clearOnShrink=true → 清屏全量重绘
│
├─ firstChanged === -1 → 无变化，只更新光标位置
│
├─ firstChanged >= newLines.length → 内容被删除
│
├─ firstChanged < viewportTop → 变化在视口之上，全量重绘
│
└─ 常规情况 → 增量更新（从 firstChanged 到 newLines.length）
```

## 2.4 组件体系

### 2.4.1 Component 接口

pi-tui 的所有组件都基于一个极其简洁的接口：

```typescript
export interface Component {
    // 渲染组件内容，width 是终端宽度
    render(width: number): string[];

    // 可选的键盘事件处理
    handleInput?(data: string): void;
    // 如果为 true，组件会收到按键释放事件（Kitty 协议）
    wantsKeyRelease?: boolean;

    // 清除缓存的渲染状态
    invalidate(): void;
}
```

**关键合约**：
- `render(width)` 返回的**每一行字符串长度不能超过 `width`**，否则 TUI 会报错并崩溃
- TUI 会在每行末尾追加 SGR 重置（`\x1b[0m`）和超链接重置（`\x1b]8;;\x07`），所以样式不会跨行延续
- `invalidate()` 方法用于清除组件的缓存状态，在主题变更等场景下调用

### 2.4.2 Focusable 接口 — IME 支持

为了支持中日韩等输入法的候选窗口正确定位，pi-tui 定义了 `Focusable` 接口：

```typescript
export interface Focusable {
    focused: boolean;  // TUI 在焦点变化时设置
}

// 类型守卫
export function isFocusable(component: Component | null): component is Component & Focusable {
    return component !== null && "focused" in component;
}
```

当某个 `Focusable` 组件获得焦点时，它需要在渲染输出中插入 `CURSOR_MARKER`（一个零宽度的 APC 转义序列 `\x1b_pi:c\x07`），TUI 会在渲染完成后扫描该标记，将硬件光标移到对应位置。

```typescript
// Focusable 组件在 render() 中使用 CURSOR_MARKER
render(width: number): string[] {
    const marker = this.focused ? CURSOR_MARKER : "";
    // 在光标位置前插入标记
    return [`> ${textBeforeCursor}${marker}${textAtCursor}`];
}
```

### 2.4.3 内置组件一览

| 组件 | 文件 | 功能描述 |
|------|------|---------|
| `Text` | `components/text.ts` | 多行文本，支持自动换行和背景色 |
| `TruncatedText` | `components/truncated-text.ts` | 单行文本，超出宽度自动截断 |
| `Input` | `components/input.ts` | 单行输入框，支持水平滚动 |
| `Editor` | `components/editor.ts` | 多行编辑器，支持自动补全、粘贴处理、滚动 |
| `Markdown` | `components/markdown.ts` | Markdown 渲染，支持语法高亮 |
| `Loader` | `components/loader.ts` | 动画加载指示器 |
| `CancellableLoader` | `components/cancellable-loader.ts` | 可取消的加载指示器（按 Escape 取消） |
| `SelectList` | `components/select-list.ts` | 交互式选择列表，键盘导航 |
| `SettingsList` | `components/settings-list.ts` | 设置面板，支持值切换和子菜单 |
| `Box` | `components/box.ts` | 容器，添加内边距和背景色 |
| `Spacer` | `components/spacer.ts` | 垂直间距 |
| `Image` | `components/image.ts` | 内联图片（Kitty/iTerm2 协议） |
| `Container` | `tui.ts`（内联） | 子组件容器 |

### 2.4.4 Container 和 TUI 的继承关系

`TUI` 继承自 `Container`，而 `Container` 实现了 `Component` 接口：

```typescript
// Container：简单地按顺序渲染所有子组件
export class Container implements Component {
    children: Component[] = [];

    render(width: number): string[] {
        const lines: string[] = [];
        for (const child of this.children) {
            const childLines = child.render(width);
            for (const line of childLines) {
                lines.push(line);
            }
        }
        return lines;
    }
}

// TUI 继承 Container，添加了差分渲染、键盘事件分发、覆盖层等功能
export class TUI extends Container {
    // ...完整实现超过 1300 行
}
```

这意味着 **TUI 本身也是一个组件**，可以嵌套在其他 TUI 中（实际上很少这么用）。

## 2.5 键盘事件系统

### 2.5.1 Key 类型系统

pi-tui 的键盘处理位于 `keys.ts`，它定义了一个完备的类型化按键标识符系统：

```typescript
// 按键标识符类型 (TypeScript 类型级别的自动补全)
type KeyId = "enter" | "escape" | "ctrl+c" | "alt+shift+a" | "super+ctrl+alt+shift+f1" | ...;

// 使用 Key 辅助对象创建类型安全的按键标识符
Key.ctrl("c")        // → "ctrl+c"
Key.alt("a")         // → "alt+a"
Key.shift("tab")     // → "shift+tab"
Key.ctrlShift("p")   // → "ctrl+shift+p"
Key.super("space")   // → "super+space"
```

### 2.5.2 matchesKey — 按键匹配

`matchesKey()` 是按键分发系统的核心函数：

```typescript
function matchesKey(data: string, keyId: KeyId | KeyId[]): boolean
```

它在 TUI 的 `handleInput()` 中被调用：

```typescript
private handleInput(data: string): void {
    // 1. 先遍历全局输入监听器（允许消费或修改输入）
    for (const listener of this.inputListeners) {
        const result = listener(current);
        if (result?.consume) return;
        // ...
    }

    // 2. 调试快捷键：Shift+Ctrl+D
    if (matchesKey(data, "shift+ctrl+d") && this.onDebug) { ... }

    // 3. 过滤按键释放事件
    if (isKeyRelease(data) && !this.focusedComponent.wantsKeyRelease) return;

    // 4. 转发给当前焦点组件
    this.focusedComponent?.handleInput?.(data);
}
```

### 2.5.3 快捷键绑定系统

`keybindings.ts` 提供了一个可配置的快捷键系统，支持用户覆盖默认绑定：

```typescript
// 定义快捷键
export const TUI_KEYBINDINGS = {
    "tui.editor.cursorUp":    { defaultKeys: "up",        description: "Move cursor up" },
    "tui.editor.cursorDown":  { defaultKeys: "down",      description: "Move cursor down" },
    "tui.editor.cursorLeft":  { defaultKeys: ["left", "ctrl+b"], description: "Move cursor left" },
    "tui.editor.submit":      { defaultKeys: "enter",     description: "Submit" },
    "tui.editor.newLine":     { defaultKeys: "alt+enter", description: "Insert new line" },
    // ...
};

// KeybindingsManager 负责解析用户配置的覆盖
export class KeybindingsManager {
    // 用户配置覆盖默认值
    resolve(keybinding: Keybinding): KeyId[] {
        const configured = this.config[keybinding];
        const defaults = TUI_KEYBINDINGS[keybinding]?.defaultKeys;
        const keys = configured !== undefined ? toArray(configured) : toArray(defaults);
        return keys;
    }
}
```

## 2.6 覆盖层（Overlay）系统

覆盖层是 pi-tui 中用于弹窗、对话框、菜单的关键机制。它允许组件渲染在现有内容之上，而不替换现有内容。

### 2.6.1 覆盖层的创建与定位

```typescript
// 显示一个覆盖层
const handle = tui.showOverlay(myComponent, {
    width: "50%",
    height: "30%",
    anchor: "center",
    minWidth: 40,
    margin: 2,
});

// 控制覆盖层
handle.hide();          // 永久移除
handle.setHidden(true); // 临时隐藏
handle.focus();         // 设置焦点到覆盖层
handle.unfocus();       // 释放焦点
```

### 2.6.2 定位策略

覆盖层的定位支持多种方式：

1. **锚点（Anchor）方式**：通过 9 个锚点定位
   ```typescript
   type OverlayAnchor = "center" | "top-left" | "top-right" | "bottom-left" | "bottom-right"
                       | "top-center" | "bottom-center" | "left-center" | "right-center";
   ```

2. **百分比方式**：基于终端尺寸的相对位置
   ```typescript
   row: "25%",  // 从顶部 25% 位置开始
   col: "50%",  // 水平居中
   ```

3. **绝对定位**：固定行列
   ```typescript
   row: 5,  col: 10,
   ```

### 2.6.3 覆盖层合成

覆盖层的渲染通过与主内容的合成来完成：

```typescript
private compositeOverlays(lines: string[], termWidth: number, termHeight: number): string[] {
    // 1. 排序覆盖层（按 focusOrder）
    // 2. 逐个渲染覆盖层内容
    // 3. 通过 compositeLineAt() 合成到主内容上
    for (const { overlayLines, row, col, w } of rendered) {
        for (let i = 0; i < overlayLines.length; i++) {
            const idx = viewportStart + row + i;
            result[idx] = this.compositeLineAt(result[idx], overlayLine, col, w, termWidth);
        }
    }
}
```

`compositeLineAt()` 负责将一个覆盖层行合成到基础行上，处理 ANSI 转义序列的宽度计算：

```typescript
private compositeLineAt(baseLine, overlayLine, startCol, overlayWidth, totalWidth): string {
    // 1. 提取基础行中覆盖层之前/之后的部分
    // 2. 提取覆盖层文本（截断到指定宽度）
    // 3. 拼接：之前部分 + 覆盖层 + 之后部分
    // 4. 最终验证宽度不超过 totalWidth，否则截断
}
```

## 2.7 Editor 组件深度解析

Editor 是 pi-tui 中最复杂的组件，它是 pi 交互式模式中最核心的用户输入界面。

### 2.7.1 功能特性

- **多行编辑**：支持换行和滚动
- **自动补全**：文件路径补全 + 斜杠命令补全
- **大粘贴处理**：超过 10 行的粘贴会被检测并折叠为 `[paste #1 +50 lines]`
- **Kill Ring**：类 Emacs 的删除/粘贴环
- **撤销栈**：支持撤销操作
- **安全提交**：Enter 键正常提交，`Shift+Enter` 或 `Alt+Enter` 换行

### 2.7.2 快捷键

Editor 支持大量快捷键（全部可配置）：

```
移动:        Ctrl+A/E (行首/行尾), Ctrl+Left/Right (单词)
             Ctrl+] (跳到字符), Ctrl+Alt+] (向后跳)
删除:        Ctrl+K (删到行尾), Ctrl+U (删到行首)
             Ctrl+W/Alt+Backspace (删单词向后), Alt+D/Alt+Delete (删单词向前)
其他:        Ctrl+Y (粘贴), Alt+Y (轮换粘贴环), Ctrl+Z (撤销)
提交:        Enter, Shift+Enter/Alt+Enter (换行)
自动补全:    Tab
```

## 2.8 文本宽度处理

终端文本宽度处理是一个被严重低估的复杂问题。pi-tui 的 `utils.ts` 专门处理这个问题。

### 2.8.1 visibleWidth — 计算可见宽度

ANSI 转义序列不计入可见宽度，但东亚字符（CJK）占用 2 列：

```typescript
// 计算字符串的可见宽度（忽略 ANSI 序列，但计入全角字符）
visibleWidth("\x1b[31mHello\x1b[0m"); // 5
visibleWidth("中文");                      // 4（每个字符 2 列）
visibleWidth("a🌍b");                      // 4（emoji 算 2 列？看具体实现）
```

实现使用了 `Intl.Segmenter` 进行字素分割，配合正则检测零宽字符和 RGI Emoji：

```typescript
export function visibleWidth(str: string): number {
    if (isPrintableAscii(str)) return str.length;  // 快速路径

    let width = 0;
    // 分割为字素簇（grapheme clusters）
    for (const segment of getSegmenter().segment(str)) {
        const s = segment.segment;
        if (s === "\t") {
            width += 8 - (width % 8);  // Tab 对齐
        } else if (zeroWidthRegex.test(s)) {
            continue;  // 零宽字符
        } else if (couldBeEmoji(s) && rgiEmojiRegex.test(s)) {
            width += 2;  // Emoji
        } else {
            const cp = s.codePointAt(0)!;
            width += eastAsianWidth(cp);  // 查表判断 1 或 2
        }
    }
    return width;
}
```

### 2.8.2 truncateToWidth — 按宽度截断

```typescript
// 截断到指定可见宽度（保留 ANSI 序列）
truncateToWidth("Hello World", 8);    // "Hello..."（默认追加 ...）
truncateToWidth("Hello World", 8, ""); // "Hello Wo"（不追加）

// 正确处理 ANSI 序列
truncateToWidth(chalk.red("Hello World"), 5);  // 红色 "Hello" + 重置
```

## 2.9 章节小结

pi-tui 是一个设计精巧的终端 UI 框架，它的核心价值在于：

1. **差分渲染**：三种渲染策略覆盖了所有场景，大幅减少了不必要的屏幕刷新
2. **简洁的组件接口**：只有 `render()`、`handleInput()`、`invalidate()` 三个方法
3. **完备的键盘处理**：支持 Kitty 协议 + 传统序列，类型安全的快捷键系统
4. **可配置的快捷键**：所有快捷键都可以被用户覆盖
5. **覆盖层系统**：支持复杂的弹窗/对话框布局
6. **国际化支持**：正确处理 CJK 字符、Emoji、零宽字符的宽度计算

pi-tui 的代码非常值得阅读——它只有 14 个源文件，但几乎涵盖了终端 UI 开发的所有核心问题。阅读它的源码是学习终端编程的最佳实践之一。
