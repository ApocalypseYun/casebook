# Casebook

**多 Agent 协作的结构化评论协议。**

把 issue 评论区从「Agent 水群」变成「可审计的案卷」。

[English](./README.md)

```
npx skills add ApocalypseYun/casebook
```

---

## 问题

多 Agent 协作的评论区长这样：

> Agent A："我做完了。"
> Agent B："我理解了。"
> Agent C："看起来没问题。"

没人知道：证据在哪？谁在负责？到底卡在哪了？真的做完了吗？

**Casebook** 定义了 agent 在每个生命周期节点必须说什么——并用一份共享的结构化案卷文件来兜底。

## 工作原理

**两层架构：**

| 层 | 内容 | 面向谁 |
|----|------|--------|
| **评论层** | 自包含的执行摘要 | 人类浏览评论区 |
| **案卷层** | `~/.casebook/<issue-id>/CASE.md`，包含实时 Clipboard + 只追加 Timeline | Agent 和审计 |

**六个生命周期节点：**

| 节点 | 触发时机 | 评论内容 |
|------|---------|---------|
| `CLAIM` | Agent 接手 issue | "我来做这个。计划、风险、工作量评估。" |
| `HANDOFF` | Agent 交付给下一个角色 | "做完了。改了什么、证据、@下一位 请做这件具体的事。" |
| `BLOCKED` | Agent 无法继续 | "卡住了。原因、谁能解决、不解决的后果。" |
| `REVIEW` | 审查者给出裁决 | "裁决：通过/驳回。发现的问题（附 file:line）。必须修复的事项。" |
| `ALERT` | 监控发现异常 | "出问题了。影响范围、谁应该处理。" |
| `CLOSE` | Issue 完结 | "已解决。问题/之前/之后/影响/交付物/经验教训。" |

**四种角色类型**，带字段级写权限控制：

| 角色 | 可写字段 | 不可写 |
|------|---------|--------|
| `executor` | phase, owner, blocker, pending-action, last-evidence | verdict |
| `reviewer` | phase, verdict, pending-action, last-evidence | owner |
| `monitor` | blocker, pending-action | owner, verdict, phase |
| `support` | pending-action, last-evidence | owner, verdict, phase |

## 快速开始

### 1. 安装

```bash
npx skills add ApocalypseYun/casebook
```

### 2. 在 agent 指令中声明角色

给每个 agent 的 system prompt 或 instructions 加一行：

```
My casebook role: executor
```

（可选：`executor`、`reviewer`、`monitor`、`support`。不声明默认为 `executor`。）

### 3. 完事

Agent 接下来会：
- 在每个生命周期事件时发结构化评论
- 创建 `~/.casebook/<issue-id>/CASE.md`，顶部是实时 Clipboard 状态
- 维护一个只追加的 Timeline 作为审计线索

## Clipboard（实时状态看板）

每个 CASE.md 顶部都有一张当前状态快照：

```markdown
## Clipboard

| Field          | Value                | Updated by  | At                   |
|----------------|----------------------|-------------|----------------------|
| phase          | in_review            | dev-agent   | 2026-05-14T10:30:00Z |
| owner          | reviewer-agent       | dev-agent   | 2026-05-14T10:30:00Z |
| blocker        |                      |             |                      |
| pending-action | Review cache logic   | dev-agent   | 2026-05-14T10:30:00Z |
| verdict        |                      |             |                      |
| last-evidence  | commit abc1234       | dev-agent   | 2026-05-14T10:30:00Z |
```

任何 agent 读 10 行就知道：谁在负责、什么阶段、有没有阻塞、下一步该谁做什么。不用翻 20 条评论。

## 评论效果对比

**用 Casebook 之前：**
> 架构师："设计好了。@开发者 请接手。"
> 开发者（12小时后）："确认阻塞中。"
> watchdog："任务卡住了。"

**用 Casebook 之后：**

> ## HANDOFF
>
> 我的部分做完了。改了什么、需要注意什么、下一步具体要做什么。
>
> **变更内容**：
> - 在 `user-service/internal/service/` 新增 `BatchGetUserInfo` tRPC 端点
> - Redis MGET 缓存层，含 DB 回源和空值防穿透
>
> **证据**：
> - Commit：`abc1234` — feat: add batch user query with cache
> - 测试：14 通过，0 失败，覆盖率 87%
>
> **未解决的风险**：
> - `user_info.biz_id` 索引在生产环境尚未确认
>
> **下一步**：@reviewer — 审查 `batch_get_user_info.go` 中的缓存失效逻辑，尤其是第 142-168 行的 MGET 回源路径
>
> 📋 phase=in_review | owner=@reviewer | pending-action=review cache logic | last-evidence=commit abc1234

## 合规监控

Casebook 内置 10 条 lint 规则，供 monitor 角色审计：

| 规则 | 严重性 |
|------|--------|
| Issue 标记完成但没有 CLOSE 评论 | CRITICAL |
| HANDOFF 缺少证据链接 | HIGH |
| Agent 写了超出角色权限的字段 | HIGH |
| Issue 已分配但 5 分钟内没有 CLAIM | HIGH |
| BLOCKED 超过 24 小时无后续跟进 | HIGH |
| Clipboard 的 phase 与 Timeline 最新条目不一致 | MEDIUM |

完整规则表见 `SKILL.md`。

## 灵感来源

- [The Clipboard Pattern](https://novaberg.de/papers/clipboard-pattern.html) — 类型化共享状态 > agent 之间用自然语言传话
- 「案卷」隐喻 — agent 不是在聊天，是在维护一份卷宗

## 平台兼容性

Casebook 与平台无关。只要你的系统能：
1. 在 issue 上发评论（任意 issue tracker）
2. 读写共享文件系统上的文件（`~/.casebook/`）

已验证：[Multica](https://multica.ai)、Claude Code、Codex、OpenCode。

## 许可证

MIT
