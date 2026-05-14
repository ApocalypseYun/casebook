# Casebook

**Structured commenting protocol for multi-agent collaboration.**

Turn issue threads from unreadable agent chatter into auditable case files.

```
npx skills add ApocalypseYun/casebook
```

---

## The Problem

Multi-agent workflows produce threads like this:

> Agent A: "I'm done."
> Agent B: "I understand."
> Agent C: "Looks fine."

Nobody knows: where's the evidence? Who owns it? What's blocking? Is it really done?

**Casebook** fixes this by defining exactly what agents must say at each lifecycle moment — and backing it with a shared, structured case file.

## How It Works

**Two layers:**

| Layer | What | For whom |
|-------|------|----------|
| **Comments** | Self-contained executive summaries | Humans scanning the thread |
| **Case File** | `~/.casebook/<issue-id>/CASE.md` with live Clipboard + append-only Timeline | Agents and auditors |

**Six lifecycle nodes:**

| Node | When | What the comment says |
|------|------|----------------------|
| `CLAIM` | Agent picks up an issue | "I own this. Here's my plan, risks, and effort estimate." |
| `HANDOFF` | Agent passes work to the next role | "Done. Here's what I changed, evidence, and exactly what @next should do." |
| `BLOCKED` | Agent can't continue | "Stuck. Here's why, who can fix it, and impact if unresolved." |
| `REVIEW` | Reviewer delivers a verdict | "Verdict: APPROVE/REJECT. Findings with file:line. Required fixes." |
| `ALERT` | Monitor detects an anomaly | "Something's wrong. Here's the impact and who should act." |
| `CLOSE` | Issue is done | "Solved. Problem/Before/After/Impact/Deliverables/Lessons." |

**Four role types** with field-level write permissions:

| Role | Writes | Can't write |
|------|--------|-------------|
| `executor` | phase, owner, blocker, pending-action, last-evidence | verdict |
| `reviewer` | phase, verdict, pending-action, last-evidence | owner |
| `monitor` | blocker, pending-action | owner, verdict, phase |
| `support` | pending-action, last-evidence | owner, verdict, phase |

## Quick Start

### 1. Install

```bash
npx skills add ApocalypseYun/casebook
```

### 2. Declare role in agent instructions

Add one line to each agent's system prompt or instructions:

```
My casebook role: executor
```

(Options: `executor`, `reviewer`, `monitor`, `support`. Defaults to `executor` if omitted.)

### 3. That's it

Agents will now:
- Post structured comments at each lifecycle event
- Create `~/.casebook/<issue-id>/CASE.md` with a live Clipboard header
- Maintain an append-only Timeline as audit trail

## The Clipboard

Every CASE.md has a live state snapshot at the top:

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

Any agent reads 10 lines and knows: who owns this, what phase it's in, what's blocking, what needs to happen next. No scanning through 20 comments.

## What Comments Look Like

**Before casebook:**
> architect: "Design is done. @dev please take over."
> dev (12h later): "Confirmed blocked."
> watchdog: "Task stuck."

**After casebook:**

> ## HANDOFF
>
> My part is done. Here's what I did, what to watch out for, and exactly what needs to happen next.
>
> **What changed**:
> - Added `BatchGetUserInfo` tRPC endpoint in `user-service/internal/service/`
> - Redis MGET cache layer with DB fallback and null-value protection
>
> **Evidence**:
> - Commit: `abc1234` — feat: add batch user query with cache
> - Tests: 14 passed, 0 failed, coverage 87%
>
> **Unresolved risks**:
> - `user_info.biz_id` index not yet confirmed in production
>
> **Next step**: @reviewer — review the cache invalidation logic in `batch_get_user_info.go`, especially the MGET fallback path on lines 142-168
>
> 📋 phase=in_review | owner=@reviewer | pending-action=review cache logic | last-evidence=commit abc1234

## Compliance Monitoring

Casebook includes 10 lint rules for a monitor agent to audit:

| Rule | Severity |
|------|----------|
| Issue marked done without CLOSE comment | CRITICAL |
| HANDOFF without evidence links | HIGH |
| Agent wrote a field outside its role permissions | HIGH |
| Issue assigned but no CLAIM within 5 minutes | HIGH |
| BLOCKED entry older than 24h without follow-up | HIGH |
| Clipboard phase doesn't match latest Timeline entry | MEDIUM |

See `SKILL.md` for the full lint rules table.

## Inspired By

- [The Clipboard Pattern](https://novaberg.de/papers/clipboard-pattern.html) — typed shared state > natural language messaging between agents
- The "case file" metaphor — agents don't chat, they maintain a dossier

## Platform Compatibility

Casebook is platform-agnostic. It works with any system where agents can:
1. Post comments on issues (any issue tracker)
2. Read/write files on a shared filesystem (`~/.casebook/`)

Tested with: [Multica](https://multica.ai), Claude Code, Codex, OpenCode.

## License

MIT
