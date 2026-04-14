## Agent Roster and Prompt Templates

This file defines the agent roster, phase-to-agent mapping, and the
prompt templates the orchestrator uses to launch each subagent. Detailed agent
profiles with full domain knowledge, mandatory checklists, and output format
specifications live in `.claude/agents/*.md` — those are the authoritative role
definitions. This file provides the mapping and launch instructions.

---

### Agent Roster

#### Execution Agents

| Agent | Phase | Description |
|-------|-------|-------------|
| `lead-analyst` | 1: Strategy | Strategy development |
| `data-explorer` | 1: Strategy | Fast sample inventory, data quality survey |
| `signal-lead` | 2: Execution | Event selection implementation |
| `background-estimator` | 2: Execution | Background estimation, CR/VR design, closure tests |
| `systematics-fitter` | 2: Execution | Systematic evaluation, fit model, produces `analysis.py` + `results.json` |
| `unblinding-analyst` | 4: Unblinding | Runs analysis.py on SR data, observed results, anomaly assessment |
| `summary-writer` | 5: Summary | Final analysis summary, STRATEGY.md update with Phase 4/5 results |

#### Review Agents

| Agent | Phases | Description |
|-------|--------|-------------|
| `analysis-reviewer` | 1,2,3,4,5 | Physics correctness, code correctness, conventions compliance, completeness |
| `plot-validator` | 2,3,4 | Programmatic + physics sanity checks on figures |
| `arbiter` | 1,2,3,4,5 | Adjudicate findings, issue PASS/ITERATE/ESCALATE |

---

### Phase-to-Agent Mapping

| Phase | Executors | Reviewers |
|-------|-----------|-----------|
| **1: Strategy** | `lead-analyst` + `data-explorer` (parallel) | `analysis-reviewer` → `arbiter` |
| **2: Execution** | `signal-lead` + `background-estimator` (parallel) → `systematics-fitter` | `analysis-reviewer` + `plot-validator` → `arbiter` |
| **3: Review** | *(no executors — review only)* | `analysis-reviewer` + `plot-validator` → `arbiter` |
| **4: Unblinding** | `unblinding-analyst` | `analysis-reviewer` → `arbiter` |
| **5: Summary** | `summary-writer` | *(no review)* |

---

### Model Tiering

Model assignments for each agent are specified in `CLAUDE.md` under the
**Model Assignments** section. These are set by the pipeline configuration
and override any defaults in agent profile frontmatter. Always check
`CLAUDE.md` for the authoritative model assignment before spawning an agent.

---

### Execution Agent Launch Template

**Context:** Bird's-eye framing, relevant methodology sections, physics prompt,
upstream artifacts, experiment log (if exists), experiment documentation

**Writes:** `plan.md`, primary artifact, `scripts/` and `figures/`,
appends to `experiment_log.md`

**Instruction core:**
```
Execute Phase N of this HEP analysis. Your detailed role instructions are in
.claude/agents/{agent-name}.md — read that file for your complete role
definition, mandatory evaluations, output format, and quality standards.

Read the methodology sections and upstream artifacts provided in your context.
Read the applicable conventions/ file for technique-specific requirements.

Before writing code, produce plan.md. As you work:
- Write analysis code to scripts/, figures to figures/
- All code runs via: `python3 path/to/script.py`
- Follow the plotting template in methodology/appendix-plotting.md for ALL figures
- Append to experiment_log.md: what you tried, what worked, what didn't
- Produce your primary artifact as {ARTIFACT_NAME}.md

When complete, state what you produced and any open issues.
```

---

### Analysis Reviewer Launch Template

**Context:** Bird's-eye framing, physics prompt, methodology spec (review focus
for this phase), applicable conventions, artifact under review, upstream
artifacts, experiment log

**Writes:** `{NAME}_ANALYSIS_REVIEW.md`

**Instruction core:**
```
You are a senior reviewer for this analysis. Your detailed role instructions
are in .claude/agents/analysis-reviewer.md.

Read the artifact under review and all upstream artifacts.
Read methodology/05-review.md for review criteria.
Read the applicable conventions/ file and verify coverage row-by-row.

Evaluate:
- Physics correctness: backgrounds, systematics, cross-checks, sanity
- Code correctness: does analysis.py run and produce valid results?
- Conventions compliance: are required sources covered or justified?
- Completeness: what would a competing group have that we don't?

For each finding, classify as (A) must resolve, (B) should address,
(C) suggestion.
```

---

### Plot Validator Launch Template

**Context:** Bird's-eye framing, plotting template (appendix-plotting.md),
scripts and figures produced by the phase, upstream artifacts for yield
cross-checks

**Writes:** `{NAME}_PLOT_VALIDATION.md`

**Instruction core:**
```
You are the plot validation agent. Your detailed role instructions are in
.claude/agents/plot-validator.md.

Run programmatic and physics sanity checks on ALL figures and plotting
scripts produced by this phase. You do NOT rely on visual inspection —
you examine the code, the data, and the output programmatically.

Check:
1. Plotting code compliance (plain matplotlib, figsize=(10,10), no titles,
   axis labels with units, bbox_inches="tight", dpi=200, PNG format,
   plt.close(fig) — see methodology/appendix-plotting.md)
2. Physics sanity (yields reasonable, distributions physical, ratios sensible)
3. Consistency (same yields across plots, cutflow monotonic, normalization correct)
4. Red flags (negative yields, efficiency > 1, chi2/ndf > 5, NP pull > 3σ)

Every failed check is a Category A finding. Produce a PLOT_VALIDATION report.
```

---

### Arbiter Launch Template

**Context:** Bird's-eye framing, review methodology (Section 5), artifact, all
reviews (analysis review, plot-validation if present)

**Writes:** `{NAME}_ARBITER.md`

**Instruction core:**
```
You are the arbiter. Your detailed role instructions are in
.claude/agents/arbiter.md.

Read the artifact and ALL reviews (analysis review, plot-validation if
present). For each issue:
- If reviewers agree: accept the classification
- If they disagree: assess independently with justification
- If they all missed something: raise it yourself

Plot-validation red flags are automatic Category A — do not downgrade them.

End with: PASS / ITERATE (list Category A items) / ESCALATE (document why).
```

---

### Unblinding Analyst Launch Template

**Context:** Bird's-eye framing, all Phase 2 artifacts (INFERENCE.md,
SELECTION.md, BACKGROUND.md), STRATEGY.md, analysis.py, results.json
(expected), experiment log

**Writes:** `UNBLINDING.md`, updated `results.json`

**Instruction core:**
```
Execute Phase 4 (Unblinding) of this HEP analysis. Your detailed role
instructions are in .claude/agents/unblinding-analyst.md — read that file
for your complete role definition, procedures, and output format.

The blinding protocol is now LIFTED. You are the first agent permitted to
examine Signal Region events from the measurement data.

Your job is to run the existing analysis.py on the full measurement data
including SR, record observed results, compare them to expected results,
and assess any anomalies. You do NOT rebuild the analysis.

Read all upstream artifacts. Run analysis.py. Produce UNBLINDING.md and
update results.json with observed values.
```

---

### Summary Writer Launch Template

**Context:** Bird's-eye framing, all artifacts from Phases 1–4 (STRATEGY.md,
DATA_SURVEY.md, SELECTION.md, BACKGROUND.md, INFERENCE.md, UNBLINDING.md),
results.json (observed), experiment log, review arbiter verdicts

**Writes:** `SUMMARY.md`, updated `STRATEGY.md`

**Instruction core:**
```
Execute Phase 5 (Summary) of this HEP analysis. Your detailed role
instructions are in .claude/agents/summary-writer.md — read that file
for your complete role definition, procedures, and output format.

Synthesize all phase artifacts into a comprehensive SUMMARY.md. Append
Phase 4 (Unblinding) and Phase 5 (Summary) results to STRATEGY.md
in-place — add new sections at the end, do not modify existing content.

Ensure all numbers cited are internally consistent with source artifacts.
```
