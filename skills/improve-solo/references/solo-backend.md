# Solo Backend — storage and dispatch when Solo is the store

This file defines how the advisor workflow persists and dispatches when Solo is the primary store (the default whenever the Solo MCP tools are available; `--files` opts out — see SKILL.md for resolution order). The workflow itself (recon → audit → vet → plan → execute/reconcile) is unchanged from SKILL.md; only the *where* and the *how of handoff* differ. Everything here assumes the Solo MCP tools are available (`whoami`, `scratchpad_*`, `todo_*`, `timer_*`, `spawn_agent`, `send_input`, `get_process_output`). If they aren't, say so and use the files store.

**Scope discipline:** use a Solo primitive only where it's the right tool. This backend uses scratchpads, todos (+ tags, blockers, comments, locks), `spawn_agent`/`send_input`, idle timers, and process inspection. It deliberately does **not** use the KV store, terminal processes, or solo.yml commands — nothing in this workflow needs them. They remain available for ad-hoc needs the user raises, but don't shoehorn them in.

---

## Phase 0 — project scoping (before any Solo write)

Solo artifacts are project-scoped. Writing into the wrong project files the backlog somewhere nobody will look.

1. Call `whoami` (and `list_projects` if needed). Confirm the selected project's path matches the repo being audited.
2. Mismatch, but a matching project exists → `select_project` it.
3. No matching project → `create_project` for the repo and announce that you did.
4. **Never write artifacts into a mismatched project.** If something is ambiguous (e.g. two projects point at parent/child paths), ask.

Note: git worktrees are linked checkouts, and Solo shares project-scoped todos and scratchpads across linked checkouts (Solo ≥ 0.8.2). An executor spawned to work in a worktree sees the same plan scratchpads and todos as the advisor — no extra scoping work needed.

## Artifact model

### Plan = scratchpad + todo pair

Each plan is **one scratchpad** (the document) plus **one todo** (the work item). Created together in Phase 4:

- **Scratchpad** — name `Plan NNN: <imperative title>`, tags `["improve", "improve-plan", "<category>"]`. Content is the full plan body from [plan-template.md](plan-template.md), unchanged: Status block, Planned-at SHA, drift check, steps, done criteria, STOP conditions — all of it. Add one line to the plan's Status block: `- **Todo**: <todo_id>`.
- **Todo** — title `Plan NNN: <imperative title>`, same tags. Priority maps P1→`high`, P2→`medium`, P3→`low`. Body is a short summary (2–3 sentences of "Why this matters") plus `Scratchpad: <scratchpad_id>` — the body is a pointer, not a copy; the scratchpad is the single source of the plan text.
- **Dependencies** — encode "Depends on" with real blockers: `todo_set_blockers(todo_id, [blocking todo ids])`. This replaces the index's dependency-graph prose; `todo_list(is_blocked=...)` answers "what's executable right now" natively.

The pair must be discoverable from either side (scratchpad → todo id in Status block; todo → scratchpad id in body).

### NNN numbering

NNN stays monotonic per project across runs, exactly like file numbering upstream. To assign the next number, list existing `improve-plan`-tagged todos and take max(NNN)+1. Humans and invocations refer to plans by NNN (`execute 003`), so resolve `NNN → pair` by title prefix.

### Status mapping

Solo todos have a fixed status enum; the two missing states are expressed with tags + comments. **Never delete plan pairs — they're the record** (archive is also wrong for plans; `backlog` keeps rejected ones queryable).

| Upstream state | Solo representation |
|---|---|
| TODO | status `open` |
| IN PROGRESS | status `in_progress` |
| DONE | status `completed` |
| BLOCKED | status `open` + tag `blocked` + a comment stating the reason |
| REJECTED | status `backlog` + tag `rejected` + a comment with the one-line rationale |

Filters stay cheap: `todo_list(tags=["improve-plan"])` is the index; add `tags=["blocked"]` or `status` filters as needed.

### Audit-report scratchpad (one per run)

After Phase 3 vetting, write one scratchpad — name `Audit <YYYY-MM-DD>[ <focus>]`, tags `["improve", "improve-audit"]` — containing:

- Recon facts (stack, verification commands, conventions) — these feed every plan.
- The vetted findings table as presented to the user.
- Direction findings.
- What was NOT audited at this effort level.
- **Findings considered and rejected**, with one-line reasons — this is the cross-run "don't re-audit" memory that `plans/README.md` carried upstream.

Re-runs and `reconcile` read the latest audit-report scratchpad (and the plan todo list) before auditing anything, to skip planned and rejected findings. Do **not** persist raw subagent findings — they're leads, not facts; only the vetted record is stored.

## `execute <plan>` — Solo dispatch

Principles are in [closing-the-loop.md](closing-the-loop.md) (preconditions, review standards, verdict table — read it first). The Solo mechanics:

### Preconditions

Same as upstream, restated for Solo: repo is a git repository; the plan pair exists; every blocker todo is `completed`; run the plan's drift check yourself and reconcile first if in-scope files changed since `Planned at`.

### Dispatch

1. **Create the worktree yourself**: `git worktree add <path> -b advisor/NNN-<slug>` (this writes `.git` metadata and a new directory only — the user's checkout is untouched; allowed by Hard Rule 2).
2. Set the todo `in_progress` (`todo_update`). You may take a short `todo_lock` while editing the todo itself; don't hold it across the run.
3. `list_agent_tools`, then `spawn_agent(agent_tool_id=...)` — default to the cheaper runtime/model unless the user named one (`execute 003 haiku`). Prepend the returned `agent_instructions` to the first prompt.
4. Send the executor prompt via `send_input`. Because the scratchpad is readable from the worktree's shared project scope, **deliver the plan by ID, not by inlining**. The prompt must contain:
   - the `agent_instructions` from `spawn_agent` (verbatim, first),
   - the worktree path and an instruction to work only there,
   - the plan scratchpad id and todo id, with: "Read the full plan with `scratchpad_read(<id>, mode=full)` before doing anything",
   - the executor preamble from closing-the-loop.md, with the reporting override adapted: *instead of replying with the report, post it as a todo comment* — `todo_comment_create(<todo_id>, <report in the standard STATUS/STEPS/STOPPED BECAUSE/FILES CHANGED/NOTES format>)` — and **never change the todo's status, tags, or completion; the reviewer owns those.**
5. Schedule the wake-up: `timer_fire_when_idle_any([<executor process id>], <max_wait_ms ~30–60 min>, "<self-contained body: executor for Plan NNN (process id, todo id, scratchpad id, worktree path) went idle — fetch the report comment via todo_comment_list and begin review; if there is no report comment, check get_process_output">)`. Timers are the wake-up mechanism — no polling loops. `get_process_output` is still available for an ad-hoc peek if the user asks how it's going.

### Review and verdict

Review exactly as closing-the-loop.md prescribes (re-run every done criterion in the worktree, scope check via `git -C <worktree> diff --stat`, read the full diff, audit the tests). Then:

- **APPROVE** → post a verdict comment on the todo, `todo_complete(todo_id, true)`. Present to the user: diff summary, worktree path and branch, anything from NOTES. Merging is the user's decision — never merge, push, or commit to their branch.
- **REVISE** → post the specific feedback as a todo comment *and* `send_input` it to the same executor process. Re-arm an idle timer. Max 2 revision rounds, then BLOCK.
- **BLOCK** → post the reason as a comment, add tag `blocked` (status stays `open`), refine or rewrite the plan scratchpad with what was learned. Tell the user.

**The advisor is the only writer of status, tags, and completion.** The executor writes exactly one thing: its report comment. (Upstream keeps index-writing away from executors because they game and misreport; the same stance, ported.)

Locks: completing a todo releases the *completing actor's* lock only. The executor doesn't need to lock or unlock anything — if it took a lease, the TTL (default 300s) expires it.

When the run is over, cancel any leftover timers (`timer_list` / `timer_cancel`) and `close_process` the executor once its session is no longer useful — but prefer leaving a STOPPED executor's process around until `reconcile` has investigated it.

## `reconcile` — Solo edition

Same per-status duties as closing-the-loop.md; the mechanics:

- Read the latest audit-report scratchpad and `todo_list(tags=["improve-plan"])` (all statuses, including `backlog`).
- **DONE** (`completed`) — spot-check cheap done criteria on current HEAD; note verification in a todo comment.
- **BLOCKED** (tag `blocked`) — read the reason comment, investigate the obstacle, then either refresh the plan or mark REJECTED (`backlog` + tag swap + rationale comment). If the approach changed fundamentally, write a new pair with a new NNN and note the supersession in both.
- **IN PROGRESS** (stale) — an executor probably died mid-run. Check `list_processes` / `get_process_output` for its process; a stopped Solo agent with a saved session can be resumed from the Solo UI ("Resume last session"), which may be cheaper than re-dispatching. Check the worktree if one exists. Flag it to the user either way.
- **TODO** (`open`) — run the drift check. If drifted: re-verify the finding still exists, then refresh the scratchpad's "Current state" excerpts and `Planned at` SHA. If fixed independently: REJECTED (`backlog` + `rejected` + comment).

**Edit scratchpads surgically, never wholesale.** Drift refreshes use `scratchpad_edit` (section or line-range replacement with the revision guard) or `scratchpad_append_section` — not a full `scratchpad_write` overwrite. This is both the safe-concurrency path and the tool-guided one (Solo ≥ 0.8.2 steers toward direct edits).

Finish with the same short report: verified done, refreshed, rejected, executable now (`todo_list` with `is_blocked=false`, status `open` answers the last one).

## `--issues` under Solo

Additive distribution, same as upstream, sourced from scratchpads:

1. Preflight `gh auth status` + GitHub remote; on failure, keep the Solo artifacts and say why issues were skipped.
2. Per plan: `scratchpad_save_to_file(<scratchpad_id>, <tmp path>)`, then `gh issue create --title "<plan title>" --body-file <tmp path>`; delete the temp file. Labels per closing-the-loop.md.
3. Record the URL twice: in the scratchpad's Status block (`- **Issue**: <url>`, via `scratchpad_edit` on that section) and as a todo comment.

## Ad-hoc export

If a plan must be handed to an executor with no Solo MCP access, `scratchpad_save_to_file` writes it as a plain markdown file in one call. This is the one user-gated exception to Hard Rule 1's "never repo files": only on explicit user request, and default the path outside the repo (e.g. a temp dir) unless the user names a repo path such as `plans/NNN-slug.md`. That file is a one-way snapshot — Solo remains the source of truth; don't maintain both.
