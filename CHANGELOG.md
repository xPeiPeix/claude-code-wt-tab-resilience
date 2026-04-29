# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2026-04-29

### Added

- 初版仓库结构与文档
- **Windows Terminal 配置 snippet**（`configs/windows-terminal.snippet.jsonc`）
  - 顶层 `firstWindowPreference: "persistedWindowLayout"` — 关闭 WT 时自动保存窗口/tab/分屏布局，下次启动自动还原
  - `keybindings` 追加 `Ctrl+Shift+Z` → `Terminal.RestoreLastClosed` — 撤销关闭 tab/pane（栈深 100，含分屏布局/profile/CWD）
  - `profiles.defaults.bellStyle: ["window", "taskbar"]` — 响铃时窗口闪 + taskbar 图标闪（无音频）
- **Claude Code hook 配置 snippet**（`configs/claude-hooks.snippet.jsonc`）
  - `Stop` hook 发 BEL（`\a`）→ 后台 tab 自动加铃铛 glyph
  - `Notification` hook (`matcher: "idle_prompt"`) 发 OSC 9 序列 → 弹 Windows 原生 Toast「Claude [项目名] 等你输入」
  - 命令均写到 `/dev/tty` 绕过 Claude Code 的 stdout 捕获
- **设计文档**（`docs/design-notes.md`）
  - 4 个已验证的技术问题（hook 输出路由、Notification matcher schema、`$CLAUDE_PROJECT_DIR` 可用性、WT keybinding 新旧 form）
  - Stop vs Notification hook 的语义差异
  - `restoreLastClosed` 源码层面的行为细节（`microsoft/terminal` `TabManagement.cpp` / `Pane.cpp`）
  - OSC 9 Toast 协议说明
  - 风险与回滚指引
- MIT License
- README 含痛点描述、三层防御思路、使用方法、备选方案对比、兼容性说明

[Unreleased]: https://github.com/xPeiPeix/claude-code-wt-tab-resilience/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/xPeiPeix/claude-code-wt-tab-resilience/releases/tag/v0.1.0
