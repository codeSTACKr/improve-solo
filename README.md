# improve-solo

A fork of [shadcn/improve](https://github.com/shadcn/improve) — an agent skill that audits any codebase and writes implementation plans for other agents to execute — with **pluggable persistence**. By default it behaves exactly like upstream (plans as files). With `--solo` it stores and dispatches natively through [Solo](https://soloterm.com): plans become scratchpad + todo pairs, dependencies become real todo blockers, executors are Solo agents, and review wake-ups use Solo idle timers.

The idea is unchanged: use your most capable model for the part where intelligence compounds — understanding the codebase, judging what's worth doing, writing the spec — and hand execution to cheaper models. The skill never implements anything itself. The plan is the product.

```
files (default)  →  plans/001-fix-n-plus-one.md     self-contained spec files
--solo           →  scratchpad "Plan 001: …"        the document
                    + todo "Plan 001: …"            status, priority, blockers, comments
--issues         →  GitHub issues                   additive distribution on either store
```

## Storage backends

Exactly **one primary store** per run; `--issues` is additive on top of either.

| | files (default) | `--solo` |
|---|---|---|
| Plan body | `plans/NNN-slug.md` | scratchpad `Plan NNN: <title>` |
| Index / status | `plans/README.md` table | the todo list itself (`improve-plan` tag) |
| Dependencies | prose in the index | native todo blockers |
| BLOCKED / REJECTED | status column | tags `blocked` / `rejected` + reason comments |
| Audit memory (rejected findings) | `plans/README.md` section | per-run audit-report scratchpad |
| Executor (`execute`) | host subagent, isolated worktree | `spawn_agent` in an advisor-created worktree |
| Executor report | subagent's final reply | todo comment |
| Review wake-up | inline | `timer_fire_when_idle_any` |

**One backlog, never two:** if prior plans exist in the other store, the skill says so and follows the existing store rather than forking the backlog.

## Installation

In Claude Code:

```
/plugin marketplace add codeSTACKr/improve-solo
/plugin install improve-solo@improve-solo
```

Or for local development, point the marketplace at your checkout:
`/plugin marketplace add /path/to/improve-solo`.

If you previously copied the skill to `~/.claude/skills/improve-solo`
manually, remove that copy after installing the plugin — two registrations
of the same skill name will conflict or silently shadow each other.

## Usage

```
/improve-solo                        full audit → prioritized findings → plans (files)
/improve-solo --solo                 same, stored and dispatched through Solo
/improve-solo quick|deep             effort level
/improve-solo security               focused audit (also: perf, tests, bugs, ...)
/improve-solo branch                 audit only what the current branch changes
/improve-solo next                   feature suggestions — where to take the project
/improve-solo plan <description>     skip the audit, spec one thing
/improve-solo review-plan <plan>     critique and tighten an existing plan
/improve-solo execute <plan>         dispatch a cheaper executor, review its work
/improve-solo reconcile              refresh the backlog: verify, unblock, retire
/improve-solo ... --issues           also publish plans as GitHub issues
```

All modifiers compose: `/improve-solo deep security --solo --issues`.

## How `--solo` mode works

- **Phase 0**: verifies the selected Solo project matches the repo (`whoami` / `select_project`; creates a project if none exists). Never writes into a mismatched project.
- **Plans**: each is a scratchpad (full plan: drift check, steps, verification gates, STOP conditions) paired with a todo (priority P1/P2/P3 → high/medium/low; blockers = dependency graph). Each half links the other's ID. NNN numbering stays monotonic across runs.
- **Audit record**: one scratchpad per run with recon facts, the vetted findings table, what wasn't audited, and rejected findings — so the next run doesn't re-audit dead ends.
- **`execute`**: the advisor creates a git worktree, spawns a Solo agent, and delivers the plan *by scratchpad ID* (worktrees are linked checkouts, which share project-scoped scratchpads/todos in Solo ≥ 0.8.2). The executor posts its report as a todo comment and never touches status — the advisor wakes on an idle timer, re-runs every done criterion itself, and renders APPROVE / REVISE (`send_input`, max 2 rounds) / BLOCK. Merging stays yours.
- **`reconcile`**: reads the todo list + latest audit report, refreshes drifted plans with section-level scratchpad edits, investigates stale in-progress executors (resumable sessions included), retires findings fixed independently.
- **Deliberately not used**: KV store, terminal processes, solo.yml commands — nothing in the workflow needs them.

Audit fan-out still uses the host's lightweight read-only subagents in both modes — Solo agents are reserved for executors, where durability and visibility pay for themselves.

## Hard rules (unchanged from upstream)

- Never modifies source code itself. Writes go only to `plans/` (files mode) or Solo artifacts (`--solo`); executors edit only in disposable worktrees, and merging is always yours.
- Never runs commands that mutate your working tree — read, search, and read-only analysis only.
- Never reproduces secret values. Locations and credential types only, rotation always recommended.
- Asked to implement? It declines and points at the plan (or offers `execute`).

## Layout

```
.claude-plugin/plugin.json
skills/improve-solo/
  SKILL.md                          the workflow (backend-aware)
  references/
    audit-playbook.md               what to look for, per category (verbatim from upstream)
    plan-template.md                the handoff plan template (+ solo notes)
    closing-the-loop.md             execute / reconcile / --issues (backend-aware)
    solo-backend.md                 NEW — the full Solo artifact model and dispatch mechanics
examples/                           example plan output (from upstream)
```

## License

MIT © shadcn (original), Solo fork additions © Jesse Hall. See [LICENSE.md](LICENSE.md).
