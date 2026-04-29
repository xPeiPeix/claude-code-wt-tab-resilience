# 设计决策与技术细节

## 已验证的技术问题

### Q1. Hook stdout/stderr 是否会到达终端？

**否，全部被 Claude Code 捕获到 debug log。**

[官方文档](https://docs.anthropic.com/en/docs/claude-code/hooks) 明确写明：

> For most events, stdout is written to the debug log but not shown in the transcript. The exceptions are UserPromptSubmit, UserPromptExpansion, and SessionStart, where stdout is added as context that Claude can see and act on.

对 `Stop` 和 `Notification` 事件，stdout 和 stderr **都不会到达终端**。

**v0.1.0 的 `/dev/tty` 方案为什么不工作**：试图绕过 stdout 捕获写到终端设备，但 Claude Code v2.x 的 hook 子进程**没有分配 controlling tty**。Git Bash 实测：

```bash
$ printf '\a' > /dev/tty
/usr/bin/bash: line 1: /dev/tty: No such device or address
```

`|| true` 兜底吞掉了错误，所以用户感知是 hook 静默失败。

**结论：响铃 / OSC 9 / 任何依赖 stdout 路由的方案都不通**。必须用**不依赖 stdout 的方案** —— 即调系统 API 产生副作用（弹 toast、播放音频等）。

### Q2. Notification hook matcher 的 6 种事件类型

| matcher | 触发场景 | 本仓库匹配 | Toast 文案 |
|---------|---------|----------|-----------|
| `idle_prompt` | Claude 完成响应，等用户输入 | ✅ | `Claude Code` / `Claude [项目名] 等你输入` |
| `permission_prompt` | Claude 请求权限确认（PreToolUse 黄条） | ✅ | `Claude Code · 等待授权` / `Claude [项目名] 需要批准命令` |
| `elicitation_dialog` | Claude 中途问用户问题（AskUserQuestion / Plan 审批） | ✅ | `Claude Code · 询问中` / `Claude [项目名] 等你决策` |
| `auth_success` | 认证成功 | ❌ | — |
| `elicitation_complete` | 用户回答完 elicitation | ❌ | — |
| `elicitation_response` | elicitation 响应 | ❌ | — |

**多 matcher 设计动机**：v0.2.0 同时匹配三个关键事件，每个 toast 用不同的 Title 区分，主人扫一眼通知就知道事件类型。`auth_success` 等次要事件不匹配，避免噪声。`elicitation_complete` / `elicitation_response` 是用户响应后的事件，不需要再通知主人。

### Q3. `shell: "powershell"` 字段

Claude Code v2.x hook 配置支持 `shell` 字段：

```json
{
  "type": "command",
  "shell": "powershell",
  "command": "..."
}
```

- 不指定 `shell` 时默认在 bash（Linux/macOS）或系统 shell（Windows）执行
- 指定 `"powershell"` 时直接在 PowerShell 中执行 command 字符串
- **避免 bash 嵌入 PowerShell 的多层引号转义噩梦**

### Q4. `$CLAUDE_PROJECT_DIR` 在 hook 上下文里可用吗？

可用。在 PowerShell shell 下访问为 `$env:CLAUDE_PROJECT_DIR`。fallback 用 `$PWD`：

```powershell
$d = if ($env:CLAUDE_PROJECT_DIR) {$env:CLAUDE_PROJECT_DIR} else {$PWD}
$p = Split-Path -Leaf $d
```

兼容 PowerShell 5.1+（避免用 `??` null coalescing 操作符 —— 那是 PowerShell 7+ 才有）。

---

## v0.1.0 → v0.2.0 根因复盘

### v0.1.0 推荐方案（不工作）

- **Stop hook** 发 BEL（`\a`） → WT `bellStyle: ["window", "taskbar"]` 触发视觉响铃
- **Notification idle_prompt** 发 OSC 9（`\033]9;text\007`） → 桌面 Toast
- 命令通过 `> /dev/tty` 绕过 Claude Code 的 stdout 捕获

### 根本错误

上述方案均依赖 hook 命令的输出字节到达终端设备，但：

1. **Claude Code 把 hook stdout/stderr 全部捕获**到 debug log（文档明文）
2. **Hook 子进程无 controlling tty**（`/dev/tty` 写入直接报错，无设备）
3. `|| true` 兜底掩盖了失败，导致用户以为 hook 没触发

**任何"让字节到 terminal 触发 WT 行为"的方案都不可行。**

### v0.2.0 修正路径

- ❌ 通过 stdout 触发 WT bellStyle 视觉信号 → 此路彻底关闭
- ✅ 直接调 Windows 系统通知 API → BurntToast 弹现代 toast

### 衍生影响

- **`bellStyle` 配置无意义**（hook 永远不会发 BEL 到 terminal），v0.2.0 移除
- **Stop hook 删除** —— 即使路径通也每轮触发太烦，留 Notification idle_prompt 即可

---

## 替代方案对比（基于真实可工作性）

| 方案 | 状态 | 备注 |
|------|------|------|
| `printf '\a'` 到 stdout | ❌ 不工作 | stdout 捕获到 debug log |
| `printf '\a' > /dev/tty` | ❌ 不工作 | hook 子进程无 controlling tty |
| `printf '\a' >&2` | ⚠️ 未确认 | 文档说 stderr 给用户看，但应是 transcript 不是 raw 字节 |
| `[System.Media.SystemSounds]::Beep.Play()` PowerShell | ✅ 工作 | 系统音频，每轮太烦不适合 Stop |
| `[System.Windows.Forms.NotifyIcon]::ShowBalloonTip()` | ✅ 工作 | 老式 Win7 气泡通知，无依赖但样式过时 |
| `New-BurntToastNotification`（BurntToast 模块） | ✅ **本仓库选用** | 现代 Win11 toast 风格，需一次性 `Install-Module` |
| WinRT `Windows.UI.Notifications` 内联 PowerShell | ⚠️ 复杂 | 代码长，未必比 BurntToast 简单 |

---

## `restoreLastClosed` 行为细节

源码层面（[`microsoft/terminal`](https://github.com/microsoft/terminal) 的 `TabManagement.cpp` 和 `Pane.cpp`）：

- 关闭 tab 时调用 `tab->BuildStartupActions(BuildStartupKind::None)`，递归遍历整棵 pane 树
- 每个 split 序列化为 `SplitPane` action，含：方向（H/V）、比例（`1.0 - _desiredSplitPosition`）、profile GUID、实时 CWD
- 触发 `restoreLastClosed` 时回放这些 action

**会保留**：分屏结构、profile、工作目录
**不会保留**：跑着的进程（重新 spawn shell）、scrollback 历史、tab 在 tab bar 里的位置

栈深 100 个，单独关闭的 pane 也能恢复（pane close 和 tab close 推同一个栈）。

---

## 风险与回滚

### 主要风险

| 风险 | 缓解 |
|------|-----|
| WT settings.json JSON 漏逗号 | WT 解析失败会弹 toast 提示并回退到 defaults，不破坏使用 |
| BurntToast 安装失败 | 切换到 NotifyIcon balloon 备选方案（见替代方案对比） |
| Notification hook 频率过高 | 后续可加前台窗口检测 / 冷却时间机制（v0.3.0 候选） |
| Windows 通知中心被静音 | 检查"专注助手"/"Focus Assist" 设置 |

### 回滚方式

- **单独禁 Notification toast**：删除 `Notification` 数组或把 matcher 改成不存在值
- **全回滚到无 hook 状态**：从 `~/.claude/settings.json` 删除整个 `Notification` 键
- **改前先备份**：`cp settings.json settings.json.bak`

---

## 不在本仓库范围

- WSL2 + tmux 完整方案（Windows 下体验欠佳）
- IDE 集成（VSCode / JetBrains 扩展）
- 第三方 tmux dashboard（`tmux-agent-status`、`opensessions` 等）
- 前台窗口检测 / 冷却时间机制（v0.3.0 候选优化）
