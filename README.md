# Claude Code on Windows Terminal — Tab Resilience Toolkit

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform: Windows](https://img.shields.io/badge/Platform-Windows-blue.svg)](#)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-v2.x-orange.svg)](https://docs.anthropic.com/claude/docs/claude-code)
[![Windows Terminal](https://img.shields.io/badge/Windows%20Terminal-v1.21+-blue.svg)](https://github.com/microsoft/terminal)
[![Zero Dependencies](https://img.shields.io/badge/dependencies-zero-brightgreen.svg)](#)

> **零依赖、纯 settings.json 配置** — 让 Windows Terminal × Claude Code 多 tab 工作流不再脆弱
>
> 五分钟搞定：tab 误关一键复活、整个 WT 关了自动恢复布局、后台 tab 完成时自动响铃 + 弹 Toast 主动提醒

在 Windows Terminal 里跑多个并发 Claude Code session 是常见工作流，但有两个反复折磨人的痛点。本仓库给出一套零依赖、纯 settings.json 配置的解法。

---

## 痛点

### 1. WT tab 意外关闭丢失工作布局

- 误点 X 关 tab、误按 `Ctrl+W`
- 系统重启 / WT 闪退
- 所有 tab 的工作目录、分屏布局、Claude 会话都需要从头摆一遍

### 2. 多 tab 看不出谁完成了

- 后台 tab 默默跑完，没有任何视觉信号
- 不知道哪个 Claude 已经 idle 等输入
- 必须一个个切过去才能确认状态，长此以往非常浪费时间

---

## 思路：三层防御

| 层 | 责任 | 工具 |
|---|------|------|
| 第一层（tab 布局） | 关 WT 自动恢复 + 误关单 tab 一键 undo | WT `firstWindowPreference` + `Ctrl+Shift+Z` 绑 `RestoreLastClosed` |
| 第二层（会话内容） | Claude 对话不丢 | Claude Code 自带 per-turn JSONL 落盘（**无需配置，免费送**） |
| 第三层（状态可见） | 完成的 tab 立刻发出信号 | Stop hook 响铃 `\a` + Notification idle hook 弹 OSC 9 Toast |

### 关键洞察

- **响铃 `\a` 是 tab 级精准信号** —— WT 只对发出响铃的 tab 加铃铛图标 + 标题闪烁，不是窗口级广播
- **Notification hook (`idle_prompt`) 比 Stop hook 更精准** —— Stop 每轮 LLM 响应都触发，`idle_prompt` 只在 Claude 真正停下等输入时触发
- **OSC 9 序列**（`\033]9;text\007`）直接弹 Windows 原生 Toast，不用装 BurntToast 等第三方模块
- **Hook 输出必须写 `/dev/tty`** —— Claude Code 把 hook 的 stdout/stderr 捕获到 debug log，不会送到终端，必须直写 `/dev/tty` 才能让 WT 看到 BEL 和 OSC 9 字节

---

## 使用方法

### 1. Windows Terminal `settings.json`

打开 WT → Settings → 右下角 "Open JSON file"，按 [`configs/windows-terminal.snippet.jsonc`](configs/windows-terminal.snippet.jsonc) 三处合并到自己的 `settings.json`：

| 位置 | 改动 |
|------|------|
| 顶层 | 加 `"firstWindowPreference": "persistedWindowLayout"` |
| `keybindings` 数组 | 追加 `Ctrl+Shift+Z` 绑 `Terminal.RestoreLastClosed` |
| `profiles.defaults` | 加 `"bellStyle": ["window", "taskbar"]` |

### 2. Claude Code `~/.claude/settings.json`

按 [`configs/claude-hooks.snippet.jsonc`](configs/claude-hooks.snippet.jsonc) 把 `Stop` 和 `Notification` 两个 hook 合并到自己的 `hooks` 块，**保留已有的 `PreToolUse` 等其他 hook**。

### 3. 验证

| 测试 | 方法 | 预期 |
|------|------|------|
| 响铃可达 | `printf '\a' > /dev/tty` | 听到响铃 / 看到 tab 铃铛图标 |
| Toast 可达 | `printf '\033]9;test\007' > /dev/tty` | 桌面右下角弹 Windows Toast |
| `Ctrl+Shift+Z` 撤销 | 开 tab → `Alt+Shift+D` 分屏 → `Ctrl+Shift+W` 关 → `Ctrl+Shift+Z` | tab 复活，分屏布局完整保留 |
| Stop hook 响铃 | 跑 `claude` 发 prompt，等响应完切到别的 tab | 原 tab 出现铃铛图标 + taskbar 闪 |
| Notification idle Toast | 等 Claude 进入 idle 等输入状态 | 桌面右下角弹「Claude [项目名] 等你输入」 |

---

## 备选方案（为什么不选）

| 方案 | 否决原因 |
|------|---------|
| WSL2 + tmux | Windows 下 tmux 体验差（要配 `vmIdleTimeout=-1`，工作流割裂） |
| zellij Windows 原生 | 无 detach/reattach 能力，关 Terminal 进程就死，等于没装 |
| AutoHotkey 拦截 WM_CLOSE | 需要写 / 维护脚本，成本太高 |
| BurntToast / SnoreToast 等第三方 toast | OSC 9 原生支持，加依赖没必要 |
| VSCode 扩展多 session 管理 | 一个 VSCode 窗口只能连 1 个 IDE-Connected 实例 |
| `firstWindowPreference` 单独使用 | 只恢复 shell 布局，不接 Claude 会话；需要配合 `claude -c` 才完整 |

---

## 详细技术决策

参见 [`docs/design-notes.md`](docs/design-notes.md) — 包含 hook stdout 路由验证、Notification matcher schema、WT keybinding 新旧两种 form 的选择依据等。

---

## 兼容性

- Windows Terminal v1.21+（`Terminal.RestoreLastClosed` action id 形式）
- Claude Code v2.x（`Notification` hook + `idle_prompt` matcher）
- Git Bash on Windows（`/dev/tty` 写入 + `$CLAUDE_PROJECT_DIR` 环境变量）

PowerShell / cmd 作为 Claude Code 宿主未测试，理论上 `/dev/tty` 在 Git Bash 之外可能需要替换为其他 tty 写法。

---

## License

[MIT](LICENSE)
