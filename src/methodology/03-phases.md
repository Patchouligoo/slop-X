## 3. Analysis Phases

The analysis proceeds through five phases: **Strategy**, **Execution**,
**Review**, **Unblinding**, and **Summary**. This matches the `/run-analysis`
skill workflow. Within a phase, **work should be parallelized where possible**
— sub-delegate independent tasks (systematic evaluations, plot generation) to
concurrent sub-agents.

### Blinding Protocol

During Phases 1–3, the Signal Region (SR) of the measurement data is
**blinded**. This is a hard rule — no agent may read, plot, fit, count, or
otherwise examine SR events from the measurement data during these phases.

**What is permitted during Phases 1–3:**
- Control region data — unrestricted access
- Monte Carlo simulation samples — unrestricted access
- Measurement data in sideband / control regions only (events outside the SR)
- Asimov (expected) data in the SR for fit validation and expected results

**What is forbidden during Phases 1–3:**
- Reading SR events from the measurement data
- Plotting SR distributions from the measurement data
- Fitting to SR events from the measurement data
- Counting observed events in the SR from the measurement data
- Any operation that reveals the observed SR event count or distribution

**Enforcement.** Blinding is enforced through review. Reviewers in Phases 1–3
must verify blinding compliance as part of every review cycle. Any operation
that reveals observed SR event counts or distributions from the measurement
data is a **Category A** finding (must resolve before advancement). This
applies to all agents, including the data-explorer. There are no exceptions
— no "quick peek", no "just checking the total count."

**When blinding is lifted.** Blinding is lifted in Phase 4 (Unblinding),
which is the first and only phase where SR events from the measurement data
are examined.

### 3.0 Artifact and Review Gates

**Every phase boundary is a hard gate.** The agent must not begin Phase N+1
until Phase N has produced its required artifact AND the required review has
been performed. This applies regardless of execution mode (orchestrated
multi-agent or single-session).

**The gate protocol:**
1. Phase executor produces the phase artifact (written to disk as markdown)
2. Phase executor updates the experiment log with what was done
3. The required review is performed (see Section 5)
4. Review findings are addressed (Category A items resolved)
5. Only then does the next phase begin

**Artifact existence is a precondition.** If Phase 2 has no artifacts on
disk, Phase 3 must not start. The artifact is both the handoff document and
the proof that the phase was completed with appropriate rigor.

**Why this is a hard rule:** Without artifact gates, agents compress or skip
phases to reach results faster. The result is an analysis with no audit trail,
no intermediate review, and gaps that compound. The artifacts exist to force
the agent to consolidate what it knows before moving on — the act of writing
the artifact surfaces gaps that coding alone does not.

Skipping a phase artifact is never acceptable, even under context pressure.
If context limits are approaching, the agent should write the artifact for the
current phase, commit, and stop — not rush through remaining work without
artifacts.

---

### Analysis Types

The spec supports two analysis types. The physics prompt must declare which
type applies; the agent confirms this during Phase 1.

- **Search / limit-setting:** Signal vs. background discrimination, signal
  region / control region structure, expected limits or significance as
  the primary deliverable.

---

### Phase 1: Strategy

**Goal:** Produce a written analysis strategy that a collaboration reviewer
could approve.

**Inputs:** Physics prompt, experiment context.

**The agent must:**
- Understand the detector capabilities, available datasets, and any prior
  work in the same or related channels
- Identify the signal process, production mechanism, and final state
- Enumerate principal backgrounds and classify them (irreducible, reducible,
  instrumental) with expected relative importance
- Propose an event selection approach with justification for the choice
- Define the discriminating variable(s) for the final statistical interpretation
- Outline the background estimation strategy per background and identify
  control regions needed
- List anticipated systematic uncertainty categories (experimental, theoretical).
  **Before finalizing the systematic plan,** read the applicable `conventions/`
  document and enumerate every required source. For each source, state whether
  it will be implemented and, if not, why not.
- Identify which collision data and simulation samples are needed

**Output artifact:** `STRATEGY.md` — a document covering the above points.
Quantitative estimates (cross-sections, expected yields) should cite sources;
order-of-magnitude estimates are acceptable where precision is unavailable.

In parallel with strategy development, the `data-explorer` agent surveys
the available data files and produces `DATA_SURVEY.md`.

**The data explorer must:**
- Inventory available samples: discover file structure, column names,
  number of events, data types
- Validate data quality: check for pathologies (empty columns, outliers,
  discontinuities in distributions) and document findings
- Survey discriminating variables: produce distributions of kinematic
  observables for signal and principal backgrounds, rank by separation power
- Establish baseline event yields after preselection

**Data discovery.** The agent should expect to discover the data format at
runtime. To avoid wasting time and memory:
1. **Metadata first.** Inspect column names and types before loading event
   data.
2. **Small slice first.** Load ~1000 events first. Do not attempt to load
   the full dataset until you know the schema.
3. **Document the schema.** The discovered structure, event counts, and any
   format quirks are artifact content.

**Output artifacts:** `STRATEGY.md`, `DATA_SURVEY.md`.

**Review:** See Section 5. Strategy review evaluates physics soundness and
completeness of background enumeration.

---

### Phase 2: Execution

**Goal:** Implement the full analysis — develop selection, estimate
backgrounds, evaluate systematics, perform the fit, and produce final
results.

**Inputs:** Strategy, data survey, experiment context, data files.

This phase is the core of the analysis. It proceeds through three logical
stages. The first two (selection and background estimation) run in parallel;
the third (statistical analysis) follows after both complete.

#### 2.1 Selection

Implement event selection based on the strategy. This runs in parallel with
background estimation (§2.2).

**The agent must:**
- Implement event selection (preselection + final selection or MVA, as
  determined by the strategy and what the data supports)
- **Default to multivariate techniques** (BDT, neural network) when the
  task involves classification in more than one dimension — flavour tagging,
  signal/background separation, event categorization. Rectangular cuts are
  acceptable only for preselection, single-variable selections with clear
  physical motivation, or when the training sample is too small (< ~1000
  events).
- If using multivariate techniques: train, validate (overtraining checks),
  and optimize the classifier. Document feature importance and working
  point choice.
  - **Train at least one alternative architecture** (e.g., NN if primary is
    BDT, or vice versa). Report both performances.
  - **When the physics has >2 classes**, try multiclass classification.
  - **Check data/MC agreement on the classifier output.**
  - **Sub-delegate MVA training** to a sub-agent.
- If cut-based: optimize cut values with a figure of merit and document N-1
  distributions.
- Produce the final signal region definition and cutflow with efficiencies.
  **Cutflow counts must be monotonically non-increasing** — if any cut
  increases the event count, something is structurally wrong. This is
  Category A if violated.
- **Every cut must be motivated by a plot.** The artifact must include, for
  each selection cut, the distribution of the cut variable showing signal
  and background.
- Define control regions enriched in each major background (search analyses).
  Document purity and the kinematic relationship to the signal region.
- Define validation regions for closure testing. These must be statistically
  independent of both CR and SR.
- Write analysis scripts to `scripts/` and produce diagnostic figures in
  `figures/`

**Sensitivity optimization.** If the expected sensitivity after the initial
selection is insufficient, the agent must systematically explore alternative
approaches. The agent maintains a **sensitivity log** (`sensitivity_log.md`)
tracking each approach tried, the figure of merit achieved, and the limiting
factor.

The exploration should progress through qualitatively different strategies:
1. *Optimize the current approach* — tune cuts for S/√B or equivalent
2. *Try a more powerful discriminant* — move from cut-based to BDT
3. *Try different extraction strategies* — shape fit vs counting experiment
4. *Try different signal extraction techniques* — rebinned histograms,
   classifier output reshaping, categorization by S/B ratio
5. *Revisit region design* — tighter signal region, different background
   decomposition

The agent should stop optimizing when:
- The sensitivity meets the physics goal, **or**
- At least 3 materially different approaches have been tried, **and**
- The most recent improvement was marginal (<10% relative), **and**
- The remaining ideas are increasingly speculative

**Workflow pattern.** This stage is iteration-heavy. Follow this progression:
1. **Setup & prototype.** Build the processing pipeline on a small slice
   (~1000 events). Verify it runs end-to-end.
2. **Full processing.** Run on the full dataset. Produce data/MC comparisons.
3. **Inspect & validate.** Systematically review all produced plots.

**Output artifact:** `SELECTION.md` — selection definition, cutflow, MVA
details if applicable, region definitions.

#### 2.2 Background Estimation

Estimate backgrounds and validate with closure tests. This runs in parallel
with selection (§2.1), using control region data for validation.

**The agent must:**
- Estimate background yields in SR using the chosen method per background
- Perform closure tests in validation regions: compare predicted yields to
  observation. Document agreement quantitatively. A closure test passes when
  agreement is consistent with statistical fluctuations (p-value > 0.05).
  A test that fails at p < 0.05 is Category A.

**Output artifact:** `BACKGROUND.md` — background estimates with
uncertainties, closure test results.

#### 2.3 Statistical Analysis & Final Results

Evaluate systematic uncertainties, construct the statistical model, produce
results, and generate the self-contained analysis script. This stage runs
after both selection and background estimation are complete.

**The agent must:**

*Systematics:*
- Evaluate experimental systematic uncertainties as rate and/or shape
  variations
- Evaluate theory systematic uncertainties as rate and/or shape variations
- Quantify the impact of each source on signal and background yields
- Prune negligible sources and document the pruning criterion

*Statistical model:*
- Construct the fit model with all signal and background components and
  systematic uncertainty terms
- Validate the model: verify the fit converges and expected results are
  physically sensible
- Perform signal injection tests: inject signal at known strengths and
  verify the fit recovers them

*Goodness-of-fit:*
- Report **both** chi2/ndf for quick assessment **and** toy-based p-value
  where binned results are involved
- chi2/ndf ~ 1 is good; >>1 indicates mismodeling; <<1 indicates
  overestimated uncertainties

*Results:*
- Compute expected results (limits, significance, or measurement precision)
  using Asimov data in the SR (blinding protocol)
- Produce post-fit diagnostics: nuisance parameter pulls, impact ranking,
  correlation matrix, goodness-of-fit — all using Asimov SR data
- Verify that expected results are physically sensible
- **Produce `analysis.py`** — a self-contained Python script that reads
  data files from the current directory, performs the full analysis
  (selection, background estimation, fit), and writes `results.json` with
  `{"mu_val": <float>, "mu_err": <float>}`. The script must support a
  `--blinded` flag: when set, the script substitutes Asimov (expected)
  data in the SR instead of using actual SR events from the measurement
  data. When the flag is absent, the script uses all data including SR.
- Run `analysis.py --blinded` and verify it produces valid `results.json`
  containing expected results
- **Produce `results.json`** with the **expected** physics results (from
  the blinded run). Observed results are produced in Phase 4.

**Output artifacts:** `INFERENCE.md` — systematic uncertainty table with
impacts, statistical model description, expected results, fit diagnostics.
`analysis.py`, `results.json` (expected).

---

### Phase 3: Review

**Goal:** Validate the analysis through multi-bot review.

**Inputs:** All Phase 2 artifacts, `analysis.py`, `results.json`.

The orchestrator spawns the review protocol defined in Section 5:
- `physics-reviewer`, `critical-reviewer`, `constructive-reviewer` in parallel
- If figures exist, also spawn `plot-validator`
- `arbiter` synthesizes findings into a verdict

**Review outcomes:**
- **PASS** — Blinded analysis validated. Proceed to unblinding.
- **ITERATE** — Feedback provided. Re-spawn execution agents to address
  Category A findings, then re-review. Warn at iteration 3, hard cap at 10.
- **ESCALATE** — Fundamental issue found. Report failure with explanation.

**On PASS:** Verify expected `results.json` exists and contains valid results.
Proceed to Phase 4 (Unblinding).

**Output:** Review artifacts in `review/` directory. Expected `results.json`.

---

### Phase 4: Unblinding

**Goal:** Run the analysis on actual Signal Region data and produce observed
results.

**Inputs:** All Phase 2 artifacts (passed review), `analysis.py`,
`results.json` (expected results from the blinded analysis).

**Precondition:** Phase 3 review must have issued PASS. The blinding protocol
is lifted for this phase — the `unblinding-analyst` is the first and only
agent permitted to examine SR events from the measurement data.

**The agent must:**
- Run the existing `analysis.py` (produced in Phase 2) **without** the
  `--blinded` flag, so it uses actual SR events from the measurement data
- Record the observed `mu_val` and `mu_err`
- Compare observed results to expected results (from Phase 2's blinded run)
  quantitatively. Report the difference in units of sigma. Flag any result
  inconsistent at > 2σ
- Run post-fit diagnostics with real SR data:
  - Post-fit distributions with data overlaid in all regions including SR
  - Nuisance parameter pulls (flag > 2σ)
  - Goodness-of-fit (flag p-value < 0.05)
- Assess anomalies: if NP pulls are large, GoF is poor, or the observed
  signal is unexpected, investigate whether these indicate a modeling problem
  or a genuine feature of the data. Document the assessment
- Update `results.json` with observed `mu_val` and `mu_err`

**The agent must NOT:**
- Rebuild the analysis or change the selection, background estimation, or
  fit model. Phase 4 runs the existing analysis — it does not redesign it
- Modify `analysis.py` in any way. The script already supports unblinded
  mode (run without `--blinded`)

**Output artifact:** `UNBLINDING.md` — observed results, observed vs expected
comparison, anomaly assessment, post-fit diagnostics with SR data. Updated
`results.json`.

**Review:** See Section 5. Unblinding review evaluates whether the observed
results are consistent with expectations and whether anomalies are properly
investigated.

---

### Phase 5: Summary

**Goal:** Produce a final summary documenting the complete analysis chain and
append results to `STRATEGY.md`.

**Inputs:** All artifacts from Phases 1–4, all review artifacts.

**Precondition:** Phase 4 review must have issued PASS.

**The agent must:**
- Read all phase artifacts (STRATEGY.md, DATA_SURVEY.md, SELECTION.md,
  BACKGROUND.md, INFERENCE.md, UNBLINDING.md) and review artifacts
- Produce `SUMMARY.md` — a comprehensive summary of the analysis covering:
  - Signal process and selection strategy
  - Background estimation and systematic uncertainty budget
  - Expected results (from blinded analysis)
  - Observed results (from unblinding)
  - Anomaly assessment and interpretation
  - Lessons learned and potential improvements
- Append Phase 4 and Phase 5 results to `STRATEGY.md` in-place, adding new
  sections at the end. Do not overwrite existing Phase 1 content — the
  original strategy remains intact, with unblinding and summary results
  appended below
- Ensure internal consistency: all numbers cited in the summary must match
  the source artifacts

**Output artifacts:** `SUMMARY.md` — full analysis chain summary. Updated
`STRATEGY.md` with Phase 4 and Phase 5 results appended.

**Review:** See Section 5. Summary review evaluates completeness, accuracy,
and internal consistency of the documentation.

---
