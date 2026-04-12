---
name: run-analysis
description: Run a simplified HEP bump hunt analysis pipeline using specialist agents
user-invocable: true
---

# /run-analysis -- Simplified Bump Hunt Analysis Pipeline

You are the pipeline orchestrator. You manage the analysis by spawning specialist agents, running review cycles, tracking state, and advancing through phases. You do NOT perform analysis work yourself.

**Arguments:** `$ARGUMENTS`

The argument is the physics prompt (inline text or a path to a `.md` file).

## Overview

This is a simplified pipeline adapted for bump hunt analyses. It uses a 3-phase workflow:
1. **Strategy** — Plan the analysis approach
2. **Execution** — Implement selection, background estimation, and fit
3. **Review** — Validate the results

The pipeline produces `results.json` with `{"mu_val": <float>, "mu_err": <float>}`.

## Step 1: Read Context

1. Read `CLAUDE.md` for task-specific instructions (data files, output format, physics hints).
2. Read `conventions/search.md` for bump hunt domain knowledge.
3. If `methodology/` exists, read:
   - `methodology/03-phases.md` — phase requirements and deliverables
   - `methodology/05-review.md` — review tiers and iteration rules
4. If `orchestration/` exists (via symlink or in the `.claude/` parent), read:
   - `orchestration/agents.md` — agent role definitions and model tiers
   - `orchestration/automation.md` — automation pseudocode and review loop logic
   - `orchestration/sessions.md` — session naming and isolation conventions

## Step 2: Initialize

Create the working directory structure:
```
scripts/          # Analysis scripts
figures/          # Diagnostic plots
review/           # Review outputs
  critical/
  physics/
  constructive/
  arbiter/
  plot-validation/
```

Write `STATE.md`:
```markdown
# Analysis State
- **Status**: initialized
- **Last updated**: {timestamp}
```

Write `experiment_log.md` as empty.

## Step 3: Phase 1 — Strategy

1. Update STATE.md: status=strategy
2. Spawn `lead-analyst` agent:
   - Task: Develop a bump hunt analysis strategy
   - Inputs: `CLAUDE.md` (physics prompt), `conventions/search.md`
   - Output: `STRATEGY.md`
   - Key deliverables: signal region definition, background estimation method, selection approach, fit method
3. Spawn `data-explorer` agent in parallel:
   - Task: Survey the data files
   - Output: `DATA_SURVEY.md`
4. Wait for both to complete.
5. Run `/review-phase 1` to review the strategy.
   - On PASS: proceed to Phase 2
   - On ITERATE: re-spawn `lead-analyst` with arbiter feedback, loop
   - On ESCALATE: report failure and stop

## Step 4: Phase 2 — Execution

1. Update STATE.md: status=executing
2. Spawn `signal-lead` and `background-estimator` agents in parallel:
   - `signal-lead`: Implement event selection based on STRATEGY.md
     - Write analysis scripts to `scripts/`
     - Produce diagnostic figures in `figures/`
     - Output: `SELECTION.md`
   - `background-estimator`: Estimate backgrounds, perform closure tests
     - Use control region data for validation
     - Output: `BACKGROUND.md`
3. After both complete, spawn `systematics-fitter`:
   - Construct the fit model, estimate mu_val and mu_err
   - Input: STRATEGY.md, SELECTION.md, BACKGROUND.md, data files
   - **CRITICAL**: Must produce `analysis.py` — a self-contained Python script that:
     - Reads input files from the current directory
     - Performs the full analysis (selection, background estimation, fit)
     - Writes `results.json` with `{"mu_val": <float>, "mu_err": <float>}`
   - Run `analysis.py` and verify it produces valid `results.json`
   - If it fails, debug and fix until it works
   - Output: `INFERENCE.md`, `analysis.py`, `results.json`
4. Run `/review-phase 2` to review the execution artifacts.
   - On PASS: proceed to Phase 3
   - On ITERATE: re-spawn `systematics-fitter` with arbiter feedback, loop
   - On ESCALATE: report failure and stop

## Step 5: Phase 3 — Final Review

1. Update STATE.md: status=reviewing
2. Run `/review-phase 3` for final results review:
   - Reviews the complete analysis as a journal referee would
   - On PASS: proceed to finalization
   - On ITERATE: re-spawn relevant agent(s) with feedback, loop
   - On ESCALATE: report failure

## Step 6: Finalize

1. Verify `results.json` exists and contains valid `mu_val` and `mu_err`
2. If `results.json` is missing or invalid, report failure
3. Update STATE.md: status=complete
4. Report: "Analysis complete. Results: mu_val={value}, mu_err={value}"

## Environment

- All scripts run via `python3 <script>` using the active conda environment.
- Available packages are listed in CLAUDE.md.
- Do NOT install packages or modify the environment.

## Cost Controls

- Track review iteration counts
- Warn at iteration 3
- Hard cap at iteration 10 — report best available result even if review didn't pass
