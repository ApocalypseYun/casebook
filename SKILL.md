---
name: casebook
description: >
  Structured commenting protocol for multi-agent collaboration. Turns issue threads
  from unreadable agent chatter into auditable case files. Governs how agents write
  comments across 6 lifecycle nodes (CLAIM, HANDOFF, BLOCKED, REVIEW, ALERT, CLOSE),
  maintains a live Clipboard header for current state, enforces role types
  (executor/reviewer/monitor/support) with field write permissions, and keeps an
  append-only Timeline for evidence archival. Platform-agnostic — works with any
  issue tracker and any agent team structure.
---

# Casebook — Case File Protocol for Multi-Agent Collaboration

> Issue is a case file, not a chatroom.
> Agents don't converse — they append evidence, post verdicts, and pass the baton.

## Why This Exists

Multi-agent workflows produce unreadable threads. One agent says "I'm done." Another says "I understand." A reviewer says "Looks fine." Meanwhile:

- Where's the evidence?
- Who owns forward progress?
- What's actually blocking?
- Is this really done?

Casebook replaces this with structured lifecycle comments and a shared case file that any agent — or human — can read in 5 seconds.

## Core Principle

**Comments are reports, not signals.**

Each comment is a self-contained executive summary. A human who reads ONLY the comment thread — without opening code, logs, or case files — fully understands the situation: what happened, why, what's next, and who owns it.

The case file is the deep archive. Comments are the narrative.

## Two-Layer Architecture

```
COMMENT LAYER (issue tracker comments)
  → Self-contained executive summary for humans
  → A person reads ONLY comments and fully understands
  → Agent-parseable via ## headings + structured fields

CASE FILE LAYER (~/.casebook/<issue-id>/CASE.md)
  → Live state via Clipboard header
  → Complete evidence chain in append-only Timeline
  → Shared via local filesystem between agents
```

| Layer | Audience | Purpose |
|-------|----------|---------|
| **Comment** | Humans, PMs, leads | Executive summary — tells the full story |
| **Case File** | Agents, auditors | Live state + structured evidence archive |

## Role Types

Four abstract roles. Your agent declares which one it is.
If not declared, defaults to `executor`.

| Role | Responsibility | Writable Clipboard Fields |
|------|---------------|---------------------------|
| **executor** | Accepts tasks, does work, delivers artifacts | phase, owner, blocker, pending-action, last-evidence |
| **reviewer** | Audits work, delivers verdicts | phase, verdict, pending-action, last-evidence |
| **monitor** | Watches health, audits compliance | blocker, pending-action |
| **support** | Auxiliary work (docs, research) | pending-action, last-evidence |

**Hard constraints**:

- Only **executor** writes `owner`.
- Only **reviewer** writes `verdict`.
- **monitor** and **support** never write `owner`, `verdict`, or `phase`.
- `pending-action` is writable by all roles.

Declare your role in your agent instructions:

```
My casebook role: executor
```

## CASE.md Structure

Created once when the first lifecycle entry is written:

```markdown
# Case File: <issue-id>

- **Title**: <issue title>
- **Created**: <ISO-8601 timestamp>
- **Issue**: <issue-id>

## Clipboard

| Field          | Value | Updated by | At |
|----------------|-------|------------|----|
| phase          |       |            |    |
| owner          |       |            |    |
| blocker        |       |            |    |
| pending-action |       |            |    |
| verdict        |       |            |    |
| last-evidence  |       |            |    |

## Timeline
```

### Clipboard Fields

| Field | Purpose | Example Values |
|-------|---------|----------------|
| phase | Current lifecycle phase | claimed, in_progress, blocked, in_review, approved, needs_work, done |
| owner | Agent currently responsible | agent-name or "unassigned" |
| blocker | Active impediment (empty if none) | "CI requires Node 20+ approval" |
| pending-action | What must happen next | "Review cache logic in replica.ts:142-168" |
| verdict | Latest review outcome (empty if not reviewed) | approve, reject, conditional |
| last-evidence | Most recent artifact | commit hash, PR link, test result |

## Write Protocol

Every lifecycle event requires three actions in this order:

1. **Append to Timeline** — append-only, never edit previous entries. Each entry begins with `---` followed by `## <NODE_TYPE>`.
2. **Update Clipboard** — read the full CASE.md, update the relevant Clipboard rows per your role permissions, write the file back.
3. **Post comment** — the human-readable executive summary with state line.

**Compose in memory first** — build the full entry before writing. Single write operation for the Timeline append + Clipboard update.

**Backward compatibility**: If CASE.md exists but has no `## Clipboard` section, insert the Clipboard template between the header metadata and `## Timeline` on the next write.

### Clipboard Updates by Node Type

| Node | phase → | owner → | blocker → | pending-action → | verdict → | last-evidence → |
|------|---------|---------|-----------|-----------------|-----------|-----------------|
| CLAIM | claimed / in_progress | self | _(clear)_ | plan summary | — | — |
| HANDOFF | in_review | handoff-to | _(clear)_ | next action for receiver | — | commit/PR/test |
| BLOCKED | blocked | — | description | unblock instruction | — | — |
| REVIEW | approved / needs_work | — | — | required fixes (if any) | approve/reject/conditional | findings summary |
| ALERT | — | — | alert detail (if applicable) | recommended action | — | — |
| CLOSE | done | — | _(clear)_ | _(clear)_ | — | final artifacts |

_(clear)_ = set field to empty. — = do not write (leave unchanged).

---

## Lifecycle Nodes

### 1. CLAIM — Accepting Ownership

**When**: Agent is assigned or self-assigns an issue.

**Status change**: Set issue status to `in_progress` in your issue tracker.

**Clipboard update**: phase=claimed/in_progress, owner=self, blocker=(clear), pending-action=plan summary.

**Case file entry** (append to Timeline):

```markdown
---
## CLAIM

- time: <ISO-8601>
- agent: <agent-name>
- owner: <agent-name>
- scope: <what this issue requires — specific, not generic>
- plan: <numbered steps 1-5>
- risks: <what might go wrong>
- estimated-effort: <time estimate with justification>
```

**Comment format**:

> I'm picking this up. Here's what I'm going to do and what might go wrong.
>
> **Scope**: <Specific description of what needs to happen — not "fix the bug" but
> "the cache invalidation in `session-store` fails to propagate TTL changes to
> replica nodes when the primary is under write pressure">
>
> **Plan**:
> 1. <Step with concrete action>
> 2. <Step with concrete action>
> 3. <Step with concrete action>
>
> **Risks**: <What might block or complicate this — be specific>
>
> **Effort**: <Estimate> — <Justification for the estimate>
>
> 📋 phase=in_progress | owner=<agent-name> | pending-action=<plan summary>
>
> ---
> 📁 Case file: `~/.casebook/<issue-id>/CASE.md`

### 2. HANDOFF — Delivering Work to Next Role

**When**: Agent completes their portion and passes to the next agent.

**Clipboard update**: phase=in_review, owner=handoff-to, blocker=(clear), pending-action=next action, last-evidence=commit/PR.

**Case file entry** (append to Timeline):

```markdown
---
## HANDOFF

- time: <ISO-8601>
- agent: <agent-name>
- handoff-to: <next-agent or role>
- changes-summary: <bullet list of what changed>
- evidence: <commit hashes, PR links, test results>
- risks: <unresolved concerns>
- next-action: <specific instruction for next agent>

### Detail
<optional extended context, diffs, logs>
```

**Comment format**:

> My part is done. Here's what I did, what to watch out for, and exactly what
> needs to happen next.
>
> **What changed**:
> - <Specific change with file/component reference>
> - <Specific change with file/component reference>
>
> **Evidence**:
> - Commit: `<hash>` — <one-line description>
> - Tests: <result summary, e.g. "14 passed, 0 failed, coverage 87%">
> - PR: <link if applicable>
>
> **Unresolved risks**: <What the next person should watch out for>
>
> **Next step**: @<next-agent> — <Specific actionable instruction. NOT "please
> review" but "review the cache invalidation logic in `session-store/src/replica.ts`,
> especially lines 142-168 where TTL propagation was rewritten">
>
> 📋 phase=in_review | owner=<next-agent> | pending-action=<next action> | last-evidence=<commit/PR>
>
> ---
> 📁 Case file: `~/.casebook/<issue-id>/CASE.md`

### 3. BLOCKED — Reporting Impediment

**When**: Agent cannot continue due to an external dependency or decision.

**Clipboard update**: phase=blocked, blocker=description, pending-action=unblock instruction.

**Case file entry** (append to Timeline):

```markdown
---
## BLOCKED

- time: <ISO-8601>
- agent: <agent-name>
- blocker-type: <environment|dependency|decision|access|other>
- description: <full context of the impediment>
- who-can-unblock: <person or role with specific action>
- fallback-action: <what can be done meanwhile>
- deadline-risk: <impact if not resolved by when>
```

**Comment format**:

> I can't continue. Here's exactly what's blocking me and who can fix it.
>
> **Blocker** (`<type>`): <Full context — not "missing dependency" but "the
> `auth-service` v2.3 SDK requires Node 20+ but our CI pinned to Node 18;
> upgrading CI requires DevOps approval and a pipeline rebuild that takes ~2h">
>
> **Who can fix**: @<person> — <Specific action they need to take>
>
> **Meanwhile**: <What productive work can happen in parallel>
>
> **Impact if unresolved**: <Consequence with timeline — "if not resolved by
> Friday, the release train slips to next sprint">
>
> 📋 phase=blocked | blocker=<description> | pending-action=<unblock instruction>
>
> ---
> 📁 Case file: `~/.casebook/<issue-id>/CASE.md`

### 4. REVIEW — Audit Verdict

**When**: Agent completes a review of another agent's work.

**Clipboard update**: phase=approved/needs_work, verdict=approve/reject/conditional, pending-action=required fixes, last-evidence=findings summary.

**Case file entry** (append to Timeline):

```markdown
---
## REVIEW

- time: <ISO-8601>
- agent: <agent-name>
- verdict: <approve|reject|conditional>
- findings:
  - [CRITICAL] <finding>
  - [HIGH] <finding>
  - [MEDIUM] <finding>
  - [LOW] <finding>
- action-required: <specific fixes needed>
- escalation: <if any>

### Detail
<optional extended analysis, code snippets, references>
```

**Comment format**:

> **Verdict: <APPROVE|REJECT|CONDITIONAL>**
>
> <1-2 sentence overall assessment summary>
>
> **Findings**:
> - 🔴 CRITICAL: <finding with file:line reference>
> - 🟡 HIGH: <finding with file:line reference>
> - 🔵 MEDIUM: <finding with file:line reference>
>
> **Required actions**:
> - <Specific fix instruction with file:line and what to change>
> - <Specific fix instruction with file:line and what to change>
>
> **Next**: @<author> fix critical/high findings; @<lead> escalation if needed
>
> 📋 phase=<approved/needs_work> | verdict=<approve/reject/conditional> | pending-action=<fixes> | last-evidence=<findings>
>
> ---
> 📁 Case file: `~/.casebook/<issue-id>/CASE.md`

### 5. ALERT — Watchdog Notification

**When**: Automated monitoring detects an anomaly requiring attention.

**Clipboard update**: pending-action=recommended action, blocker=alert detail (if applicable).

**Case file entry** (append to Timeline):

```markdown
---
## ALERT

- time: <ISO-8601>
- agent: <agent-name>
- alert-type: <task_stuck|task_failed|format_violation|timeout|sla_breach|other>
- evidence: <specific data points>
- recommended-action: <what should happen>
- urgency: <critical|high|medium|low>
- addressed-to: <agent or role>
```

**Comment format**:

> **🚨 <URGENCY> — <alert-type>**
>
> <Plain-language explanation of what's wrong and WHY IT MATTERS — not "task
> failed" but explain the consequence>
>
> **Evidence**: <Specific data — timestamps, error codes, metrics>
>
> **Recommended action**: @<agent> — <Specific instruction to resolve>
>
> 📋 pending-action=<recommended action> | blocker=<alert detail>
>
> ---
> 📁 Case file: `~/.casebook/<issue-id>/CASE.md`

### 6. CLOSE — Case Closure

**When**: Issue is fully resolved and delivered.

**Status change**: Set issue status to `done` in your issue tracker.

**Clipboard update**: phase=done, blocker=(clear), pending-action=(clear), last-evidence=final artifacts.

This is the **most important comment** on the entire issue.

**Case file entry** (append to Timeline):

```markdown
---
## CLOSE

- time: <ISO-8601>
- agent: <agent-name>
- problem: <plain-language problem statement>
- before: <previous state>
- after: <new state>
- impact: <measurable numbers>
- ecosystem-benefit: <what this means beyond this issue>
- artifacts:
  - <type>: <link>
  - <type>: <link>
- landed-at: <where the change is deployed/merged>
- lessons: <optional retrospective insight>
```

**Comment format**:

> **Problem**: <Plain language, PM-readable — what was broken or missing>
>
> **What we did**:
> - Before: <Previous state — concrete, not abstract>
> - After: <New state — concrete, not abstract>
>
> **Impact**: <Measurable numbers — latency reduction, error rate change,
> coverage improvement, user-facing improvement>
>
> **What this means for the ecosystem**: <Broader benefit beyond this ticket>
>
> **Deliverables**:
> | Type | Link |
> |------|------|
> | PR | <link> |
> | Commit | <hash> |
> | Docs | <link> |
>
> **Landed at**: <environment/branch/version where this is live>
>
> **Lessons learned**: <Optional — what we'd do differently next time>
>
> 📋 phase=done | last-evidence=<final artifacts>
>
> ---
> 📁 Case file: `~/.casebook/<issue-id>/CASE.md`

---

## Lint Rules (Monitor Reference)

| Rule | Check | Severity |
|------|-------|----------|
| `MISSING_CLAIM` | Issue assigned but no CLAIM comment within 5 minutes | HIGH |
| `MISSING_HANDOFF` | Agent mentions another agent without structured HANDOFF | HIGH |
| `INCOMPLETE_FIELDS` | Required fields in case file entry are empty or placeholder | MEDIUM |
| `MISSING_EVIDENCE` | HANDOFF or CLOSE without commit hash, PR link, or test result | HIGH |
| `MISSING_CLOSE` | Issue marked done but no CLOSE comment | CRITICAL |
| `STALE_BLOCK` | BLOCKED entry older than 24h with no follow-up | HIGH |
| `NO_CASE_FILE` | Comment posted but no corresponding case file entry | MEDIUM |
| `FIELD_PERMISSION_VIOLATION` | Agent wrote Clipboard field outside its role permissions | HIGH |
| `STALE_CLIPBOARD` | Clipboard phase doesn't match latest Timeline entry | MEDIUM |
| `CLIPBOARD_MISSING` | CASE.md exists but has no Clipboard section | LOW |

---

## Quick Reference

```
CLAIM    → "I own this."          Clipboard: phase + owner + pending-action
HANDOFF  → "Done. @next do this." Clipboard: phase + owner + pending-action + last-evidence
BLOCKED  → "Stuck. @who unblock." Clipboard: phase + blocker + pending-action
REVIEW   → "Verdict: X."          Clipboard: phase + verdict + pending-action + last-evidence
ALERT    → "Something's wrong."   Clipboard: pending-action + blocker
CLOSE    → "Solved. Impact: Y."   Clipboard: phase + pending-action(clear) + last-evidence
```

Every comment tells the full story. Clipboard tells the current state. Timeline tells the full history.
