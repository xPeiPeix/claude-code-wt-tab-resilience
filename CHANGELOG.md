# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.0] - 2026-04-29

### Fixed

- **关键修复**：v0.1.0 推荐的 `/dev/tty` 路由方案在 Claude Code v2.x hook 子进程里无法工作 —— hook 子进程没有 controlling tty 分配，Git Bash 实测 `printf '\a' > /dev/tty` 报 `No such device or address`，被 `|| true` 兜底吞掉，用户感知是 hook 完全没触发
- **文档错误**：v0.1.0 错误推断 Stop hook 能通过响铃路径触发 WT `bellStyle` 视觉信号 —— 实际上 Claude Code 把 hook 的 stdout/stderr 全部捕获到 debug log（[官方文档](https://docs.anthropic.com/en/docs/claude-code/hooks) 明确写明），**没有任何路径让字节到达 terminal**

### Changed

- **替换核心通知方案**：从 BEL（`\a`） / OSC 9 转义序列改为 PowerShell + BurntToast 直接调 Windows 原生 toast API
- Hook 配置使用 `shell: "powershell"` 字段直接在 PowerShell 中执行 command（避免 bash 嵌入 PowerShell 的引号转义）
- README / design-notes 增加 `Notification` 6 种 matcher 事件类型对比，明确 `idle_prompt` 的精准触发场景

### Added

- **Notification hook 三个 matcher 区分场景**：`idle_prompt`（等输入）/ `permission_prompt`（等授权）/ `elicitation_dialog`（等决策），每个 toast 用不同 Title 一眼区分（`Claude Code` / `Claude Code · 等待授权` / `Claude Code · 询问中`）
- BurntToast PowerShell 模块作为依赖（一次性 `Install-Module BurntToast -Scope CurrentUser -Force`）
- README "备选方案" 增加多个**已实测不工作**的路径及原因（stdout / `/dev/tty` / NotifyIcon balloon 等）
- `docs/design-notes.md` 增加 v0.1.0 → v0.2.0 根因复盘章节
- README 强调「Claude Code 配置改动需要新启动 `claude` session 才生效」

### Removed

- WT `profiles.defaults.bellStyle: ["window", "taskbar"]` 配置（响铃通知机制无法通过 hook 触发，配置无意义）
- Claude Code `Stop` hook（响铃路径不通，且每轮 LLM 响应都触发频率过高）
- 原 `printf '\a' > /dev/tty 2>/dev/null || true` 命令模式

### Migration from v0.1.0

如果已按 v0.1.0 部署：

1. 安装 BurntToast：`pwsh -NoProfile -Command "Install-Module BurntToast -Scope CurrentUser -Force"`
2. 删除 `~/.claude/settings.json` 里的 `Stop` hook 整个块
3. 替换 `Notification` hook 的 command 为 v0.2.0 的 PowerShell 版本（参见 `configs/claude-hooks.snippet.jsonc`）
4. 删除 WT `settings.json` 里的 `profiles.defaults.bellStyle`
5. 新开一个 `claude` session 让新配置生效

## [0.1.0] - 2026-04-29

> ⚠️ v0.1.0 推荐的 hook 配置经实测不工作，**已被 v0.2.0 修正**。请勿使用本版本配置。

### Added

- 初版仓库结构与文档
- **Windows Terminal 配置 snippet**（`configs/windows-terminal.snippet.jsonc`）
  - 顶层 `firstWindowPreference: "persistedWindowLayout"` — 关闭 WT 时自动保存窗口/tab/分屏布局，下次启动自动还原
  - `keybindings` 追加 `Ctrl+Shift+Z` → `Terminal.RestoreLastClosed` — 撤销关闭 tab/pane（栈深 100，含分屏布局/profile/CWD）
  - `profiles.defaults.bellStyle: ["window", "taskbar"]` —（**v0.2.0 移除**：响铃路径不通）
- **Claude Code hook 配置 snippet**（`configs/claude-hooks.snippet.jsonc`）
  - `Stop` hook 发 BEL（`\a`）—（**v0.2.0 移除**：路径不通 + 频率过高）
  - `Notification` hook (`matcher: "idle_prompt"`) 发 OSC 9 —（**v0.2.0 重写**：改用 PowerShell + BurntToast）
  - 命令均写到 `/dev/tty` —（**v0.2.0 修正**：hook 子进程无 controlling tty）
- **设计文档**（`docs/design-notes.md`）
- MIT License
- README 含痛点描述、三层防御思路、使用方法、备选方案对比、兼容性说明

[Unreleased]: https://github.com/xPeiPeix/claude-code-wt-tab-resilience/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/xPeiPeix/claude-code-wt-tab-resilience/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/xPeiPeix/claude-code-wt-tab-resilience/releases/tag/v0.1.0
