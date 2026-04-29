# Claude Code on Windows Terminal — Tab Resilience Toolkit

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform: Windows](https://img.shields.io/badge/Platform-Windows-blue.svg)](#)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-v2.x-orange.svg)](https://docs.anthropic.com/claude/docs/claude-code)
[![Windows Terminal](https://img.shields.io/badge/Windows%20Terminal-v1.21+-blue.svg)](https://github.com/microsoft/terminal)
[![PowerShell](https://img.shields.io/badge/PowerShell-5.1+-blue.svg)](https://learn.microsoft.com/powershell/)

> **三层防御** — 让 Windows Terminal × Claude Code 多 tab 工作流不再脆弱
>
> 五分钟搞定：tab 误关一键复活、整个 WT 关了自动恢复布局、Claude idle 时桌面 Toast 主动提醒「哪个项目等输入」

在 Windows Terminal 里跑多个并发 Claude Code session 是常见工作流，但有两个反复折磨人的痛点。本仓库给出一套配置加一个 PowerShell 模块的解法。

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
| 第二层（会话内容） | Claude 对话不丢 | Claude Code 自带 per-turn JSONL 落盘（**无需配置**） |
| 第三层（状态可见） | Claude idle 时桌面通知 | Notification `idle_prompt` hook + PowerShell + BurntToast |

### 关键技术点

- **三个 matcher 覆盖关键事件**：Notification hook 同时匹配 `idle_prompt`（等输入）/ `permission_prompt`（等授权）/ `elicitation_dialog`（等决策），**每种 toast 文案不同便于一眼区分**；其他次要事件（`auth_success` / `elicitation_complete` / `elicitation_response`）不触发，避免噪声
- **PowerShell + BurntToast 弹原生 Win11 toast**，**不走** terminal escape sequences —— Claude Code v2.x 的 hook 子进程没有 controlling tty，stdout/stderr 全部捕获到 debug log，BEL（`\a`）和 OSC 9 序列**到不了** Windows Terminal
- **`shell: "powershell"` 字段**让 hook 命令直接在 PowerShell 中执行，避免 bash 嵌入 PowerShell 的引号转义噩梦
- 所有提示通过**调系统 API 产生副作用**，不依赖 stdout 路由

---

## 使用方法

### 1. 安装 BurntToast 模块（一次性）

```powershell
Install-Module BurntToast -Scope CurrentUser -Force
```

或在 Git Bash / cmd 一行命令：

```bash
pwsh -NoProfile -Command "Install-Module BurntToast -Scope CurrentUser -Force"
```

[BurntToast](https://github.com/Windos/BurntToast) 是 PowerShell Gallery 里的开源模块，封装 Windows.UI.Notifications 弹原生 toast。

### 2. Windows Terminal `settings.json`

打开 WT → Settings → 右下角 "Open JSON file"，按 [`configs/windows-terminal.snippet.jsonc`](configs/windows-terminal.snippet.jsonc) 两处合并到自己的 `settings.json`：

| 位置 | 改动 |
|------|------|
| 顶层 | 加 `"firstWindowPreference": "persistedWindowLayout"` |
| `keybindings` 数组 | 追加 `Ctrl+Shift+Z` 绑 `Terminal.RestoreLastClosed` |

### 3. Claude Code `~/.claude/settings.json`

按 [`configs/claude-hooks.snippet.jsonc`](configs/claude-hooks.snippet.jsonc) 把 `Notification` hook 合并到自己的 `hooks` 块（**保留已有的 `PreToolUse` 等其他 hook**）。

### 4. 验证

| 测试 | 方法 | 预期 |
|------|------|------|
| BurntToast 直接弹 | `pwsh -NoProfile -Command "New-BurntToastNotification -Text 'test'"` | 桌面右下角弹 toast |
| `Ctrl+Shift+Z` 撤销关闭 | 开 tab → `Alt+Shift+D` 分屏 → `Ctrl+Shift+W` 关 → 按 `Ctrl+Shift+Z` | tab 复活带分屏布局 |
| 自动恢复布局 | 开 3 tab + 1 个分屏 → 关 WT → 重开 WT | 全部回来 |
| `idle_prompt` Toast | Claude 完成响应进入 idle | 弹「Claude Code」+「Claude [项目名] 等你输入」 |
| `permission_prompt` Toast | Bash 命令需要批准时（PreToolUse 黄条） | 弹「Claude Code · 等待授权」+「Claude [项目名] 需要批准命令」 |
| `elicitation_dialog` Toast | Claude 用 AskUserQuestion 问 / Plan 等审批 | 弹「Claude Code · 询问中」+「Claude [项目名] 等你决策」 |

> ⚠️ Claude Code 配置改动需要**新启动 `claude` session**才会 reload，重启 terminal 不必要

---

## 备选方案（为什么不选）

| 方案 | 否决原因 |
|------|---------|
| WSL2 + tmux | Windows 下 tmux 体验差（要配 `vmIdleTimeout=-1`，工作流割裂） |
| zellij Windows 原生 | 无 detach/reattach，关 Terminal 进程就死 |
| AutoHotkey 拦截 WM_CLOSE | 需要写 / 维护脚本，成本高 |
| 在 hook 里 `printf '\a'` 响铃 | **路径不通** — Claude Code 把 hook stdout 捕获到 debug log，不会送到 terminal |
| 在 hook 里写 `/dev/tty` | **不可达** — hook 子进程在 Git Bash 里没有 controlling tty，`No such device or address` |
| Stop hook 触发提醒 | 每轮 LLM 响应都触发，频率太高（含中间工具调用结束）；改用 Notification idle_prompt 更精准 |
| NotifyIcon balloon | Windows 7 风格气泡通知，丑；BurntToast 给现代 Win11 风格 toast |
| VSCode 扩展多 session 管理 | 一个 VSCode 窗口只能连 1 个 IDE-Connected 实例 |

---

## 详细技术决策

参见 [`docs/design-notes.md`](docs/design-notes.md) — 含 hook stdout 路由权威结论、`Notification` 6 种事件类型对比、v0.1.0 → v0.2.0 根因复盘。

变更历史见 [`CHANGELOG.md`](CHANGELOG.md)。

---

## 兼容性

- Windows Terminal v1.21+（modern keybinding `id` form 支持 `Terminal.RestoreLastClosed`）
- Claude Code v2.x（hook `shell` 字段 + `Notification.idle_prompt` matcher 支持）
- PowerShell 5.1+（Windows 默认）或 PowerShell 7+
- [BurntToast](https://github.com/Windos/BurntToast) PowerShell 模块（一次性 `Install-Module`）

---

## License

[MIT](LICENSE)
