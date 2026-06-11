> **Real artifacts.** A plan scratchpad + todo pair produced by `/improve-solo`
> against this very repo on 2026-06-11 (Plan 005, shipped in v1.1.0). The
> repo has moved on — don't execute this plan; it is here as an illustration of
> the Solo artifact model. For the files-store format, see
> [001-extract-shadow-config-resolution.md](001-extract-shadow-config-resolution.md).

# Solo Artifact Model — Plan 005 Example

Each `/improve-solo` plan produces two linked Solo artifacts: a **scratchpad**
holding the full plan document, and a **todo** holding status, priority, tags, and
blockers — the todo list is the index, and the blocker graph is the dependency
chain. The executor reports progress via a todo comment; the reviewer owns status
and completion. The comment trail below is excerpted — routine bookkeeping
comments (intermediate revise rounds, reconcile notes) are omitted.

---

## The plan scratchpad

*Scratchpad 23, project improve-solo. Body reproduced verbatim; the one
absolute path it contained has been redacted to `<repo>`.*

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in "STOP conditions" occurs, stop and report — do not
> improvise. Report via a comment on the paired todo; never change the todo's
> status, tags, or completion — the reviewer owns those.
>
> **Drift check (run first)**: `git diff --stat 68fef88..HEAD -- skills/improve-solo/ README.md .claude-plugin/plugin.json`
> Changes from Plans 002 and 003 (both blockers of this plan) are EXPECTED in
> `SKILL.md`, `solo-backend.md`, `plan-template.md`, `audit-playbook.md`, and
> `closing-the-loop.md` — this plan's anchors below already assume 002/003
> have landed. Changes from Plan 004 in `README.md`/`plugin.json` may also be
> present (new Installation section, new repository/homepage fields) and are
> fine. Any OTHER change to the quoted anchor text is a STOP condition —
> verify each step's anchor grep before editing.

## Status
- **Priority**: P1
- **Effort**: M
- **Risk**: MED
- **Depends on**: Plan 002 (todo 385) and Plan 003 (todo 386) — both edit the same SKILL.md sections this plan rewrites; their replacement text is the anchor here. Plan 004 (todo 387) is NOT a blocker but touches README/plugin.json — rebase over it if it lands first.
- **Category**: direction
- **Planned at**: commit `68fef88`, 2026-06-11
- **Todo**: 388
## Why this matters

Owner decision (2026-06-11): the fork's identity is the Solo backend, but it ships opt-in behind `--solo` while the default mimics upstream — backwards for a project named improve-solo whose primary audience runs Solo. After this plan: the store is **auto-detected** (Solo when the Solo MCP tools are available, files otherwise), `--files` forces file-based plans, and `--solo` remains as an explicit override for back-compat. The one-backlog rule still outranks everything — an existing backlog is never forked by a default.

## Current state

All paths relative to `<repo>`. **These excerpts describe the expected post-002/003 state** (commit `68fef88` + Plans 002/003 as specified in scratchpads 20/21):

- `skills/improve-solo/SKILL.md`:
  - Frontmatter `description:` contains: `Plans persist to local files by default, or natively into Solo (scratchpads + todos, Solo-agent executors) with --solo.`
  - `## Storage backends` opens with `Exactly **one primary store** per run holds the plans, the index, and the rejected-findings memory:` followed by three bullets — `**files** (default)`, `**\`--solo\`**`, `**\`--issues\`**` — then the `**Store-conflict rule (one backlog, never two).**` paragraph.
  - Phase 1 bullet (post-002 Step 5 text): `Under \`--solo\`, first run the Phase 0 project-scoping steps in [references/solo-backend.md](references/solo-backend.md) (verify the selected Solo project matches this repo before anything is written — the Solo side of the next check is project-scoped). Then run the store-conflict check above.`
  - Phase 3 contains: `Under \`--solo\`, persist the run's audit record now:`
  - Phase 4 contains: `Under **\`--solo\`**, each plan is a scratchpad`
  - Invocation variants list contains: `- \`--solo\` (modifier on any invocation) → use Solo as the primary store and dispatcher, per [references/solo-backend.md](references/solo-backend.md).`
  - Hard Rule 1 (post-002 Step 1 text) contains: `Under the default **files** store, the ONLY files you may create or modify live under \`plans/\`` and `Under **\`--solo\`**, the only writes are Solo artifacts — … never repo files (one exception: the user may explicitly request an ad-hoc export of a plan to a file — see solo-backend.md).`
- `README.md`: intro paragraph (`By default it behaves exactly like upstream (plans as files). With \`--solo\` it stores and dispatches natively…`), the ASCII store diagram (lines ~7–12), the `## Storage backends` table (`| | files (default) | \`--solo\` |`), the `## Usage` block (`/improve-solo --solo` row, composing example `deep security --solo --issues`), and `## How \`--solo\` mode works`.
- `references/solo-backend.md`: title `# Solo Backend — storage and dispatch under \`--solo\`\``; intro `This file defines how the advisor workflow persists and dispatches when Solo is the primary store.`
- `references/plan-template.md`: two `**Under \`--solo\`**` occurrences (template notes, line ~13; index-file section, line ~159).
- `references/closing-the-loop.md`: backend note (line ~7) `…apply identically under \`--solo\`.`
- `.claude-plugin/plugin.json` `description`: `…to plans/ files by default, or natively into Solo (scratchpads + todos, Solo-agent executors) with --solo. It never edits your code.`

Conventions: dense bold-lead instruction prose; flags written as `` `--flag` ``; the README mirrors SKILL.md's structure at summary level — both must tell the same story after this plan.

## The new semantics (normative — write THIS into the docs)

Store resolution, in order; the first rule that applies wins:

1. **Explicit flag**: `--files` → files store; `--solo` → Solo store (kept for explicitness/back-compat). If `--solo` is given but Solo MCP is unavailable, say so and use files (unchanged behavior).
2. **Existing backlog**: if prior plans exist in exactly one store, follow that store regardless of flags or default; an explicit flag pointing elsewhere wins only if the user repeats it after the warning (the existing store-conflict rule, unchanged).
3. **Auto-detect (the new default)**: Solo if the Solo MCP tools are available and Phase 0 project scoping succeeds; otherwise files. Announce the selected store and why in one line at the start of the run.

## Commands you will need

| Purpose | Command (from repo root) | Expected on success |
|---------|--------------------------|---------------------|
| Anchor check | `grep -n "<anchor>" <file>` | exactly one match |
| Residue sweep | `grep -rn -- "--solo" skills/ README.md .claude-plugin/` | only intentional mentions (see done criteria) |
| New-flag sweep | `grep -rn -- "--files" skills/ README.md .claude-plugin/` | every doc that mentions `--solo` flag semantics |
| JSON validity | `python3 -m json.tool .claude-plugin/plugin.json` | exit 0 |
| Scope check | `git status --porcelain` | only in-scope files |

## Scope

**In scope** (the only files you may modify):
- `skills/improve-solo/SKILL.md`
- `README.md`
- `skills/improve-solo/references/solo-backend.md`
- `skills/improve-solo/references/plan-template.md`
- `skills/improve-solo/references/closing-the-loop.md`
- `.claude-plugin/plugin.json` (the `description` string only)

**Out of scope** (do NOT touch):
- `references/audit-playbook.md`, `examples/`, `LICENSE.md` — no backend-selection language in them.
- The GitHub repo description (`gh repo edit`) — owner/maintenance step, noted below.
- Renaming the skill, the slash command, or any tag names (`improve-plan` etc.) — identity stays.

## Git workflow

- Branch: `advisor/005-solo-default-store` off `main` (after 002/003 are merged).
- Commits per logical unit, e.g. `feat: auto-detect Solo as the default store`, `docs: README + manifest for solo-default`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Rewrite SKILL.md `## Storage backends`

Replace the three store bullets with (keep the section's opening sentence and the store-conflict paragraph, which Step 1 does not touch except as noted):

- `**solo** (default when available) — auto-detected: used when the Solo MCP tools are present and Phase 0 project scoping (see [references/solo-backend.md](references/solo-backend.md)) succeeds. Plans become scratchpad + todo pairs, the todo list IS the index, executors are Solo agents, and review wake-ups use Solo idle timers. **Read solo-backend.md before the first Solo-mode write.** \`--solo\` forces this store explicitly; if Solo MCP is unavailable, say so and use files.`
- `**\`--files\`** — plans as local files, exactly like upstream \`improve\`: \`plans/NNN-slug.md\`, index in \`plans/README.md\`, executors dispatched as host subagents with worktree isolation. Also the automatic fallback when Solo MCP is absent.`
- (keep the `--issues` bullet verbatim)

Then append one sentence to the section, before the store-conflict paragraph: `Resolution order: explicit flag → existing backlog (rule below) → auto-detect (Solo if available, else files). Announce the selected store and why in one line at the start of the run.`

**Verify**: `grep -c "Resolution order: explicit flag" skills/improve-solo/SKILL.md` → `1`; `grep -c "files** (default)" skills/improve-solo/SKILL.md` → `0`.

### Step 2: Sweep SKILL.md's remaining `--solo` conditionals

Recast each as store-conditional rather than flag-conditional:

- Frontmatter description: replace the persistence sentence with `Plans persist natively into Solo (scratchpads + todos, Solo-agent executors) when Solo is available — auto-detected — or to local files with --files.`
- Hard Rule 1: `Under the default **files** store` → `Under the **files** store`; `Under **\`--solo\`**, the only writes` → `Under the **Solo** store, the only writes` (keep the post-002 exception clause verbatim).
- Phase 1 bullet: `Under \`--solo\`, first run the Phase 0` → `When Solo MCP is available, first run the Phase 0` (rest of the post-002 sentence unchanged — Phase 0 now doubles as the auto-detect probe).
- Phase 3: `Under \`--solo\`, persist` → `Under the Solo store, persist`.
- Phase 4: `Under **\`--solo\`**, each plan` → `Under the **Solo** store, each plan`.
- Invocation variants: replace the `--solo` bullet with two bullets:
  `- \`--files\` (modifier on any invocation) → force the files store (plans as local files), even when Solo is available.`
  `- \`--solo\` (modifier on any invocation) → force the Solo store explicitly; normally unnecessary since Solo is auto-detected — per [references/solo-backend.md](references/solo-backend.md).`

**Verify**: `grep -cn -- '--files' skills/improve-solo/SKILL.md` ≥ `3`; `grep -n -- 'Under \`--solo\`' skills/improve-solo/SKILL.md` → no matches.

### Step 3: README.md — tell the same story

- Intro paragraph: rewrite the second sentence to: `When Solo is available it stores and dispatches natively through [Solo](https://soloterm.com) — auto-detected, no flag needed: plans become scratchpad + todo pairs, dependencies become real todo blockers, executors are Solo agents, and review wake-ups use Solo idle timers. With \`--files\` it behaves exactly like upstream (plans as files).`
- ASCII diagram: swap the rows — `solo (auto-detected)` first, `--files` second, `--issues` unchanged.
- Backends table header: `| | solo (default when available) | \`--files\` |` — swap the two content columns to match; update the lead-in sentence (`Exactly one primary store per run; \`--issues\` is additive…` stays, add `Solo is auto-detected; \`--files\` opts out.`).
- Usage block: replace the `/improve-solo --solo` row with `/improve-solo --files                plans as local files (upstream behavior)`; bare `/improve-solo` row description becomes `full audit → prioritized findings → plans (Solo when available, else files)`; composing example `deep security --solo --issues` → `deep security --files --issues`.
- `## How \`--solo\` mode works` → `## How the Solo store works` (mention auto-detection + `--solo` as explicit override in the first bullet, which already covers Phase 0).
- One-backlog line: unchanged.

**Verify**: `grep -c -- '--files' README.md` ≥ `4`; `grep -c "How the Solo store works" README.md` → `1`.

### Step 4: Reference docs — retitle the conditionals

- `solo-backend.md` title → `# Solo Backend — storage and dispatch when Solo is the store`; intro sentence: `…persists and dispatches when Solo is the primary store (the default whenever the Solo MCP tools are available; \`--files\` opts out — see SKILL.md for resolution order).`
- `plan-template.md`: both `**Under \`--solo\`**` → `**Under the Solo store**`.
- `closing-the-loop.md` backend note: `apply identically under \`--solo\`` → `apply identically under the Solo store`.

**Verify**: `grep -rn -- 'Under \`--solo\`' skills/improve-solo/references/` → no matches; `grep -c "when Solo is the store" skills/improve-solo/references/solo-backend.md` → `1`.

### Step 5: plugin.json description

Replace the `description` value with: `Fork of shadcn/improve: audits a codebase and writes plans other agents can execute — natively into Solo (scratchpads + todos, Solo-agent executors) when Solo is available, or to plans/ files with --files. It never edits your code.`

**Verify**: `python3 -m json.tool .claude-plugin/plugin.json` → exit 0; `grep -c -- '--files' .claude-plugin/plugin.json` → `1`.

### Step 6: Coherence read

Read SKILL.md and README.md end-to-end once. Every remaining `--solo` occurrence must be one of: the explicit-override invocation bullet, the README usage note of the override, or historical/upstream references. List every remaining occurrence in your report NOTES with one-word justification each.

**Verify**: `grep -rn -- '--solo' skills/ README.md .claude-plugin/ | grep -v "improve-solo"` → every hit is on your justified list.

## Test plan

No automated harness (known gap — audit finding 7). Gates: the per-step greps, the Step 6 residue sweep with per-hit justification, and the reviewer's own end-to-end read. The real test is the next `/improve-solo` run with no flags on a Solo-connected project: it must announce Solo as the auto-selected store.

## Done criteria

ALL must hold, from the repo root:

- [ ] All step-verification greps pass as stated
- [ ] `grep -rn -- 'Under \`--solo\`' skills/` → 0 matches
- [ ] Every surviving `--solo` literal is on the justified list in the report (Step 6)
- [ ] `python3 -m json.tool .claude-plugin/plugin.json` exits 0
- [ ] `git status --porcelain` shows only the six in-scope files
- [ ] SKILL.md and README.md state the same resolution order (flag → existing backlog → auto-detect)

## STOP conditions

Stop and report back (do not improvise) if:

- Plans 002 or 003 have not actually landed (their post-state anchors quoted above don't match) — this plan's edits assume theirs.
- Any anchor grep matches zero or multiple times (drift or ambiguity).
- You conclude `--solo` should be removed entirely rather than kept as an override — that's a semantics change the owner hasn't made; keep it and note the argument.
- The store-conflict rule's wording stops making sense under the new default in a way these steps don't cover — report rather than rewording beyond the spec.

## Maintenance notes

- After merge, the owner should: re-sync or plugin-reinstall the running skill copy, and refresh the GitHub repo description (`gh repo edit codeSTACKr/improve-solo --description "<new plugin.json description>"`) — it was set from the old wording on 2026-06-11.
- Plan 004 touches README (Installation section) and plugin.json (repository/homepage): whichever of 004/005 lands second rebases; the edits are line-disjoint by design.
- Reviewer focus: the auto-detect rule must not weaken the one-backlog rule — rule 2 outranks rule 3 in every retelling (SKILL.md, README). Also confirm the Phase 1 bullet still reads correctly post-002+005 composition.
- Bump consideration: this changes default behavior — version `1.1.0` in BOTH plugin.json and SKILL.md frontmatter at release (the known dual-declaration papercut).

---

## The paired todo

**Todo 388 — Plan 005: Make Solo the auto-detected default store; add --files for file-based plans**

| Field | Value |
|---|---|
| Priority | high |
| Status | completed |
| Tags | direction, improve, improve-plan |
| Blockers | todo 385 (Plan 002), todo 386 (Plan 003) — both rewrote the same SKILL.md sections; this plan anchored on their post-state |
| Completed | 2026-06-11 |

**Body:** Owner-directed (2026-06-11): flip the default — Solo auto-detected as the primary store when Solo MCP is available, `--files` to force upstream file behavior, `--solo` kept as explicit override. Resolution order: explicit flag → existing backlog (one-backlog rule unchanged) → auto-detect. Touches SKILL.md, README, three reference docs, plugin.json description. Blocked by Plans 002/003 (same SKILL.md sections; this plan anchors on their post-state).

---

## The comment trail

*Two comments excerpted: the executor's final STATUS report and the reviewer's APPROVE verdict. Intermediate REVISE/revision rounds and a post-merge reconcile note are omitted.*

---

**Comment by executor-plan-005** — executor's STATUS/STEPS report after completing all six plan steps:

> STATUS: COMPLETE
>
> STEPS:
> - **Drift check** (pre-step): done — `git diff --stat 68fef88..HEAD -- skills/improve-solo/ README.md .claude-plugin/plugin.json` confirmed Plans 002/003/004 all landed (SKILL.md, solo-backend.md, plan-template.md, closing-the-loop.md, audit-playbook.md, README.md, plugin.json changes — all expected). Anchors verified before each edit.
>
> - **Step 1** (Storage backends bullets): done — replaced the three store bullets with the new `solo (default when available)`, `--files`, and `--issues` bullets; appended Resolution order sentence before store-conflict paragraph. Verification: `grep -c "Resolution order: explicit flag" SKILL.md` → `1`; `grep -c "files** (default)" SKILL.md` → `0`. ✓
>
> - **Step 2** (SKILL.md --solo conditionals sweep): done — frontmatter description updated; Hard Rule 1 "default **files** store" → "**files** store", "Under **\`--solo\`**, the only writes" → "Under the **Solo** store, the only writes"; Phase 1 bullet "Under \`--solo\`, first run" → "When Solo MCP is available, first run"; Phase 3 "Under \`--solo\`, persist" → "Under the Solo store, persist"; Phase 4 "Under **\`--solo\`**, each plan" → "Under the **Solo** store, each plan"; Invocation variants: replaced the `--solo` bullet with two bullets (`--files` + `--solo` as explicit override). Verification: `grep -cn -- '--files' SKILL.md` → `3`; `grep -n -- 'Under \`--solo\`' SKILL.md` → no matches. ✓
>
> - **Step 3** (README.md): done — intro paragraph rewritten; ASCII diagram rows swapped (solo auto-detected first, --files second); backends table header swapped to `solo (default when available)` / `--files` with columns swapped; lead-in sentence updated with "Solo is auto-detected; `--files` opts out."; Usage block updated (`--files` row added, `/improve-solo` description updated, composing example changed to `--files --issues`); `## How \`--solo\` mode works` → `## How the Solo store works` with auto-detection note in Phase 0 bullet; Hard rules `--solo` reference updated to `--files` mode / Solo store. Verification: `grep -c -- '--files' README.md` → `7` (≥ 4); `grep -c "How the Solo store works" README.md` → `1`. ✓
>
> - **Step 4** (Reference docs): done — `solo-backend.md` title → `# Solo Backend — storage and dispatch when Solo is the store`; intro sentence updated with "(the default whenever the Solo MCP tools are available; `--files` opts out — see SKILL.md for resolution order)"; `plan-template.md` both `**Under \`--solo\`**` → `**Under the Solo store**`; `closing-the-loop.md` "apply identically under \`--solo\`" → "apply identically under the Solo store". Verification: `grep -rn -- 'Under \`--solo\`' references/` → no matches; `grep -c "when Solo is the store" solo-backend.md` → `1`. ✓
>
> - **Step 5** (plugin.json description): done — replaced with "…natively into Solo (scratchpads + todos, Solo-agent executors) when Solo is available, or to plans/ files with --files. It never edits your code." Verification: `python3 -m json.tool plugin.json` → exit 0; `grep -c -- '--files' plugin.json` → `1`. ✓
>
> - **Step 6** (Coherence read + residue sweep): done — `grep -rn -- '--solo' skills/ README.md .claude-plugin/ | grep -v "improve-solo"` (from repo root, relative paths) returned exactly 2 hits, both justified:
>   1. `README.md:67` — **override-note**: "Solo is auto-detected when the Solo MCP tools are available; `--solo` forces it explicitly when needed." This is the README usage note of the explicit override.
>   2. `.claude-plugin/marketplace.json:8` — **out-of-scope**: `marketplace.json` is not in the edit scope per the plan; still reads "plans as files by default, or native Solo scratchpads + todos with --solo." (old default). Cannot edit per scope constraints; flagged as maintenance item below.
>
> FILES CHANGED:
> - `skills/improve-solo/SKILL.md`
> - `README.md`
> - `skills/improve-solo/references/solo-backend.md`
> - `skills/improve-solo/references/plan-template.md`
> - `skills/improve-solo/references/closing-the-loop.md`
> - `.claude-plugin/plugin.json`
>
> Commit: `9c43df9` on branch `advisor/005-solo-default-store`.
>
> NOTES:
> - **marketplace.json residue**: `.claude-plugin/marketplace.json` has a `--solo` reference (`plans as files by default, or native Solo scratchpads + todos with --solo.`) that is now stale — it still implies files is the default. This file is out of scope per the plan. Reviewer should decide whether to include marketplace.json in the edit scope (it would be a one-line description update) or handle it as a post-merge maintenance step alongside the GitHub repo description update mentioned in the plan's maintenance notes.
> - **Version bump**: The plan notes that 1.1.0 should be bumped in both plugin.json and SKILL.md frontmatter at release. This was not done per the plan scope (version bumping is a release step, not in the spec).
> - **Scope clean**: `git status --porcelain` confirmed exactly 6 in-scope files, no extras.
> - **Resolution order consistency**: Both SKILL.md and README.md state flag → existing backlog → auto-detect, and rule 2 (existing backlog) outranks rule 3 (auto-detect) in both documents.
> - **Phase 1 note for reviewer**: Post-002+005 composition, Phase 1 bullet reads "When Solo MCP is available, first run the Phase 0 project-scoping steps… Then run the store-conflict check above." This doubles as the auto-detect probe as intended by the plan.
> - **STOP conditions**: none triggered. Plans 002/003 anchors matched. All anchor greps returned exactly one match. No semantic issues with the store-conflict rule under the new default — rule 2 still outranks rule 3 in both docs.

---

**Comment by Claude (reviewer)** — final APPROVE verdict after one revision round:

> REVIEW VERDICT: APPROVE (after round-1 revision)
>
> Re-verified in the worktree: marketplace.json valid JSON with the corrected description; residue sweep now exactly one surviving non-"improve-solo" `--solo` hit (README.md:67, the authorized explicit-override note); scope = the 6 original files + the reviewer-authorized marketplace.json, nothing else; revision commit `ff3223f` is exactly the one-line change authorized. Full diff read across both commits (`9c43df9` + `ff3223f`) — the Solo-default semantics are consistent across SKILL.md, README, all three reference docs, and both manifests, with the resolution order (explicit flag → existing backlog → auto-detect) stated identically everywhere.
>
> Result: merging to main and pushing per the user's delegation. Post-merge follow-ups for the owner, recorded here: (1) re-sync ~/.claude/skills/improve-solo or switch to plugin install — the running copy predates Plans 004+005; (2) decide on the 1.1.0 version bump (default behavior changed; bump plugin.json + SKILL.md frontmatter together); (3) GitHub repo description refreshed by the advisor to match the new plugin.json wording.
