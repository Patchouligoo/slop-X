## 3. Analysis Phases

The analysis proceeds through three phases: **Strategy**, **Execution**, and
**Review**. This matches the `/run-analysis` skill workflow. Within a phase,
**work should be parallelized where possible** — sub-delegate independent
tasks (systematic evaluations, plot generation) to concurrent sub-agents.

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
- **Measurement:** Corrected differential or inclusive cross-sections, event
  shape distributions, or extracted physical parameters (e.g., αs). No
  signal/background separation per se — the "signal" is the process being
  measured. The primary deliverable is a corrected spectrum or extracted
  parameter with full uncertainties.

Where phase descriptions below reference search-specific concepts (signal
region, control regions, S/B optimization), measurement analyses substitute
the analogous concepts: fiducial region, sideband/validation regions,
purity optimization. The review criteria adapt accordingly.

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

**For measurement analyses,** the agent must additionally:
- Define the observable(s) to be measured and their physical interpretation
- Identify the correction/unfolding strategy and what inputs it requires
- Survey prior measurements of the same observable
- Identify what theory predictions or MC generators can be compared to the
  corrected result

**Output artifact:** `STRATEGY.md` — a document covering the above points.
Quantitative estimates (cross-sections, expected yields) should cite sources;
order-of-magnitude estimates are acceptable where precision is unavailable.

**Review:** See Section 5. Strategy review evaluates physics soundness and
completeness of background enumeration.

---

### Phase 2: Execution

**Goal:** Implement the full analysis — explore data, develop selection,
estimate backgrounds, evaluate systematics, perform the fit, and produce
final results.

**Inputs:** Strategy, experiment context, data files.

This phase is the core of the analysis. It proceeds through four logical
stages, each producing its own artifact. The stages are sequential — each
builds on the previous — but work within a stage should be parallelized.

#### 2.1 Data Exploration

Survey the available data and establish the foundation for event selection.

**The agent must:**
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

**Output artifact:** `EXPLORATION.md` — sample inventory, data quality
summary, variable ranking with distributions, preselection cutflow.

#### 2.2 Selection & Background Estimation

Implement the analysis approach defined in the strategy.

**The agent must:**

*Selection:*
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

*Regions (search analyses):*
- Define control regions enriched in each major background. Document purity
  and the kinematic relationship to the signal region.
- Define validation regions for closure testing. These must be statistically
  independent of both CR and SR.

*Background estimation (search analyses):*
- Estimate background yields in SR using the chosen method per background
- Perform closure tests in validation regions: compare predicted yields to
  observation. Document agreement quantitatively. A closure test passes when
  agreement is consistent with statistical fluctuations (p-value > 0.05).
  A test that fails at p < 0.05 is Category A.

*Correction infrastructure (measurement analyses):*
- Produce data/MC comparisons for **all** kinematic variables entering the
  observable. Observable-level agreement can mask compensating category-level
  mismodeling.
- Construct the response matrix from MC. Report matrix properties: dimensions,
  diagonal fraction, condition number, efficiency.
- Implement the correction/unfolding chain end-to-end on MC. Run closure
  tests: unfold MC truth through the response and verify recovery. Run stress
  tests with reweighted truth.
- Closure test failure (chi2 p-value < 0.05) is Category A.
- **Binning must be justified.** Every binning choice must be motivated by
  detector resolution, statistical precision, or physics features.

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
details if applicable, region definitions, background estimates with
uncertainties, closure test results.

#### 2.3 Statistical Analysis

Evaluate systematic uncertainties, construct the statistical model, and
compute expected results.

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

*Expected results:*
- Compute expected results (limits, significance, or measurement precision)
- Produce fit diagnostics

**For measurements:** "Expected results" means the result extracted from
MC pseudo-data — not from real data.

**For measurement analyses:** the artifact must additionally include:
- The full bin-to-bin covariance matrix (statistical + each systematic source
  separately + total) as machine-readable files
- Comparison of the corrected result to at least one theory prediction or MC
  generator. Compute a chi2 or p-value using the full covariance matrix.

**Output artifact:** `INFERENCE.md` — systematic uncertainty table with
impacts, statistical model description, expected results, fit diagnostics.

#### 2.4 Validation & Final Results

Reality-check the analysis with data and produce final results.

**The agent must:**
- Optionally select 10% of data using a fixed random seed for an initial
  validation pass (recommended for large datasets)
- Run the full analysis chain on the complete dataset
- Produce post-fit diagnostics: nuisance parameter pulls, impact ranking,
  correlation matrix, goodness-of-fit
- Compare observed results to expected results. Report consistency
  quantitatively. Flag any result that disagrees with expected at >2σ.
- If results show anomalies (large NP pulls, poor GoF, unexpected signal),
  investigate and document whether these indicate a modeling problem or a
  genuine feature of the data
- **Produce `analysis.py`** — a self-contained Python script that performs
  the full analysis and writes `results.json`
- **Produce `results.json`** with the final physics results

**Output artifacts:** Updated `INFERENCE.md` with observed results,
`analysis.py`, `results.json`.

---

### Phase 3: Review

**Goal:** Validate the analysis through multi-bot review.

**Inputs:** All Phase 2 artifacts, `analysis.py`, `results.json`.

The orchestrator spawns the review protocol defined in Section 5:
- `physics-reviewer`, `critical-reviewer`, `constructive-reviewer` in parallel
- If figures exist, also spawn `plot-validator`
- `arbiter` synthesizes findings into a verdict

**Review outcomes:**
- **PASS** — Analysis is complete. Finalize results.
- **ITERATE** — Feedback provided. Re-spawn execution agents to address
  Category A findings, then re-review. Warn at iteration 3, hard cap at 10.
- **ESCALATE** — Fundamental issue found. Report failure with explanation.

**On PASS:** Verify `results.json` exists and contains valid results. Update
state to complete.

**Output:** Review artifacts in `review/` directory. Final `results.json`.

---
