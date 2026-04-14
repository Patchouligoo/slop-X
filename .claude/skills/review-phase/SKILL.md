---
name: review-phase
description: Run the review cycle for a completed phase artifact
user-invocable: true
---

# /review-phase -- Run Review Cycle for a Phase

Run the review cycle for a completed phase artifact. The reviewer composition
varies by phase: phases that produce figures include the plot-validator.

**Arguments:** `$ARGUMENTS`

The argument is optionally a phase identifier: `1`, `2`, `3`, `4`, or `5`. If omitted, read STATE.md to determine the current phase.

## Step 1: Determine Phase and Reviewer Composition

1. Read `STATE.md` to confirm the analysis state.
2. If a phase argument was given, use it. Otherwise use the current phase from STATE.md.
3. Determine the reviewer composition for this phase:

| Phase | Reviewers | Plot-validator |
|-------|-----------|----------------|
| 1 (Strategy) | analysis-reviewer → arbiter | No |
| 2 (Execution) | analysis-reviewer + plot-validator → arbiter | Yes |
| 3 (Final Review) | analysis-reviewer + plot-validator → arbiter | Yes |
| 4 (Unblinding) | analysis-reviewer + plot-validator → arbiter | Yes |
| 5 (Summary) | analysis-reviewer → arbiter | No |

## Step 2: Locate the Artifact Under Review

Find the latest artifact for this phase:

| Phase | Artifact pattern | Location |
|-------|-----------------|----------|
| 1 | `STRATEGY.md`, `DATA_SURVEY.md` | working directory |
| 2 | `INFERENCE.md`, `SELECTION.md`, `BACKGROUND.md`, `analysis.py`, `results.json` | working directory |
| 3 | All Phase 1 and 2 artifacts + `figures/` | working directory |
| 4 | `UNBLINDING.md`, updated `results.json` | working directory |
| 5 | `SUMMARY.md`, updated `STRATEGY.md` | working directory |

Read the experiment log for this phase if it exists.

Identify all figures in the `figures/` directory -- these will be passed to the plot-validator (Phases 2-4).

## Step 3: Run the Review

Update STATE.md: `status: reviewing`, timestamp.

Initialize iteration counter: `iteration = 0`.

### Review Focus by Phase

Before spawning reviewers, note the focus area for the current phase:

- **Phase 1 (Strategy)**: Are backgrounds complete? Is the approach motivated by the literature? Does the systematic plan cover the standard sources for this analysis type (consult `conventions/`)? Are 2-3 published reference analyses identified? **Blinding compliance:** verify no SR events from measurement data were examined. Any violation is Category A.
- **Phase 2 (Execution)**: Is the fit healthy? Are systematics complete — both internally consistent AND relative to conventions? Does the systematic completeness table account for all planned sources? Do signal injection tests pass? Are post-fit diagnostics clean? Are expected results physically sensible? Does `analysis.py` run and produce valid `results.json`? **Blinding compliance:** verify all SR results use Asimov data only — no SR events from measurement data were examined. Any violation is Category A.
- **Phase 3 (Final Review)**: Review the complete analysis as a journal referee would. Check for: systematic sources planned in Phase 1 but dropped without justification, validation evidence that exists but was not included, logical gaps where claims lack supporting evidence, quantitative results that are inconsistent between tables. Does the result contain enough information for an independent analyst to reproduce the measurement? **Blinding compliance:** verify no SR events from measurement data were examined in any phase. Any violation is Category A.
- **Phase 4 (Unblinding)**: Are observed results consistent with expected (within 2σ or investigated)? Are NP pulls reasonable (< 2σ)? Is GoF acceptable (p-value > 0.05)? Are anomalies properly investigated and documented? Is the updated `results.json` valid and contains observed values? Was `analysis.py` run without unauthorized modifications?
- **Phase 5 (Summary)**: Is the summary complete and accurate? Does STRATEGY.md contain Phase 4 and Phase 5 results (appended sections)? Are all numbers internally consistent with source artifacts? Is the full analysis chain documented?

### Review Loop

Loop until PASS, ESCALATE, or max iterations:

1. Increment iteration counter.
2. Check cost controls:
   - If `iteration > 3`: log WARNING -- "Review iteration {iteration} for phase {phase}. Consider whether issues are fundamental enough to escalate."
   - If `iteration > 5`: log STRONG WARNING.
   - If `iteration >= 10`: force ESCALATE to human. Update STATE.md: `status: blocked`. Report and stop.

3. **Spawn reviewers based on phase composition:**

   **CRITICAL: You (the orchestrator) must NOT perform any review yourself.
   Every reviewer below MUST be spawned as a SEPARATE agent via the Agent
   tool. Do NOT summarize or assess the artifact in place of the reviewer.
   If you produce review conclusions without spawning the reviewer agent,
   that is a protocol violation.**

   **For Phases 2, 3, 4** (with plot-validator): Spawn `analysis-reviewer` and `plot-validator` **in parallel** via SendMessage.

   **For Phases 1, 5** (without plot-validator): Spawn `analysis-reviewer` only via SendMessage.

   Analysis reviewer instructions:
   - Read: the artifact under review, upstream artifacts, experiment log
   - Read: methodology spec (review focus for this phase), applicable conventions
   - Evaluate physics correctness, code correctness, conventions compliance, and completeness
   - Apply the phase-specific review focus listed above
   - Classify every issue as (A) must resolve, (B) should address, (C) suggestion
   - Write output to: `review/analysis/{REVIEW}.md` with session-named filename

   Plot-validator instructions (Phases 2-4 only):
   - Read: all figures in the `figures/` directory
   - Read: `conventions/` plotting standards (axis labels, font sizes, color schemes, legend placement, ratio panels, style requirements)
   - Validate each figure against the conventions
   - Check: axis labels and units, legend completeness, ratio panel presence where required, color accessibility, resolution and format, statistical uncertainty display
   - Run programmatic physics sanity checks on plotting code and output data
   - Classify issues as (A) must fix, (B) should fix, (C) cosmetic suggestion
   - Write output to: `review/plot-validation/{REVIEW}.md`

3a. **Verify review files exist.** Before spawning the arbiter, confirm:
   - `review/analysis/` contains a new file from this review iteration
   - For Phases 2/3/4: `review/plot-validation/` also has a new file
   If any expected file is missing, re-spawn the missing reviewer(s).
   Do NOT proceed to the arbiter without on-disk review artifacts.

4. **After all reviewers complete, spawn arbiter** via SendMessage:
   - Read: the artifact, all review files (latest from `review/analysis/`, and `review/plot-validation/` if it exists)
   - For each issue: if multiple reviewers agree, accept; if they disagree, assess independently; if all missed something, raise it
   - Incorporate plot-validator findings: Category A plot issues are treated as Category A overall
   - Write output to: `review/arbiter/` with session-named filename
   - End with a clear decision: **PASS**, **ITERATE** (list Category A items including plot issues), or **ESCALATE** (document why)

5. **Read the arbiter decision** from the latest file in `review/arbiter/`.

6. **Handle the decision:**

   - **PASS**: Check for regression triggers in the review output. If none found, update STATE.md (`status: passed`), record in Phase History table (including iteration count), and return the result.

   - **ITERATE**: Re-spawn the phase executor via SendMessage with:
     - All original inputs (prompt, methodology, upstream artifacts)
     - The arbiter's feedback (Category A items to address, including plot fixes)
     - The previous artifact version
     - The experiment log
     - Instruction: "Address the Category A issues identified by the arbiter. Produce an updated artifact."
     - After executor completes, loop back to step 1 of the review.

   - **ESCALATE**: Update STATE.md (`status: blocked`, record escalation reason). Report to user: "Phase {phase} review escalated. Reason: {reason}. Human intervention required." Stop.

## Step 4: Regression Detection

After any PASS decision, scan all review outputs for regression triggers. A regression trigger is any statement indicating that work done in a prior phase is now known to be incorrect, incomplete, or based on wrong assumptions. Examples:
- "The background estimation in Execution used an incorrect normalization"
- "The systematic uncertainty source X was overlooked in the Strategy"
- "The selection criteria from Execution are based on an outdated assumption"

If a regression trigger is found:
1. Update STATE.md: status=regression
2. Spawn an `investigator` agent with the trigger description
3. Investigator produces `REGRESSION_TICKET.md`
4. Re-run the origin phase, re-review, and re-run affected downstream phases
5. Log in `regression_log.md`

## Step 5: Report Results

After the review completes, report:

```
Phase {phase} review: {PASS | ESCALATE}
  Reviewers: {list of reviewer types used}
  Plot validation: {included | not applicable}
  Iterations: {count}
  Artifact: {path to final artifact}
  Decision: {arbiter decision}
  Category A issues resolved: {count if any}
  Plot issues resolved: {count if any}
  Regression triggers: {none | description}
```
