---
name: check-status
description: Display current analysis pipeline status
user-invocable: true
---

# /check-status -- Analysis Status Report

Display the current status of the analysis pipeline.

**Arguments:** `$ARGUMENTS` (ignored)

## Instructions

1. Find the analysis directory by locating `STATE.md` in the current directory or immediate subdirectories (including under `analyses/`).

2. Read `STATE.md` in full.

3. Read `experiment_log.md` if it exists and is non-empty.

4. Present a status report in the following format:

```
=== Analysis Status ===

Current phase:  {phase number and name}
Status:         {executing | reviewing | passed | blocked | complete}
Last updated:   {timestamp from STATE.md}

--- Phase History ---

| Phase | Status | Artifact | Review Result | Iterations |
|-------|--------|----------|---------------|------------|
(reproduce the Phase History table from STATE.md)

--- Completed Artifacts ---

(For each passed phase, list the path to the final artifact file.)

Phase 1: STRATEGY.md, DATA_SURVEY.md
Phase 2: SELECTION.md, BACKGROUND.md, INFERENCE.md, analysis.py, results.json
Phase 3: review/ (arbiter verdict)
Phase 4: UNBLINDING.md, results.json (observed)
Phase 5: SUMMARY.md, STRATEGY.md (updated with Phase 4/5 results)
...only for phases that have status=passed

--- Blockers ---

(reproduce the Blockers section from STATE.md, or "None" if empty)

--- Review Iteration Counts ---

(For each phase that has been reviewed, report the number of review iterations.
 Count the number of files in each review/ subdirectory to estimate this.)

Phase 1: {N} iterations
Phase 2: {N} iterations
Phase 3: {N} iterations
Phase 4: {N} iterations
Phase 5: {N} iterations
...etc

--- Regressions ---

(If regression_log.md is non-empty, reproduce its contents.
 Otherwise: "No regressions recorded.")
```

5. If the status is `blocked`, add:

```
>>> BLOCKED <<<
Reason: {blocker description from STATE.md}
Human intervention is required to proceed.
```

6. If the status is `complete`, add:

```
>>> ANALYSIS COMPLETE <<<
Final results: results.json (observed)
Analysis script: analysis.py
Summary: SUMMARY.md
```
