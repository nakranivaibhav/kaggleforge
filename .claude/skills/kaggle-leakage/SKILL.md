---
name: kaggle-leakage
description: Reference — leakage & validation discipline for kaggleforge. Preloaded into the kaggle-reviewer subagent (and any node that computes CV). Use when validating a node's CV, deciding whether a score counts, running tools/leakage_scan.py, or judging a CV↔LB gap.
allowed-tools: Bash, Read
---

# Leakage & validation discipline (reference)

A node's CV **does not count until the leakage suite passes**. Leakage voids a
score regardless of how good the CV looks (CLAUDE.md hard rule 3). This skill is
the standing checklist the `kaggle-reviewer` applies to every node — including
data-cleaning and feature-engineering nodes.

The suite is **static + structural** → `tools/leakage_scan.py` (one command, JSON
report, exit code is the gate).

---

## The surviving suite (what each check is)

From `tools/leakage_scan.py` (run `uv run tools/leakage_scan.py --selftest` to
confirm the tool itself is sound before trusting a report):

| check | surface | severity | catches |
|---|---|---|---|
| `no_global_fit_in_source` | static scan of `solution.py` | **warn** | a transform `.fit(` on full / `concat([train,test])` data → verify fit-inside-fold |
| `target_not_in_features` | structural | **error** | target (or any `--target-cols`) used as a feature |
| `id_not_in_features` | structural | **error** | id column / row-order used as a feature |
| `feature_target_correlation` | structural | **error** if a hit | a feature with ≥0.999 \|corr\| vs target (target-leak smell) |
| `train_test_duplicates` | structural | **warn** | train rows that duplicate a test row on features |
| `cv_too_good_tripwire` | in-node (`cv_too_good`) | **warn** | implausible CV jump over baseline — human eyeballs before a slot is spent |

Group / temporal correctness is **enforced upstream** by `tools/make_folds.py`
(TimeSeriesSplit = past→future expanding window; GroupKFold = a group never
straddles folds): the frozen `folds.json` is what enforces that a group never
straddles folds and that each fold trains only on past rows. Never refit folds
across the run — freeze once in `/kaggle-validate` and read `folds.json`.

---

## Static + structural scan — the invocation

```bash
uv run tools/leakage_scan.py \
  --train comps/<slug>/data/train.csv \
  --test  comps/<slug>/data/test.csv \
  --target <TARGET_COL> \
  --id     <ID_COL> \
  --features-file comps/<slug>/nodes/node_NNNN/src/features.txt \
  --source        comps/<slug>/nodes/node_NNNN/src/solution.py \
  --out           comps/<slug>/nodes/node_NNNN/leakage_scan.json
```

- `--features-file` is a **newline-separated list of the exact feature column
  names** the node feeds the model. The node writes this (one feature per line)
  before review; the scan reads it to check target/id aren't among them and that
  no feature copies the target. If the node has no test split available, omit
  `--test` (the duplicate check is then skipped as a warn).
- `--target-cols a,b,c` (comma-separated) lists any *additional* columns that are
  deterministic functions of the target (e.g. a leaked aggregate) — they're
  treated as target columns by `target_not_in_features`.
- `--source` static-scans that file for a global `.fit(`; pass the node's real
  `solution.py`.

### Reading the exit code (the gate)

```bash
uv run tools/leakage_scan.py ... ; echo "exit=$?"
```

- **exit 0** → no `error`-severity check failed. `warn`s may still be present
  (printed as `[warn]`, written to the JSON) — surface them in the node's
  `gate_note` (and `cv_too_good` in `gates:`), but they do **not** void the CV.
  Resolve a `no_global_fit_in_source` warn by reading the flagged line and
  confirming the fit is inside-fold.
- **exit 1** → at least one **`error`**-severity check failed → **VOID this
  node's CV.** The node is *buggy*, not *valid*, no matter what its metric says.
  In `node.md` set `gates.leak_clean: false`, `leak: VOID`, and
  `status: buggy`; the fix is a **debug** child, not a re-score.

The reviewer's PASS/FAIL is exactly this exit code — it must be 0 for the CV to
count.

---

## CV-too-good tripwire (warn, before spending a slot)

Before the node's CV is allowed to claim champion / a submission slot, sanity it:

```python
from tools.leakage_scan import cv_too_good
chk = cv_too_good(node_cv, baseline_cv, DIRECTION)   # default max_rel_gain=0.9
# chk.passed False → an implausible jump over baseline; do not auto-submit —
# surface in the Decision Card for human eyes (it is a warn, not a void).
```

---

## The rules (do not negotiate these)

1. **Leakage voids a score, full stop.** An `error` from `leakage_scan.py`
   (exit 1) marks the node `buggy` and its CV does not count — regardless of how
   good the CV is. Fix via a **debug** child.
2. **A CV↔LB gap is a DIAGNOSTIC ONLY — never an auto-demote.** Trust a
   well-built local CV over the small, noisy public LB (CLAUDE.md rule 6). Log
   the gap in the journal, surface it in a Decision Card, and only investigate
   (re-read folds for a group/time mismatch; consider a one-off adversarial-
   validation diagnostic) — never silently swap the champion because the LB
   disagreed.
3. **Adversarial validation is NOT a standing gate.** It is a one-off diagnostic,
   used only when a large unexplained CV↔LB gap appears. Do not add it to the
   per-node suite, and never demote a node on its output.
4. **Every node clears this suite before its CV counts** — cleaning and
   feature-engineering nodes included. A feature that "improves CV" but fails
   `target_not_in_features` / fit-inside-fold is buggy, not good.
5. **Folds are frozen.** Read `folds.json`; never call `make_folds.py` again
   mid-run. Every transform fits inside the train fold only.
