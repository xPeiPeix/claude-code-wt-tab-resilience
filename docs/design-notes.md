# 设计决策与技术细节

## 已验证的技术问题

### Q1. Hook stdout 是否会到达终端？

**否。** Claude Code 把 `Stop` 和 `Notification` hook 的 stdout 捕获到 debug log，不会转发到终端设备。

**结论：必须写 `/dev/tty`** 才能让 Windows Terminal 收到 BEL（`\a`）和 OSC 9 字节。stderr（`>&2`）也被捕获，所以也不行。

`/dev/tty` 是终端设备的特殊文件，只有交互式 shell session 有；`claude -p` 非交互模式没有，命令会失败 —— 因此用 `2>/dev/null || true` 兜底。

### Q2. Notification hook matcher 怎么写？

字符串 `"idle_prompt"`（不嵌套对象）。可选值是以下字面量：

- `permission_prompt` — Claude 请求权限确认
- `idle_prompt` — Claude 完成响应，等待用户输入（**本仓库使用**）
- `auth_success` — 认证成功
- `elicitation_dialog` / `elicitation_complete` / `elicitation_response` — elicitation 流程相关

### Q3. `$CLAUDE_PROJECT_DIR` 在 hook 上下文里可用吗？

可用。在 Stop 和 Notification hook 里都有。需要引号保护含空格的路径。fallback 用 `$PWD`：

```bash
p=$(basename "${CLAUDE_PROJECT_DIR:-$PWD}")
```

如果两者都不可靠，hook stdin 也包含完整 JSON payload（带 `cwd` 字段），可以用 Python / jq 解析（参考 Claude Code 现有 PreToolUse hook 的 `block-rm-rf.sh` 模式）。

### Q4. Windows Terminal keybinding 用 `id` form 还是 `command` form？

WT v1.21+ 支持两种：

```jsonc
// modern id form（推荐）
{ "id": "Terminal.RestoreLastClosed", "keys": "ctrl+shift+z" }

// legacy command form
{ "command": "restoreLastClosed", "keys": "ctrl+shift+z" }
```

新建配置一律用 `id` form，与 WT 自身 `defaults.json` 保持一致。`Terminal.RestoreLastClosed` 这个 id 在 defaults.json 里已定义但未绑定快捷键，所以用户自己绑就生效。

---

## Stop hook vs Notification hook 的语义差异

| 钩子 | 触发时机 | 频率 | 适合用途 |
|------|---------|------|---------|
| `Stop` | 每一轮 LLM 响应结束（包括中间工具调用完成） | 高 | 流程控制、质量门禁 |
| `Notification` (`idle_prompt`) | Claude 完全停下等用户输入 | 低 | **状态追踪首选** |

本仓库**两个都用**，分工不同：
- Stop hook 发响铃 `\a`，让后台 tab 出铃铛图标 —— 用户切回 Terminal 时一眼锁定哪些 tab 已完成本轮工作
- Notification idle hook 发 OSC 9 Toast —— 桌面级通知，主动推送给用户

---

## `restoreLastClosed` 的行为细节

源码层面（`microsoft/terminal` 仓库的 `TabManagement.cpp` 和 `Pane.cpp`）：

- 关闭 tab 时调用 `tab->BuildStartupActions(BuildStartupKind::None)`，递归遍历整棵 pane 树
- 每个 split 序列化为 `SplitPane` action，含：方向（H/V）、比例（`1.0 - _desiredSplitPosition`）、profile GUID、实时 CWD
- 触发 `restoreLastClosed` 时回放这些 action

**会保留**：分屏结构、profile、工作目录
**不会保留**：跑着的进程（重新 spawn shell）、scrollback 历史、tab 在 tab bar 里的位置

栈深 100 个，单独关闭的 pane 也能恢复（pane close 和 tab close 推同一个栈）。

---

## OSC 9 Toast 协议

格式：`\033]9;<message>\007`

- ConEmu 协议扩展，Windows Terminal 原生支持（v1.x 起）
- 弹出 Windows 系统级 Toast，受系统通知设置影响
- 点击 Toast 通常会聚焦到 WT 窗口（具体行为取决于 WT 版本）
- 老版本 WT 可能不识别此序列，会静默忽略（无错误）

参考：[ConEmu OSC 9 documentation](https://conemu.github.io/en/AnsiEscapeCodes.html#OSC_C_commands)

---

## 风险与回滚

### 主要风险

| 风险 | 缓解 |
|------|-----|
| WT settings.json JSON 漏逗号 | WT 解析失败会弹 toast 提示并回退到 defaults，不会破坏使用 |
| Stop hook 触发频率高 | 前台 tab 也会闪一下；嫌烦可以只保留 Notification idle hook |
| `bellStyle` 被 profile 单独覆盖 | 检查 `profiles.list[]` 里是否有同名 key，defaults 才会生效 |
| OSC 9 在很老的 WT 不识别 | 升级到 WT 1.21+ |

### 回滚方式

- **单独禁响铃**：把 Stop hook 的 command 改成 `true`
- **单独禁 Toast**：删除 `Notification` 数组或把 matcher 改成不存在值
- **全回滚**：移除上面所有新增配置（建议改前先 `cp settings.json settings.json.bak`）

---

## 不在本仓库范围

- WSL2 + tmux 完整方案（Windows 下体验欠佳，本仓库刻意不走这条路）
- IDE 集成（VSCode / JetBrains 扩展的多 session 管理）
- 第三方 tmux dashboard（`tmux-agent-status`、`opensessions` 等）

如有兴趣可参考相关项目，与本仓库的纯 settings.json 方案是替代关系而非补充。
