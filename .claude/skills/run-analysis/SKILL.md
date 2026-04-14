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

This is a simplified pipeline adapted for bump hunt analyses. It uses a 5-phase workflow:
1. **Strategy** — Plan the analysis approach
2. **Execution** — Implement selection, background estimation, and fit (blinded)
3. **Review** — Validate the results (blinded)
4. **Unblinding** — Run analysis on actual SR data, produce observed results
5. **Summary** — Document the full analysis chain, update STRATEGY.md

**Blinding protocol:** During Phases 1–3, agents must NOT access Signal Region
events from the measurement data. Only control region data, MC samples, and
sideband regions are permitted. See `methodology/03-phases.md` for details.

The pipeline produces `results.json` with observed `{"mu_val": <float>, "mu_err": <float>}` after unblinding.

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

**Blinding protocol active:** Agents must not access SR events from the
measurement data during this phase.

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

**Blinding protocol active:** Agents must not access SR events from the
measurement data during this phase. All SR results must use Asimov data.

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
     - Supports a `--blinded` flag: when set, substitutes Asimov data in SR
       instead of actual SR events. When absent, uses all data including SR.
   - Run `analysis.py --blinded` and verify it produces valid `results.json`
     containing expected results. Do NOT run without `--blinded` during this phase.
   - If it fails, debug and fix until it works
   - Output: `INFERENCE.md`, `analysis.py`, `results.json` (expected)
4. Run `/review-phase 2` to review the execution artifacts.
   - On PASS: proceed to Phase 3
   - On ITERATE: re-spawn `systematics-fitter` with arbiter feedback, loop
   - On ESCALATE: report failure and stop

## Step 5: Phase 3 — Final Review

1. Update STATE.md: status=reviewing
2. Run `/review-phase 3` for final results review:
   - Reviews the complete analysis as a journal referee would
   - On PASS: proceed to Phase 4 (Unblinding)
   - On ITERATE: re-spawn relevant agent(s) with feedback, loop
   - On ESCALATE: report failure

## Step 6: Phase 4 — Unblinding

**Blinding protocol lifted.** The `unblinding-analyst` is the first agent
permitted to examine SR events from the measurement data.

1. Update STATE.md: status=unblinding
2. Spawn `unblinding-analyst` agent:
   - Task: Run the existing `analysis.py` without `--blinded` on the full measurement data including SR
   - Inputs: All Phase 2 artifacts, `analysis.py`, `results.json` (expected), `STRATEGY.md`
   - The blinding protocol is now lifted — this agent may access SR data
   - Run `analysis.py` (without `--blinded`) to produce observed mu_val and mu_err
   - Must compare observed vs expected, flag anomalies
   - Must produce post-unblinding diagnostic plots (data overlaid in SR)
   - Does NOT rebuild the analysis or modify analysis.py — runs it as-is
   - Output: `UNBLINDING.md`, updated `results.json` (observed)
3. Run `/review-phase 4` to review unblinding results.
   - On PASS: proceed to Phase 5
   - On ITERATE: re-spawn `unblinding-analyst` with arbiter feedback, loop
   - On ESCALATE: report failure and stop

## Step 7: Phase 5 — Summary

1. Update STATE.md: status=summarizing
2. Spawn `summary-writer` agent:
   - Task: Produce final analysis summary and update STRATEGY.md
   - Inputs: All artifacts from Phases 1–4, all review artifacts
   - Must document the full analysis chain
   - Must append Phase 4 and Phase 5 results to STRATEGY.md (in-place, at the end)
   - Output: `SUMMARY.md`, updated `STRATEGY.md`
3. Run `/review-phase 5` to review the summary.
   - On PASS: proceed to finalization
   - On ITERATE: re-spawn `summary-writer` with arbiter feedback, loop
   - On ESCALATE: report failure and stop

## Step 8: Finalize

1. Verify `results.json` exists and contains valid observed `mu_val` and `mu_err`
2. Verify `SUMMARY.md` exists
3. Verify `STRATEGY.md` contains Phase 4 and Phase 5 results
4. If any deliverable is missing or invalid, report failure
5. Update STATE.md: status=complete
6. Report: "Analysis complete. Observed results: mu_val={value}, mu_err={value}"

## Environment

- All scripts run via `python3 <script>` using the active conda environment.
- Available packages are listed in CLAUDE.md.
- Do NOT install packages or modify the environment.

## Cost Controls

- Track review iteration counts
- Warn at iteration 3
- Hard cap at iteration 10 — report best available result even if review didn't pass
