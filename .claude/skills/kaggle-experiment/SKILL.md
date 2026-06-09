---
name: kaggle-experiment
description: Stage 4 — the experiment loop: refine N proposals (proposer↔critic via propose-loop), register them, build-and-gate EVERY proposal (kaggle-developer builds leak-free AND self-gates), decide promotion. Use when there's a frozen CV + baseline champion and the human says "run experiments" / "/kaggle-experiment" / "improve the model" / "go auto".
argument-hint: "[interactive|auto] [--n-proposals N]"
allowed-tools: Bash, Read, Write, Edit, Agent, Workflow, Skill
---

# kaggle-experiment — propose → build all → gate → decide

You are the **orchestrator**. Each round you get a set of proposals, build **every**
one of them, and promote the best. There is no best-first branching and no pruning
here — the **proposer** decides what to try; **you build all of it**. Three workers
do the work:

| worker | role |
|---|---|
| **kaggle-proposer** | proposes N experiments, revises them, and (once confirmed) writes the node records |
| **kaggle-proposal-reviewer** | critiques the proposals before any code is written |
| **kaggle-developer** | builds one node AND self-gates it — fold-correct + performant CV, leakage + unit gate, gate booleans written, a valid submission (a leak VOIDs the CV) |

Read `CLAUDE.md` for the standing contract; this skill is the procedure. Subagents
can't nest, so **you** (the main session) sequence proposer → developer.

## 0 · Orient (every entry)
- `<slug>` from `comps/` (or the arg). `DATE=$(date -u +%Y-%m-%dT%H:%MZ)` — never type a date.
- Read `config.md` → mode. `auto_except_submit`/`full_auto` ⇒ **AUTO**; `interactive` ⇒ **MANUAL**.
- Read `spec.md` machine block (`metric, direction, target, target_cols, id, task_type, …`), `graph.md` (the champion + node table), `data.md` (the engineered feature-sets), and the `journal.md` tail. Confirm `folds.json` + `champion/` exist (else run `/kaggle-validate` + `/kaggle-baseline` first).
- **Resume:** if a node is `running`, open its `node.md` and resume from its `stage` (e.g. `built` with no `cv` ⇒ resume at §5 score). A `running` node with no artifacts ⇒ mark `dead`, move on.

## 1 · PROPOSE — refine the round's proposals (`experiment_plan` gate)
Run the **propose-loop** workflow — it spawns kaggle-proposer (draft **3**
proposals; set `nProposals` to change) ↔ kaggle-proposal-reviewer (critique),
looping up to 2 rounds until the critic is happy:
```
Workflow propose-loop   args: { slug: <slug>, nProposals: 3, maxIters: 2 }
→ returns the refined proposals.
```
- **AUTO:** take the refined proposals straight to §2.
- **MANUAL:** render the **Proposal Card** (below) and **wait**. You are the
  director — the human accepts some, discards some, and gives a new direction for
  what to explore instead. On a redirect, spawn **kaggle-proposer** (REVISE) with
  the human's direction and re-card. On approval, go to §2 with the accepted set.

## 2 · REGISTER — write the confirmed nodes
Spawn **kaggle-proposer** (REGISTER) with the confirmed proposals. It reserves each
node id, writes `nodes/node_NNNN/node.md` (status `proposed`, the `## plan`, the
`uses_data` field), adds each to `graph.md`, and updates `data.md` (new/reused
feature-sets). You never hand-write node.md — the proposer owns it. It's one
sequential call, so the parallel builders in §3 never collide on `graph.md`/`data.md`.

## 3 · BUILD-AND-GATE ALL — hand every node to kaggle-developer
Build **every** registered node: spawn the developers **in parallel** when the nodes
are independent (one `Agent` call each, in one message), or **sequentially** if
compute/GPU is tight (esp. GPU nodes — serialize them; one 32 GB card can't run two
big-model nodes at once). Hand each developer: `spec.md`, `folds.json`, its
`parent_src`, its node dir, the one-line change, metric+direction, and the
**baseline + parent per-fold scores** (for the cv-too-good + per-fold-delta gates).
The developer writes a fold-correct, **performant** `solution.py` (it times one unit
before the full run — never an unprofiled multi-hour job), the per-fold CV into
`node.md`, the OOF + `submission.csv` + `features.txt`, **then self-gates**: runs the
unit tests + leakage scan, writes the `gates:` booleans + `leak`, sets `status:
valid|buggy|dead` and `stage: reviewed`. A traceback ⇒ `status: buggy` (propose a
`debug` node next round); any error-severity leak ⇒ `leak: VOID` (CV does **not**
count). One worker builds and proves — there is no separate review step.

> If a developer agent ever **re-launches a run you killed** or exits before its
> backgrounded train finishes, take the node over directly (the orchestrator owns
> the marker file): kill stray processes, attach your own waiter, and on completion
> write the CV + run the gate yourself. Don't re-message a zombie agent.

## 5 · SCORE — confirm the CV
The per-fold scores + gate booleans are already in `node.md` (the developer wrote
them in §3). Confirm `cv = mean`, `sem = std(ddof=1)/sqrt(k)`, and `gates.passed` are
filled, that `status: valid` (for a passing node), and fill the node's `cv` cell +
Mermaid label in `graph.md`. A `buggy`/`VOID` node's CV does not count.

## 6 · DECIDE — promote or keep
For each valid node, compare to the champion (from `champion/README` / `graph.md`).
**Promote** iff its CV beats the champion **beyond 2·sem** in the spec's direction
AND it's leak-clean AND (if the lineage has a submitted LB) the CV gain is
LB-consistent. On promote: byte-copy (cp, never symlink) `src/` + `submission.csv` →
`champion/`, update `champion/README`. Then move the champion crown in ONE pass — set
the new node AND demote the old one across all three places each (per CLAUDE.md's
graph rule): frontmatter `status` (`champion` ↔ `valid (prev champ)`), Mermaid `:::champ`
(add ↔ remove), table status cell, and the header `champion:` line. On reject: leave
`champion/` untouched. Either way set `stage: decided`, `decided: $DATE` on the decided
node, and append one `journal.md` line per node. **Before leaving §6 verify the
invariant: exactly one node reads `champion` in frontmatter, Mermaid, table, and header
— the same node.**

## 7 · SUBMIT (gated)
Submit only a node whose CV beats the **last submitted CV** by more than fold-noise
(2·sem) — never spend a slot to A/B on the LB. Validate the file and check budget
first (`uv run tools/kaggle_io.py budget --ledger comps/<slug>/submissions.md`).
- **MANUAL / `auto_except_submit`:** render the SUBMIT Decision Card and **wait** — the human owns every real submission. Run `/kaggle-submit <slug> node_NNNN`.
- **`full_auto` + budget:** `/kaggle-submit <slug> node_NNNN`, append the ledger row, poll the public score.

## Proposal Card (manual `experiment_plan` gate)
```
📋 experiment plan · <n> proposals for <slug>
What's going on:   <one plain sentence on where the search stands>
Proposals:         1. <op> <desc> — <why> (vs <parent> cv <x>)
                   2. …
                   3. …
Critic's take:     <one line from the proposal-reviewer>
Cost:              <~mins · cpu/gpu each · ALL will be built>
Your call:         [Approve all] [Accept some / discard some] [Redirect: try X instead] [Tell me more]
Autonomy: <mode> — waiting
```

## Modes
- **MANUAL (interactive)** — §1 refine, render the Proposal Card, **wait**. On
  approval, §2 register and §3–§6 build/gate/decide the accepted node(s), then stop.
  Every submission is human-gated (§7).
- **AUTO (`auto_except_submit` / `full_auto`)** — §1→§6 with no pause: refine,
  register, build EVERY proposal, gate, decide. Only §7 submit stops (queue + ask in
  `auto_except_submit`; spend a slot in `full_auto`). Re-enter for the next round.

## Invariants
- Build EVERY confirmed proposal — the proposer prunes, the orchestrator doesn't.
- One atomic change per node; every CV delta is attributable.
- Leakage voids the score; a leaky node never promotes.
- Trust CV over the LB; a CV↔LB gap is a diagnostic to surface, not an auto-demote.
- Artifact-then-mark; all dates from `date -u`; all scripts via `uv run`.
