# Closing the Loop — execute, reconcile, issues

The advisor's job doesn't end at the plan. This file covers the three follow-through flows: dispatching an executor and reviewing its work (`execute`), keeping the plan backlog alive (`reconcile`), and publishing plans where work gets picked up (`--issues`).

The founding rule survives unchanged: **the advisor never edits source code.** In `execute`, a *separate executor agent* edits code in an isolated git worktree; the advisor dispatches, reviews, and renders a verdict — like a tech lead who doesn't push commits to your branch.

**Backend note:** this file describes the flows in files-store terms; the principles (preconditions, review standards, verdict table, reconcile duties) apply identically under the Solo store. Where the *mechanics* differ — Solo-agent dispatch, report-as-todo-comment, idle-timer wake-ups, status via todo updates — [solo-backend.md](solo-backend.md) overrides the mechanics described here. Read both before the first Solo-mode dispatch.

---

## `execute <plan>` — dispatch and review

### Preconditions (check all before dispatching)

- The repo is a git repository (worktree isolation requires it). If not: stop and say so.
- The plan exists and its dependencies show DONE in the index (`plans/README.md`; under solo: every blocker todo is `completed`). If not: stop, name the missing dependency.
- Run the plan's drift check yourself. If in-scope files changed since `Planned at`, reconcile the plan first (see below) — don't hand a stale plan to an executor.

### Dispatch

Files store: spawn **one** `general-purpose` subagent with `isolation: "worktree"`. Executor model: default `sonnet`; use what the user named if they named one (`execute 003 haiku`). (Solo store: `spawn_agent` into a worktree the advisor creates — see solo-backend.md.)

The subagent prompt must contain:

1. **The full plan text.** Files store: inlined — the worktree contains only committed files, so if `plans/` is uncommitted the executor can't read it; never assume, always inline. (Solo store: deliver the scratchpad id instead — linked checkouts share project scope, so the executor reads the plan via MCP.)
2. The executor preamble:

> You are the executor for the implementation plan below. Follow it step by
> step. Run every verification command and confirm the expected result before
> moving on. Touch only the files listed as in scope. If any STOP condition
> occurs, stop immediately and report. Do not improvise around obstacles.
> Commit your work in the worktree following the plan's git workflow section.
> One override: SKIP the plan's instruction to update the plan index —
> your reviewer maintains it. Before reporting, audit every claim in
> your report against an actual tool result from this session — only report
> what you can point to evidence for; if a verification failed or was
> skipped, say so plainly. When finished, reply with exactly the report
> format below.

(Solo store: the last sentence becomes "post the report in exactly the format below as a comment on todo <id>", and the override extends to never touching the todo's status, tags, or completion.)

3. The report format:

```
STATUS: COMPLETE | STOPPED
STEPS: per step — done/skipped + verification command result
STOPPED BECAUSE: (only if STOPPED) which STOP condition, what was observed
FILES CHANGED: list
NOTES: anything the reviewer should know (deviations, surprises, judgment calls)
```

### Review (the advisor's real job here)

Note on fresh worktrees: they share git history but not `node_modules` or build artifacts — the executor must install dependencies first, and check tooling that resolves from `dist/` may need one build even though the plan's command table (recon'd in the main tree) didn't mention it. Expect this; it isn't a deviation.

Review like a tech lead reviewing a PR against the spec — never fix anything yourself:

1. **Re-run every done criterion** in the worktree. Don't trust the executor's report — verify.
2. **Scope compliance**: `git -C <worktree> diff --stat` against the plan's in-scope list. Any file outside scope fails review, full stop.
3. **Read the full diff.** Judge it against "Why this matters" (does it solve the actual problem?) and the repo conventions named in the plan (does it look like the rest of the codebase?).
4. **Audit the new tests.** Executors game criteria — a test that asserts nothing meaningful passes `pnpm test` and proves nothing. Read what the tests assert.

### Verdict

**Documented deviations are judged on merit, not reflex-blocked.** "Do not improvise" exists to stop silent drift; an executor that hits a real obstacle (e.g. the plan's approach breaks existing test mocks), adapts minimally, and explains it in NOTES has done the right thing. Approve it if the adaptation serves the plan's intent and stays in scope; treat *undocumented* deviations as review failures.

| Verdict | When | Action |
|---|---|---|
| **APPROVE** | Criteria pass, scope clean, quality holds | Update index status to DONE (solo: verdict comment + `todo_complete`). Present to the user: diff summary, worktree path and branch, anything from NOTES. **Merging is the user's decision — never merge, push, or commit to their branch.** |
| **REVISE** | Fixable gaps | Send the same executor specific, actionable feedback ("criterion 3 fails: X; the error handling in `api.ts:90` swallows the error — use the Result pattern per the plan") — SendMessage under files, `send_input` + a fresh idle timer under solo. **Max 2 revision rounds**, then BLOCK. |
| **BLOCK** | STOP condition hit, scope violated unrecoverably, or revisions exhausted | Mark BLOCKED in the index with the reason (solo: comment + tag `blocked`). Refine or rewrite the plan with what was learned. Tell the user what happened and what changed in the plan. |

Running verification commands inside the executor's worktree is fine — it's isolated and disposable. The no-mutating-commands rule protects the user's working tree, not the worktree.

---

## `reconcile` — keep the backlog alive

Process what happened since the last session. **Follow whichever store the plans actually live in** (see the store-conflict rule in SKILL.md). Read the index and every plan (files: `plans/README.md` + plan files; solo: the latest audit-report scratchpad + `todo_list(tags=["improve-plan"])`, all statuses), then per status:

- **DONE** — spot-check that the done criteria still hold on the current HEAD (cheap ones only). Mark verified in the index. Don't delete plans — they're the record.
- **BLOCKED** — read the reason. Investigate the underlying obstacle in the codebase. Either rewrite the plan around it (new number if the approach changed fundamentally, in-place refresh otherwise) or mark REJECTED with one line of rationale.
- **IN PROGRESS** (stale) — flag it to the user; an executor probably died mid-run. Check the worktree if one exists. (Solo: also check the executor process — a stopped Solo agent with a saved session may be resumable.)
- **TODO** — run the drift check. If drifted: re-verify the finding still exists (it may have been fixed in passing), then refresh the "Current state" excerpts and `Planned at` SHA (solo: section-level `scratchpad_edit`, never a full overwrite). If the finding is gone, mark REJECTED ("fixed independently").

Finish with a short report: what's verified done, what was refreshed, what's rejected, and what's executable right now.

---

## `--issues` — publish plans as GitHub issues

Modifier on any planning invocation (`/improve-solo --issues`, `/improve-solo security --issues`). Additive distribution on top of the primary store — never a store of its own. The flag is the user's authorization to create issues — never create them without it.

1. Preflight: `gh auth status` succeeds and the repo has a GitHub remote. If either fails, write the plans to the primary store as normal and say why issues were skipped.
2. Show the list of titles about to become issues; confirm once if interactive.
3. Per plan: `gh issue create --title "<plan title>" --body-file <plan file>` (solo: export the scratchpad to a temp file first — see solo-backend.md). Labels: `improve` plus the category — apply only if the labels exist or can be created without erroring; skip labels rather than fail.
4. Record each issue URL in the plan's Status block and the index (solo: scratchpad Status block + todo comment).

The primary store remains the source of truth; the issue is distribution. The self-containment rule pays off here — the issue body needs no edits to make sense to whoever (or whatever) picks it up. Issues are public output: the playbook's untrusted content rule applies, so repo-embedded text appears only as minimal quoted evidence.
