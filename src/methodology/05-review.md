## 5. Review Protocol

**Review is mandatory, not aspirational.** Every phase has a defined review
type. The review must be performed before the next phase begins. Skipping
review — even under context pressure — is a process failure that produces
analyses with compounding gaps. See Section 3.0 for the gate protocol.

Review intensity is proportional to stakes. Heavyweight review is reserved for
gate points where errors are costly. Phases with high execution iteration
(data exploration, selection) use lightweight review to avoid bottlenecking
the natural trial-and-error of data analysis.

### 5.1 Review Classification

All reviews — regardless of intensity — use the same classification:

- **(A) Must resolve:** Errors or missing elements that would cause a
  collaboration reviewer to reject the analysis. Work cannot proceed until
  addressed.
- **(B) Should address:** Issues that weaken the analysis but don't invalidate
  it. Tracked and resolved before the analysis is finalized.
- **(C) Suggestions:** Style, clarity, or minor improvements.

### 5.2 Tiered Review Structure

| Phase | Review type | Rationale |
|-------|------------|-----------|
| Phase 1: Strategy | **4-bot + plot-validator** (physics + critical + constructive + arbiter) | Sets direction for everything. Physics errors propagate. Cheap phase, so review cost is well spent. |
| Phase 2: Execution | **Self-review** during exploration; **4-bot + plot-validator** after inference | Exploration is mostly mechanical. But the fit model, systematics, and final results need full tribunal review. |
| Phase 3: Review | **4-bot + plot-validator** (physics + critical + constructive + arbiter) | The final product. Worth the full treatment. |

**Plot-validator** is spawned alongside all other reviewers (in parallel) for
every phase that produces figures (all phases except Phase 1 strategy-only).
The plot-validator runs programmatic checks (not visual inspection) on all
plotting code and output data. Its findings are passed to the arbiter as
additional review input. Plot-validator red flags are automatic Category A —
the arbiter must not downgrade them. See `.claude/agents/plot-validator.md`
for the complete validation protocol.

**4-bot review** = physics reviewer + critical reviewer ("bad cop") +
constructive reviewer ("good cop") + arbiter. The **physics reviewer**
receives ONLY the physics prompt and the artifact — no methodology, no
conventions. It reviews as a senior collaboration member would: "Is this
physics correct? Is it complete? Would I approve this?"
The critical reviewer's goal is to find flaws — both in what is present
and in what is absent. The constructive reviewer's goal is to strengthen
the analysis — clarity, additional validation, improved presentation.
Reviewers run in parallel (they cannot see each other's work); the arbiter
reads all reviews (including the plot-validator report) and the original
artifact, adjudicates disagreements, and issues PASS / ITERATE / ESCALATE.

**1-bot review** = single critical reviewer + plot-validator. Issues
classified A/B/C. Plot-validator red flags are automatic Category A.
Executor addresses Category A items and re-submits. No arbiter needed.

**Self-review** = the executing agent explicitly reviews its own work before
producing the artifact. Plan review and code review happen within the session.
No separate agent invocation.

### 5.3 Reviewer Framing

The critical reviewer's job is not to check whether the artifact meets its
own stated criteria. It is to evaluate whether the artifact would survive
**external scrutiny** — a journal referee, a collaboration review committee,
or a competing group doing the same measurement independently.

The key question is not "does this pass its tests?" but "what would a
knowledgeable referee ask for that isn't here?" This requires the reviewer
to bring external standards to the evaluation:

- **Conventions:** What does the applicable `conventions/` document require
  for this analysis technique? Is anything missing?
- **Reference analyses:** What did published measurements of the same or
  similar observables do? Is anything they did missing here?

A reviewer that only checks internal consistency will miss the most
dangerous class of errors: things that are absent. Closure tests passing,
fits converging, and chi2 values below threshold are necessary but not
sufficient. The reviewer must also check that the *right things* are being
tested.

**Concrete operating principle:** Before concluding the review, the reviewer
must explicitly answer these questions:

1. Are all systematic sources listed in the applicable `conventions/`
   document either implemented or explicitly justified as inapplicable?
2. Have 2-3 published reference analyses been identified, and does this
   analysis match or exceed their systematic coverage?
3. If a competing group published a measurement of the same quantity next
   month, what would they have that we don't?

If any answer is "no" or "non-empty" without justification, those are
Category A findings. The arbiter must not issue PASS until all three are
resolved.

**Structural limitation of LLM-only review.** Bot reviewers share training
distributions and, consequently, blind spots. They catch structural errors
effectively — missing sections, incomplete systematic tables, inconsistencies
between the artifact and conventions. They are weaker on domain-specific
physics that falls outside their training or that requires genuine novel
reasoning (e.g., a subtle interaction between ISR treatment and the
particle-level definition that no indexed paper discusses). Conventions and
reference analyses partially mitigate this by providing external anchors the
reviewer can check against mechanically, but the mitigation is bounded by
the completeness of those documents.

### 5.4 Review Focus by Phase

| Phase | Review focus |
|-------|-------------|
| Strategy | Are backgrounds complete? Is the approach motivated by the literature? Does the systematic plan cover the standard sources for this analysis type (consult `conventions/`)? |
| Execution (exploration) | (Self-review) Are samples complete? Any data quality issues? Do distributions look physical? |
| Execution (selection) | Does the background model close? Is every cut motivated by a plot? Is signal contamination controlled? Cutflow counts monotonically non-increasing (Category A if violated)? **If MVA used:** is data/MC agreement acceptable? Was an alternative architecture tried? |
| Execution (inference) | Is the fit healthy? Are systematics complete — both internally consistent AND relative to conventions? Do signal injection tests pass? Are post-fit diagnostics clean? Are observed results consistent with expected? |
| Final results | See §5.4.3 below. |

#### 5.4.1 Completeness Review (Strategy and Inference)

Reviews at Phase 1 (Strategy) and Phase 2 inference must include an
explicit **completeness check** in addition to the standard correctness review.
The completeness check asks what is *missing*, not just whether what is
*present* is correct.

**At Phase 1:**
- Consult the applicable `conventions/` document for the analysis technique
  (e.g., `conventions/unfolding.md` for an unfolded measurement). Verify that
  the planned systematic program covers the standard sources listed there.
  Flag any omission as Category A unless the strategy explicitly justifies
  the omission.
- Verify that the strategy identifies 2-3 published reference analyses and
  tabulates their systematic programs. If this table is missing, flag as
  Category A.

**At inference (Phase 2.3):**

The executor must produce a **systematic completeness table** as a
mandatory section of the inference artifact. This table has two parts:

1. **Planned vs. implemented.** Every systematic source listed in the
   Phase 1 strategy must appear with its implementation status. Any source
   that was planned but dropped must have a justification. Silent omissions
   are Category A.

2. **Conventions and reference comparison.** Every source listed in the
   applicable `conventions/` document and in the reference analyses from
   Phase 1 must appear:

   ```
   | Source           | Conventions | Ref 1 | Ref 2 | This analysis | Status    |
   |------------------|-------------|-------|-------|---------------|-----------|
   | Hadronization    | Required    | P+H   | P+H+S | Pythia only   | MISSING   |
   | Selection cuts   | Required    | yes   | yes   | yes           | OK        |
   | Luminosity       | Normalized  | —     | —     | N/A (norm.)   | JUSTIFIED |
   ```

   The reviewer must verify this table **row by row**. Any row with status
   MISSING or PARTIAL is Category A unless the justification column
   explains why (e.g., "resource unavailable — documented as limitation
   in §4 and Future Directions"). The arbiter does not PASS with
   unresolved MISSING rows.

- Cross-check the conventions document's "required validation checks"
  against the artifact. For unfolded measurements, this includes the
  prior-sensitivity test and the particle-level validation plots.

This completeness review is the primary defense against the failure mode
where an analysis passes all *internal* consistency checks but omits a
standard systematic source. Internal consistency (closure tests pass, fits
converge) is necessary but not sufficient — the review must also check
external completeness (are we evaluating what the field considers standard?).

#### 5.4.2 Figure and Label Review (all phases producing figures)

Every review that evaluates figures — whether self-review, 1-bot, or multibot —
must include a mechanical pass over all figures checking the following (see
Appendix A for the plotting template that prevents most of these). These
are Category A if wrong:

- [ ] **√s and energy labels** match the actual dataset (not copied from a
  template for a different collider or energy)
- [ ] **Experiment name** is correct in all figure text and annotations
- [ ] **No figure titles** — use captions instead of `ax.set_title()`
- [ ] **Axis labels** include units in brackets; y-axis label matches the
  normalization actually applied
- [ ] **Luminosity / event count** annotations match the data sample used
- [ ] **Legend entries** match what is actually plotted (no stale labels from
  earlier iterations)
- [ ] **Aspect ratios and font sizes** are consistent across all figures
- [ ] **Bin widths** are noted on the y-axis label for variable-width binning
- [ ] **Uncertainties** are reasonable — error bars not suspiciously small or
  large relative to bin content
- [ ] **No clipped content** — data points, error bars, and legend fully visible
- [ ] **Appropriate scales** — log y-axis used when range spans >2 orders of
  magnitude, linear otherwise
- [ ] **Ratio panels** are readable — axis range appropriate, reference line
  visible, uncertainties shown
- [ ] **Systematic breakdown** is sensible — individual sources smaller than
  total, dominant source identified. No single source with relative uncertainty
  >100% in any bin (typically indicates a bug — see Appendix A)
- [ ] **Uncertainties make physical sense** — error bars proportional to bin
  content (for Poisson-dominated bins), systematics in the right ballpark
  for the analysis type
- [ ] **Log scale used appropriately** — y-axis spanning >2 orders of magnitude
  should use log scale; linear otherwise

This is a cheap check — it can be delegated to a dedicated reviewer as a
mechanical pass before or during review. Metadata errors in figures destroy
reviewer trust disproportionate to their severity, because they signal that
the author did not look at their own plots.

**Reviewers must read every plot.** Every figure produced by the phase must
be opened and inspected by the reviewer — not just referenced. The reviewer
must verify that the plot makes physical sense (distributions have expected
shapes, data/MC agreement is reasonable, uncertainties are proportional to
statistics, no unphysical features). A reviewer that skips plot inspection
is not reviewing.

**Known limitation: LLM visual inspection is unreliable.** Current models
are poor at catching text overlaps, alignment issues, clipped labels, and
figure sizing problems. To mitigate this:

- **Programmatic checks in plotting scripts.** Where possible, plotting code
  should validate properties that are hard to see: `assert not ax.get_title()`
  (no titles), check that axis labels are set, verify figure dimensions match
  a standard template. These run as part of the plot task and catch issues
  before review.
- **Standardized figure function.** Analyses should define a common plotting
  setup function (figure size, font sizes, axis formatting) used by all
  scripts, rather than configuring each plot independently. This prevents
  inconsistency by construction rather than by review.

#### 5.4.3 Plot Validation Protocol (all figure-producing phases)

The plot-validator agent runs alongside all other reviewers in every review
cycle that evaluates phases producing figures. Unlike human or LLM visual
inspection (which is unreliable for catching errors), the plot-validator
operates **programmatically** — examining plotting code, running the scripts,
and checking the output data against physics expectations.

**Why this exists.** The most common failure mode in LLM-driven HEP analysis
is producing plots that look syntactically correct but are physically
ridiculous — data/MC ratios of 10x, flat pT distributions, negative yields,
systematically shifted distributions. LLM reviewers reliably miss these
because they share training biases: they can parse a plot's structure but
cannot reliably judge whether the physics content makes sense. The
plot-validator compensates by applying quantitative, programmatic checks
that do not require visual judgment.

**Validation categories:**

**A. Code compliance** (from plotting scripts):
- `figsize=(10, 10)` (or multiples for subplots)
- No `ax.set_title()` calls
- Axis labels set with units in brackets
- `bbox_inches="tight"` and `dpi=200` at save time
- PNG format saved
- `plt.close(fig)` after saving

**B. Physics sanity** (from output data/histograms):
- All yields non-negative
- Efficiencies between 0 and 1
- Data/MC ratios in CRs between 0.5 and 2.0
- Uncertainties proportional to √N for Poisson-dominated bins
- pT/mass distributions fall off at high values (not flat or rising)
- Cutflow yields monotonically non-increasing
- Background fractions sum to ~100%
- Chi²/ndf for data/MC < 3.0 in control regions
- Total predicted yield within 2× of back-of-envelope (σ × L × ε)

**C. Consistency** (across plots and tables):
- Same process has consistent yield across different plots
- Pre-fit/post-fit yields consistent with fit result
- NP pulls mostly within ±2σ
- Impact rankings consistent with uncertainty breakdown

**D. Red flags** (automatic Category A):
- Any negative event yield
- Any efficiency > 1 or < 0
- Any data/MC ratio > 5.0 or < 0.2 in a control region
- Total uncertainty = 0 in any bin with non-zero content
- Chi²/ndf > 5.0 for any data/MC comparison
- Systematic variation > 100% in any bin
- Cutflow yield that increases at any step
- NP pull > 3σ for any parameter
- Fit non-convergence

Red flag findings from the plot-validator are passed directly to the arbiter
(in 4-bot reviews) or treated as Category A (in 1-bot reviews). The
arbiter must not downgrade red flags without explicit justification in the
ARBITER report.

**Integration with review cycle:** The plot-validator runs in parallel with
other reviewers. Its report (`PLOT_VALIDATION.md`) is read by the arbiter
alongside the physics, critical, and constructive reviews. In 1-bot reviews,
the critical reviewer reads the plot-validation report and incorporates its
findings.

#### 5.4.4 Final Results Review (Phase 3)

The Phase 3 review evaluates the final results as a **complete package** —
the reviewer should evaluate them as a journal referee would, not as someone
who has followed the analysis from Phase 1.

**Framing:** The reviewer examines the final results, figures, and tables.
The question is: "Based solely on what is here, am I convinced that this
result is correct and complete?"

**What this catches that earlier reviews may not:**
- A systematic source that was planned in Phase 1 but quietly dropped
- Validation evidence that exists in phase artifacts but was not included
  in the final results (e.g., particle-level data/MC plots were made but
  not shown)
- Logical gaps: a claim is made (e.g., "the MC accurately models the
  detector response") without the evidence to support it
- Quantitative results that don't add up (e.g., efficiencies or event
  counts that are inconsistent between tables)

**Required checks:**
- For every systematic source in the uncertainty table: is the method
  described, is the magnitude reported, and is validation evidence shown?
- For every comparison to a reference (published data, MC prediction):
  is a quantitative compatibility metric given, and is the level of
  agreement or tension interpreted?
- Does the result contain enough information that an independent analyst
  could reproduce the measurement? (Selection criteria, binning, unfolding
  parameters, MC samples, correction procedures.)
- Consult the applicable `conventions/` document one final time. Is
  anything required there that is absent from the final results?
- Figures pass the cosmetic checklist (§5.4.2).

The cost of this review is additional iteration loops if it finds gaps that
should have been caught earlier. That cost is acceptable — it is better to
iterate at Phase 3 than to publish an incomplete result.

### 5.4.5 Single-Session Review via Subagents

When the analysis runs in a single session, reviews are implemented by
spawning dedicated reviewer subagents. The reviewer subagent reads the phase
artifact from disk and applies the review criteria.

**Minimum review checklist (all phases):**

| Phase | Minimum checks |
|-------|---------------|
| 1: Strategy | Conventions consulted? Systematic plan covers standard sources? |
| 2: Execution (exploration) | Sample inventory complete? Data quality checked? Experiment log updated? |
| 2: Execution (selection) | Every cut motivated by a plot? Data/MC validation done? Cutflow complete? |
| 2: Execution (inference) | Systematic completeness table? Signal injection tests pass? Post-fit diagnostics clean? Results consistent with expected? |
| 3: Final review | `results/` directory populated? `analysis.py` runs and reproduces `results.json`? Figures pass cosmetic checklist (§5.4.2)? |

**No self-review fallback.** Strategy and final review require independent
reviewer subagents. Self-review is not an acceptable substitute — the
author reviewing their own work misses both correctness and completeness
failures.

### 5.5 Iteration and Escalation

For **4-bot reviews:** the cycle repeats until the arbiter issues PASS.
Correctness is the termination condition. The orchestrator emits warnings
after 3 iterations and a strong warning after 5 as signals that the issues
may require human input. A configurable hard cap (default 10) forces
escalation if reached — this is a safety net, not the intended termination
condition. The arbiter should ESCALATE rather than loop indefinitely.

For **1-bot reviews:** the executor addresses Category A items and re-submits.
These typically converge in 1–2 iterations; the orchestrator warns after 2 and
escalates after 3. Issues surviving 3 rounds of single-reviewer
feedback likely need a fundamentally different approach, not
another iteration of the same fix cycle.

For **self-review:** no formal iteration — the agent corrects issues as it
finds them during execution.

### 5.6 Cost Controls

To prevent runaway costs from pathological iteration:

**Review iteration warnings:** For **4-bot reviews**, the orchestrator emits a
warning after **3** iterations and a strong warning after **5**. For **1-bot
reviews**, the orchestrator warns after **2** and escalates after
**3** (1-bot issues that survive 3 rounds likely need a different approach).
These are soft thresholds — correctness remains the
termination condition. A configurable hard cap (`max_review_iterations`,
default 10) forces escalation if reached. In interactive mode, the
orchestrator surfaces warnings for guidance. In batch mode,
warnings are logged and the arbiter is prompted to consider ESCALATE.

### 5.7 Phase Regression

The pipeline is normally forward-only, but a reviewer or executor may discover
that a fundamental assumption from an earlier phase is wrong — a major
background was missed, the selection approach is unworkable, or the data
contradicts the strategy.

**Regression must be triggered, not avoided.** The natural tendency of
agents (and humans) is to paper over upstream problems with downstream
workarounds — adding a systematic to cover a mismodeled input, accepting a
poor closure test, or documenting an instability as a "known limitation."
This produces weak analyses. If a reviewer identifies a physics issue
traceable to an earlier phase, the correct action is regression, not
acceptance. Concrete triggers that must not be rationalized away:
- Data/MC disagreement on a variable that enters the observable or MVA
- Closure test failure (chi2 p-value < 0.05) in any validation
- Operating point instability (result varies significantly with cut value)
- An undocumented or unexplained dominant systematic
- MC used for periods/conditions it was not generated for

When this happens:

1. The agent (or reviewer) documents the issue and identifies which earlier
   phase it originates from
2. The issue is classified as a **regression trigger** in the review artifact
3. The orchestrator spawns an **Investigator** (see below) to assess the
   impact before any re-execution begins
4. Fixes are dispatched based on the Investigator's ticket

#### The Investigator

The Investigator is a dedicated agent role spawned when a
regression trigger is flagged. It does *not* read all artifacts end-to-end.
Instead it follows a structured, minimal-read process:

1. **Read the trigger description** from the review artifact that flagged the
   regression.
2. **Read the origin phase artifact** to identify the specific wrong
   conclusion(s).
3. **Trace forward phase by phase:** for each downstream artifact, read only
   the Summary and Method sections. If the artifact depends on the wrong
   conclusion, read the full affected sections and add them to the impact list.
   If it does not, mark the phase unaffected and stop tracing that branch.
4. **Produce `REGRESSION_TICKET.md`** containing:
   - Root cause and origin phase
   - Affected phases with specific sections that must change
   - Unaffected phases with reasoning for why they are safe
   - Fix scope per affected phase (what the executor must redo)

#### The fix cycle

The orchestrator dispatches fixes automatically.

- **Origin phase:** the executor re-runs with the previous artifact, the
  regression ticket, and the experiment log as inputs. The arbiter reviews the
  before/after to confirm the root cause is resolved.
- **Affected downstream phases:** each receives the same treatment (previous
  artifact + ticket + updated upstream artifact). Fixes proceed in phase order.
- **Unaffected phases:** skipped entirely, per the Investigator's assessment.

#### Timing

Regression can be triggered at any point during the analysis pipeline.
Once the final results are produced, discovered issues become iteration
items or documented observations, not regression triggers.

**Regression vs. documentation fix.** Not every issue found in review
requires regression. The distinction:
- **Physics issue** (wrong systematic treatment, missing background, flawed
  correction) → regression trigger. Re-run earlier phases.
- **Presentation issue** (axis label wrong, figure unclear, caption sparse,
  missing cross-reference) → Phase 3 iteration. Fix without re-running
  earlier phases.

#### Upstream feedback (non-blocking)

Any executor may proactively produce an `UPSTREAM_FEEDBACK.md` artifact when
it encounters something an earlier phase did not consider — an unexpected
background shape, a missing systematic, a data feature not mentioned upstream.
This is *not* a regression trigger and does not block execution. The
orchestrator routes it to the next review gate for the affected upstream phase.
If the reviewers agree the issue is material, they may flag a regression
trigger through the normal mechanism.

Phase regression is expensive and should be rare. The strategy and exploration
phases exist to prevent it. But when it happens, it is better to go back and
fix the foundation than to build on a known-faulty premise.

Regression is logged in a `regression_log.md` at the analysis root, tracking
what triggered the regression, the Investigator's ticket, which phases were
re-run, and what changed.

---
