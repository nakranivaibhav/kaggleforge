---
name: kaggle-experiment
description: Stage 4 — the experiment-graph loop (PROPOSE → DEVELOP → REVIEW → SCORE → DECIDE). Use when the comp has a frozen CV (folds.json) and a baseline champion, and the human says "run experiments", "/kaggle-experiment", "improve the model", "go auto", or after /kaggle-baseline submits. Drives one node per turn in interactive mode (Decision Card + gated submit) or hands the fan-out to the experiment-loop workflow in auto mode.
argument-hint: "[interactive|auto] [--node-id NNNN] [--draft|--improve|--debug|--combine]"
allowed-tools: Bash, Read, Write, Edit
---

# kaggle-experiment — the graph loop

You grow `comps/<slug>/graph.md` — a **DAG**, not a tree. Each **node = one atomic
change** attached to the deepest ancestor(s) whose work it keeps. You PROPOSE an
operator+parents, DEVELOP it (via the `kaggle-developer` subagent), REVIEW it (via
the `kaggle-reviewer` subagent — **you sequence developer→reviewer because
subagents can't nest**), SCORE its CV, DECIDE promotion. Read CLAUDE.md for the
standing contract; this skill is the procedure.

## 0 · Orient (every entry)
Resolve `<slug>` from `comps/` (or `$ARGUMENTS`). Then:
- `DATE=$(date -u +%Y-%m-%dT%H:%MZ)` — never type a date.
- Read `comps/<slug>/config.md` → autonomy mode. `auto_except_submit`/`full_auto`
  ⇒ **AUTO**; `interactive` ⇒ **INTERACTIVE**. CLI arg overrides.
- Read `comps/<slug>/spec.md` machine block: `metric, direction (minimize|maximize),
  target, target_cols, id, task_type, time_col?, group_key?`.
- Read `graph.md` to rebuild the frontier — the Mermaid DAG plus the `## nodes`
  table (statuses: proposed·running·buggy·valid·champion·dead) — and the
  `journal.md` tail. Confirm `folds.json` + `champion/` exist (else tell the human
  to run `/kaggle-validate` + `/kaggle-baseline` first).
- **Resume first:** if any node is `running`, open its `nodes/node_NNNN/node.md`
  and resume from its **`stage`** field (artifact-then-mark — verify the stage
  against its named artifact; e.g. `built` with no `cv` ⇒ resume at *score*). A
  `running` node whose `stage` is ahead of its artifacts, or with no artifacts at
  all, ⇒ mark `dead`, move on.

## 1 · Search policy — pick operator + parents
Count `valid`+`champion` root-branch families. `num_drafts` default **4**;
`debug-depth` ≤ **5** attempts. In order:
1. **draft** while valid-root-families < `num_drafts` → new branch under root, a
   *structurally different* approach (e.g. GBDT vs NN vs Darts). `parents: [root]`.
2. else **debug** the shallowest `buggy` node within depth → child of the buggy
   node. Regenerate the node from scratch after **3** failed attempts; prune to
   `dead` after **5**. `parents: [<buggy node>]`.
3. else **improve** the best `valid`/`champion` node with **exactly one** atomic
   change → child of that node. A/B against its parent; reject on CV regress.
   `parents: [<that node>]`.
4. else **combine** 2+ `valid`, **de-correlated** nodes when a blend/ensemble/stack
   of their OOF beats the best single — a DAG join, `parents: [<id>, <id>, …]`
   (this is also what `/kaggle-final` does).
- **Keep ≥2 families alive.** If the best lineage hasn't beaten CV by more than
  one parent-SEM over **5 consecutive improves**, force a **draft** of a
  different family — pivot architecture, don't keep tuning.
- Parent rule: attach to the *deepest ancestor(s) whose work the change keeps* —
  one for draft/improve/debug, the **list** of merged nodes for combine.

## 2 · Create the node
`NNNN` = next zero-padded id. `mkdir -p comps/<slug>/nodes/node_NNNN/src`.
Write `node.md` from CLAUDE.md's template (frontmatter = all data, **no
checkboxes**), filling the `## plan` with real detail:
**built on** (parent(s) + what stays byte-identical), **change** (2–4 lines, the
concrete HOW the developer will implement and the `solution.py` docstring will
expand), **hypothesis**, **target**. Set `op`, `parents` (a list — `[root]` for a
draft, the 2+ merged ids for combine), `family`, `created: $DATE`,
`status: running`, `stage: proposed`.
Then register the node in `graph.md`:
- add a Mermaid edge from each parent — e.g. `node_<parent> --> node_NNNN[node_NNNN · <desc> · …]`
  (one edge per parent for a combine);
- add a `## nodes` table row:
  `| node_NNNN | <one-line change> | — | — | running | \`nodes/node_NNNN/node.md\` |`.

## 3 · DEVELOP — delegate to `kaggle-developer`
Spawn the **kaggle-developer** subagent (fresh context). Hand it explicitly:
- spec path `comps/<slug>/spec.md`, folds path `comps/<slug>/folds.json`;
- **parent src path(s)** = `champion/src` for a draft off baseline, else
  `nodes/node_<parent>/src` (the list of parent dirs for a combine); target node
  `nodes/node_NNNN/`;
- the **one-line atomic change**, and the metric+direction.
It must: copy parent src → node `src/`, apply only that change, write
`src/solution.py` that (a) loops `folds.json`, (b) **fits every transform
inside the train fold only**, (c) writes the per-fold scores into `node.md`'s
frontmatter (`folds`, `cv`, `sem`), the OOF + `submission.csv`, and a
`features.txt` (one feature col per line) for the scan, and (d) runs the
**shuffled-label control** in its own CV harness (import `shuffled_label_ok` from
`tools/leakage_scan.py`). Long train → marker file:
```bash
DONE=/tmp/<slug>_node_NNNN.done ; rm -f "$DONE"
(uv run python comps/<slug>/nodes/node_NNNN/src/solution.py \
   > comps/<slug>/nodes/node_NNNN/train.log 2>&1 ; touch "$DONE") &
# wait on [ -f "$DONE" ]; tail filtered for: cv=|Traceback|Error|Killed|OOM
```
On a clean run (no traceback in `train.log`) the developer sets node.md
`stage: built` — then, with per-fold scores written, `stage: scored`. A traceback
⇒ set `status: buggy` in node.md + its `graph.md` row, loop back to §1 (debug).

## 4 · REVIEW — sequence the `kaggle-reviewer` subagent
After the developer returns, **you** spawn the **kaggle-reviewer** subagent on
`nodes/node_NNNN/`. It runs unit tests + the leakage suite and returns PASS/FAIL.
The structural scan it drives:
```bash
uv run tools/leakage_scan.py \
  --train comps/<slug>/data/train.csv --test comps/<slug>/data/test.csv \
  --target <target> --target-cols <target_cols> --id <id> \
  --features-file comps/<slug>/nodes/node_NNNN/src/features.txt \
  --source comps/<slug>/nodes/node_NNNN/src/solution.py \
  --out    comps/<slug>/nodes/node_NNNN/leakage_scan.json
```
Plus the in-harness **shuffled-label control** (CV must collapse to the random
baseline) and `cv_too_good` tripwire — surface a tripwire to the human before any
submission. The reviewer **writes the `gates:` booleans into node.md**
(`schema_ok, oof_full, no_nan, dist_sane, leak_clean, shuffle_collapsed,
cv_too_good, passed`) and sets `stage: reviewed`. **Any `error`-severity check ⇒
the CV is void**: set `gates.leak_clean: false` (or `shuffle_collapsed: false`),
`leak: VOID`, `status: buggy` (or `dead` if the leak is intrinsic to the change),
and return to §1. PASS + leak-clean ⇒ `gates.passed: true`, `leak: clean`.

## 5 · SCORE — compute CV (mean ± sem)
The per-fold scores already live in node.md frontmatter (`folds`); compute
`cv = mean` and `sem = std(ddof=1)/sqrt(k)` and ensure both fields are filled.
Set `stage: scored` (if not already) and `status: valid`, then update the node's
`graph.md` row (fill its `cv` cell) and its Mermaid label (`… · <cv>`).

## 6 · DECIDE — promote or keep
Compare against the current champion CV (from `champion/README` / the champion row
in `graph.md`). Let `k=2`. **Accept as new champion** iff **all** hold:
- CV beats champion **beyond k·sem** in the spec's `direction` (a within-noise
  win is *not* a promotion — leave `status: valid`, keep champion);
- leakage-clean (§4 passed, no void);
- **CV↔LB not diverging** — if this lineage has a submitted LB, the CV gain is
  directionally consistent with LB (a gap is *surfaced*, never an auto-demote;
  CV still decides what to submit).

On **accept**: byte-copy (cp, never symlink)
`nodes/node_NNNN/src` + `submission.csv` → `champion/`, update `champion/README`
(node id, cv±sem, $DATE, one-line change), set node.md `status: champion`, and
demote the prior champion node to `status: valid`. On **reject**: leave
`champion/` untouched; `status` stays `valid` (or `buggy`/`dead`). Either way set
`stage: decided`, `decided: $DATE`, and **update `graph.md`**: apply/move the
`:::champ` class on the Mermaid node, refresh the `## nodes` table `status`
column(s), and regenerate the header line (`metric · champion: node_NNNN (cv … ·
lb …) · updated $DATE`). Append one timestamped journal line:
```bash
echo "- $DATE  node_NNNN  <op>(parents=<ids>)  cv=<mean>±<sem>  leak=clean  -> <champion|valid|buggy|dead>: <one-line reason>" \
  >> comps/<slug>/journal.md
```

## 7 · SUBMIT (gated — only the best beats the last submitted CV)
Submit only a node whose CV beats the **last submitted CV** by more than
fold-noise (k·sem) — never spend a slot to A/B on the LB. Check budget:
`uv run tools/kaggle_io.py budget --ledger comps/<slug>/submissions.md` (used is
derived from today's UTC rows; resets 00:00 UTC). Validate before spending a slot:
```bash
uv run tools/validate_submission.py \
  --submission comps/<slug>/nodes/node_NNNN/submission.csv \
  --sample comps/<slug>/data/sample_submission.csv --id <id>
```
- **INTERACTIVE / `auto_except_submit`:** render a **Decision Card** (below) and
  **wait** — the human owns every real submission.
- **`full_auto` + budget remaining:**
  `uv run tools/kaggle_io.py submit <slug> --file …/submission.csv --message "node_NNNN cv=<mean>"`,
  append the row to `submissions.md` (`| $DATE | node_NNNN | <cv> | <lb-pending> |`),
  poll `uv run tools/kaggle_io.py submissions <slug>` for the public score, write
  it back. A 403 ⇒ rules-not-accepted/unverified (human gate), **not** bad creds.
Once the ledger row exists, set node.md `lb: <public score>`, `submitted: $DATE`,
`stage: submitted`, and update the node's `lb` cell in the `graph.md` table.

## Decision Card (render at every interactive gate)
```
📋 experiment · node_NNNN <op>(parents=<ids>)
What's going on:   <plain sentence of the change>
Found / propose:   • cv <mean>±<sem> vs champ <champ_cv> (<beats by Nσ | within noise>)
                   • leakage: clean · families alive: <n> · stall-count: <m>/5
                   • <submit this? slot N/5 today | keep exploring>
Why:               <one line>
Cost:              <~mins · cpu/gpu · submissions N/5 today>
Your call:         [Approve] [Change something] [Skip] [Tell me more]
Autonomy: <mode> — waiting
```

## Modes
- **INTERACTIVE** — exactly **one node per turn**: §1→§6, render the Card, stop.
  Resume next turn at §0. Every submission is human-gated (§7).
- **AUTO** — invoke the workflow for the best-first fan-out instead of looping by
  hand: `.claude/workflows/experiment-loop.js` (it sequences developer→reviewer
  per node across the frontier and applies §1's policy). It can't show cards: in
  `auto_except_submit` it **queues** the best node and *this* main session
  surfaces the Decision Card before submitting; in `full_auto` it submits within
  budget. When the workflow returns, re-orient (§0) and continue.

## Invariants
- Artifact-then-mark — a node's `stage` never runs ahead of its named artifact.
- One atomic change per node; every CV delta is attributable.
- Leakage voids the score — a leaky node (`leak: VOID`) never counts, never promotes.
- Trust CV over LB; a CV↔LB gap is a diagnostic to surface, not an auto-demote.
- All dates from `date -u`; all scripts via `uv run`; reusable code stays in `tools/`.
