# Agent Teams — Master Reference Guide

> Source: Claude Code docs v2.1.186+  
> Last updated: 2026-06-28

---

## What Are Agent Teams?

Agent teams coordinate multiple Claude Code instances working together. One session acts as **team lead** — it spawns teammates, coordinates work via a shared task list, and synthesizes results. Each teammate runs in its own context window and can communicate directly with other teammates.

> **Status:** Experimental, disabled by default.

---

## Agent Teams vs. Subagents — Quick Decision

| Factor | Use Subagents | Use Agent Teams |
|---|---|---|
| Workers need to talk to each other | No | Yes |
| Workers only report results back | Yes | No |
| Token cost | Lower | Higher (each teammate = full Claude instance) |
| Best for | Focused, bounded tasks | Complex work needing discussion and collaboration |
| Coordination | Main agent manages everything | Shared task list + self-coordination |

**Rule of thumb:** if teammates don't need to share findings or challenge each other, use subagents. If they do, use a team.

---

## Enable Agent Teams

Add to `settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Or set as a shell environment variable.

---

## Architecture

| Component | Role |
|---|---|
| **Team lead** | Main Claude Code session; spawns teammates, assigns tasks, synthesizes results |
| **Teammates** | Separate Claude Code instances; each has its own context window |
| **Task list** | Shared work item list that teammates claim and complete |
| **Mailbox** | Messaging system for direct inter-agent communication |

### File Locations

```
~/.claude/teams/{team-name}/config.json   # Runtime state (auto-managed, don't edit)
~/.claude/tasks/{team-name}/              # Task list (persists across sessions)
```

- Team name = `session-` + first 8 chars of session ID
- Team config directory is removed when session ends
- Task list directory persists locally (never uploaded); controlled by `cleanupPeriodDays`

---

## Display Modes

| Mode | How it works | Requirements |
|---|---|---|
| `in-process` (default) | All teammates run inside your main terminal | Any terminal |
| `auto` | Split panes if tmux/iTerm2 detected, else in-process | tmux or iTerm2 |
| `tmux` | Split panes via tmux or iTerm2 (auto-detect) | tmux or iTerm2 |
| `iterm2` | iTerm2 native split panes explicitly | `it2` CLI + Python API enabled |

Set globally in `~/.claude/settings.json`:

```json
{ "teammateMode": "auto" }
```

Or per-session:

```bash
claude --teammate-mode auto
```

### In-Process Controls

| Key | Action |
|---|---|
| ↑ / ↓ | Select a teammate |
| Enter | Open that teammate's transcript + message it |
| Escape | Interrupt selected teammate's current turn |
| `x` | Stop selected teammate |
| Ctrl+T | Toggle task list |

> Idle teammates hide after 30 seconds but stay running — send them a message to bring the row back.

---

## Spawning Teammates

### Natural Language (Recommended)

```text
I'm designing a CLI tool. Spawn three teammates:
- One on UX
- One on technical architecture
- One playing devil's advocate
```

Claude auto-populates the task list and spawns teammates based on the description.

### Specify Count and Model

```text
Spawn 4 teammates to refactor these modules in parallel. Use Sonnet for each teammate.
```

Teammates don't inherit the lead's model by default. To change the default:
- Set **Default teammate model** in `/config`
- Pick **Default (leader's model)** to have them follow the lead's model

> Teammates inherit the lead's **effort level** (v2.1.186+).

### Using Subagent Definitions as Teammates

Reference a named subagent type for reusable roles:

```text
Spawn a teammate using the security-reviewer agent type to audit the auth module.
```

- The teammate honors that definition's `tools` allowlist and `model`
- The definition body appends to the teammate's system prompt (doesn't replace it)
- `SendMessage` and task tools are always available regardless of `tools` restrictions
- `skills` and `mcpServers` from the definition are NOT applied — those come from project/user settings

---

## Task Management

Tasks have three states: **pending → in progress → completed**.

Tasks can have **dependencies** — a task with unresolved dependencies cannot be claimed until they complete.

### How Tasks Get Claimed

- **Lead assigns**: tell the lead explicitly which task goes to which teammate
- **Self-claim**: after finishing, a teammate picks up the next unassigned, unblocked task
- File locking prevents race conditions when multiple teammates try to claim simultaneously

### Task Sizing

- **Too small** → coordination overhead exceeds benefit
- **Too large** → teammates work too long without check-ins, wasted effort risk
- **Just right** → self-contained unit with a clear deliverable (a function, a test file, a review)

**Sweet spot:** 5–6 tasks per teammate.

---

## Communication

- Teammates send messages directly to each other by name
- Messages are delivered automatically — no polling required
- When a teammate goes idle, it automatically notifies the lead
- To reach everyone: send one message per recipient (no broadcast)
- Lead assigns names when spawning; specify names in your prompt if you want predictable references

---

## Plan Approval Workflow

Require a teammate to plan before implementing:

```text
Spawn an architect teammate to refactor the authentication module.
Require plan approval before they make any changes.
```

Flow:
1. Teammate works read-only in plan mode
2. Teammate sends plan approval request to lead
3. Lead reviews and approves or rejects with feedback
4. If rejected → teammate revises and resubmits
5. If approved → teammate exits plan mode and implements

Influence the lead's judgment via prompt criteria:
- `"only approve plans that include test coverage"`
- `"reject plans that modify the database schema"`

---

## Hooks for Quality Gates

| Hook | Trigger | Exit 2 behavior |
|---|---|---|
| `TeammateIdle` | Teammate about to go idle | Send feedback, keep teammate working |
| `TaskCreated` | Task being created | Prevent creation, send feedback |
| `TaskCompleted` | Task being marked complete | Prevent completion, send feedback |

Use these to enforce standards automatically — e.g., block task completion if tests don't pass.

---

## Permissions

- Teammates start with the **lead's permission settings**
- If lead runs with `--dangerously-skip-permissions`, all teammates do too
- You can change individual teammate modes after spawning
- You **cannot** set per-teammate modes at spawn time

> Pre-approve common operations in permission settings before spawning to reduce interruption friction.

---

## Context Rules

- Each teammate loads project context fresh: `CLAUDE.md`, MCP servers, skills
- Lead's **conversation history does NOT carry over** to teammates
- Teammates receive the spawn prompt from the lead

Always include task-specific context in the spawn prompt:

```text
Spawn a security reviewer teammate with the prompt: "Review the authentication module
at src/auth/ for vulnerabilities. Focus on token handling, session management,
and input validation. The app uses JWT tokens stored in httpOnly cookies.
Report issues with severity ratings."
```

---

## Token Cost Guidance

- Token costs scale **linearly** with teammate count
- Each teammate has its own full context window
- Worth it for: research, review, parallel implementation of independent modules
- Not worth it for: sequential tasks, same-file edits, many dependencies between tasks

**Start with 3–5 teammates.** Scale up only when work genuinely benefits from simultaneity.

---

## Proven Use Case Patterns

### 1. Parallel Code Review (Multi-lens)

```text
Spawn three teammates to review PR #142:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

Why it works: clear independent domains, no file conflicts, lead synthesizes at end.

### 2. Competing Hypothesis Debugging

```text
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific debate.
Update the findings doc with whatever consensus emerges.
```

Why it works: adversarial structure fights anchoring bias. The theory that survives debate is more likely to be correct.

### 3. New Feature Parallel Build

```text
Spawn 3 teammates to build the notification system:
- Teammate A: backend API endpoints
- Teammate B: frontend components
- Teammate C: test suite
Each owns distinct files. No overlaps.
```

Why it works: clear file ownership prevents conflicts. Lead integrates at the end.

### 4. Research from Multiple Angles

```text
Spawn three teammates to research library X from different angles:
- One on UX/developer experience
- One on technical architecture and performance
- One playing devil's advocate (finding problems and downsides)
```

Why it works: diverse perspectives surface what a single investigator would miss.

---

## Best Practices Checklist

- [ ] Enable via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: "1"` in settings
- [ ] Start with 3–5 teammates, not more
- [ ] Give each teammate a clear, independent domain (no file conflicts)
- [ ] Include all necessary context in the spawn prompt (no conversation history carries over)
- [ ] Use 5–6 tasks per teammate for good utilization
- [ ] Pre-approve common operations to reduce permission prompts
- [ ] Use hooks (`TeammateIdle`, `TaskCreated`, `TaskCompleted`) for automated quality gates
- [ ] Define reusable roles as subagent definitions; reference them in spawn prompts
- [ ] Monitor teammates' progress; don't let teams run unattended too long
- [ ] If lead starts working instead of waiting → tell it: `"Wait for your teammates to finish before proceeding"`

---

## Limitations (Experimental)

| Limitation | Workaround |
|---|---|
| `/resume` and `/rewind` don't restore in-process teammates | Tell lead to spawn new teammates after resuming |
| Task status can lag (teammate fails to mark complete) | Tell lead to nudge the teammate or update manually |
| Shutdown can be slow (finishes current request first) | Plan for delay; don't force-kill unless necessary |
| One team per session | Can't create named teams or share across sessions |
| No nested teams (teammates can't spawn teammates) | Only the lead manages the team |
| Lead is fixed for session lifetime | Can't promote a teammate or transfer leadership |
| Split panes require tmux or iTerm2 | Use in-process mode in VS Code / Windows Terminal / Ghostty |

---

## Troubleshooting

**Teammates not appearing:**
- In-process mode: check agent panel below the prompt input (↑↓ to select, Enter to open)
- Hidden idle row: send the teammate a message by name to wake it
- Check if the task was complex enough to warrant a team
- For split panes: run `which tmux` to confirm it's in PATH

**Too many permission prompts:**
Pre-approve operations in permission settings before spawning.

**Teammate stopped on error:**
Select it in agent panel → Enter to view output → give it new instructions or spawn a replacement.

**Lead shut down before work finished:**
Tell it to keep going. Also useful to say at spawn time: `"Wait for all teammates to finish before you summarize."`

**Orphaned tmux session:**
```bash
tmux ls
tmux kill-session -t <session-name>
```

---

## Quick Reference Card

```
ENABLE:   "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" in settings.json
SPAWN:    Natural language — "Spawn 3 teammates: one for X, one for Y, one for Z"
SIZE:     3–5 teammates, 5–6 tasks per teammate
CONTEXT:  Teammates don't inherit lead's history — put everything in spawn prompt
CONFLICT: Each teammate must own distinct files
HOOKS:    TeammateIdle / TaskCreated / TaskCompleted for automated quality gates
LIMIT:    One team per session, no nested teams, no session resumption with in-process
```

---

## Related

- [Subagents](https://code.claude.com/docs/en/sub-agents) — lighter delegation, no inter-agent communication
- [Git Worktrees](https://code.claude.com/docs/en/worktrees) — manual parallel sessions without automated coordination
- [Hooks](https://code.claude.com/docs/en/hooks) — `TeammateIdle`, `TaskCreated`, `TaskCompleted`
- [Settings](https://code.claude.com/docs/en/settings) — `teammateMode`, `cleanupPeriodDays`
- [Token Costs](https://code.claude.com/docs/en/costs#agent-team-token-costs) — usage guidance
