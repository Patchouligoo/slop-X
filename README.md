# SLOP-X

LLM-driven HEP bump hunt analysis framework. An orchestrator agent delegates
work to specialist subagents through five sequential phases with a blinding
protocol, producing a self-contained analysis script and measured signal
strength.

## How it works

```
┌─────────────────────────────────────────────────────────────┐
│                      ORCHESTRATOR                           │
│  Never writes code. Holds: prompt, summaries, verdicts only │
└─────┬───────────────────────────────────────────────────────┘
      │
      ▼
 ┌──────────┐   ┌──────────────┐   ┌──────────┐   ┌─────────────┐   ┌──────────┐
 │ Phase 1  │──▶│   Phase 2    │──▶│ Phase 3  │──▶│   Phase 4   │──▶│ Phase 5  │
 │ Strategy │   │  Execution   │   │  Review  │   │ Unblinding  │   │ Summary  │
 │ (4-bot)  │   │  (4-bot)     │   │ (4-bot)  │   │  (4-bot)    │   │ (4-bot)  │
 └──────────┘   └──────────────┘   └──────────┘   └─────────────┘   └──────────┘
       ◄──── BLINDED (SR data forbidden) ────►
```

### Phases

| Phase | Executors | Review | Key deliverables |
|-------|-----------|--------|------------------|
| **1. Strategy** | `lead-analyst` + `data-explorer` | 4-bot | `STRATEGY.md`, `DATA_SURVEY.md` |
| **2. Execution** | `signal-lead` + `background-estimator` → `systematics-fitter` | 4-bot | `SELECTION.md`, `BACKGROUND.md`, `INFERENCE.md`, `analysis.py`, `results.json` |
| **3. Review** | *(none — review only)* | 4-bot | PASS / ITERATE / ESCALATE |
| **4. Unblinding** | `unblinding-analyst` | 4-bot | `UNBLINDING.md`, updated `results.json` |
| **5. Summary** | `summary-writer` | 4-bot | `SUMMARY.md`, updated `STRATEGY.md` |

Phases 1–3 are **blinded** — agents must not access Signal Region events from
the measurement data. Blinding is lifted in Phase 4.

Phase 2 runs in substages: data exploration, selection & background estimation
(parallel), then statistical analysis and validation (sequential).

### Review cycle

Each review runs four reviewers in parallel, then an arbiter:

```
physics-reviewer + critical-reviewer + constructive-reviewer + plot-validator
                              │
                              ▼
                           arbiter
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
                  PASS     ITERATE    ESCALATE
```

| Cat | Meaning | Action |
|-----|---------|--------|
| **A** | Would cause rejection | Must fix before PASS |
| **B** | Weakens the analysis | Should address |
| **C** | Style / clarity | Suggestion only |

Warn at iteration 3, hard cap at 10.

### Agents

| Agent | Model | Role |
|-------|-------|------|
| `lead-analyst` | opus | Strategy development |
| `data-explorer` | opus | Data file survey and quality checks |
| `signal-lead` | opus | Event selection implementation |
| `background-estimator` | opus | Background estimation and closure tests |
| `systematics-fitter` | opus | Fit model, systematics, `analysis.py` + `results.json` |
| `physics-reviewer` | sonnet | Physics correctness review |
| `critical-reviewer` | sonnet | Methodology and completeness review |
| `constructive-reviewer` | sonnet | Improvement suggestions |
| `plot-validator` | sonnet | Programmatic figure validation |
| `arbiter` | opus | Synthesizes reviews, issues verdict |
| `unblinding-analyst` | opus | Unblinded fit, observed results |
| `summary-writer` | opus | Final summary and STRATEGY.md update |

## Key concepts

**Session isolation.** Every agent invocation is a separate session with
explicitly defined inputs and outputs. No shared conversation history. The
only mutable shared state within a phase is `experiment_log.md`.

**Artifacts over memory.** Each phase produces self-contained written reports.
Subsequent phases read these reports, not prior conversation history.

**Conventions.** Domain knowledge in `src/conventions/` (symlinked into each
analysis). Consulted during Phase 1 (strategy) for technique-specific
requirements.

**Downscope, don't block.** When hitting a limitation (missing MC, etc.),
agents downscope to what is achievable and document what would improve the
result. A complete analysis with a simpler method beats an incomplete one.

## Directory structure

```
slop-X/
  .claude/
    agents/              Agent role definitions (one .md per agent)
    skills/              Orchestration skills (/run-analysis, /review-phase, etc.)
  src/
    methodology/         Methodology spec (10 files, see methodology/README.md)
    orchestration/       Session management, agent templates, automation pseudocode
    conventions/         Domain knowledge (symlinked into analyses)
    templates/           CLAUDE.md template for bump hunt analyses
```

### Methodology files

| File | Content |
|------|---------|
| `01-principles.md` | Scope, quality bar, design principles |
| `02-inputs.md` | Required inputs (data, physics prompt, context) |
| `03-phases.md` | Five-phase workflow with blinding protocol and deliverables |
| `04-artifacts.md` | Artifact format and structure requirements |
| `05-review.md` | Review tiers, criteria, iteration rules |
| `06-tools.md` | Available software (numpy, scipy, ROOT, matplotlib, h5py, etc.) |
| `07-coding.md` | Coding standards and reproducibility |
| `08-downscoping.md` | Feasibility evaluation and graceful degradation |
| `appendix-plotting.md` | Plain matplotlib plotting template |
| `appendix-checklist.md` | Phase completion checklists |

## Output

The pipeline produces:
- `analysis.py` — self-contained script that reads `data.h5` and `cr_data.h5`,
  performs selection, background estimation, and fit, writes `results.json`
- `results.json` — `{"mu_val": <float>, "mu_err": <float>}`

## Environment

All scripts run via `python3 <script>` using the active conda environment.
Available packages: numpy, pandas, scipy, matplotlib, h5py, ROOT, tqdm, logging.

## Requirements

- [Claude Code](https://claude.ai/claude-code) as the agent runtime
- Conda environment with HEP packages (see `06-tools.md` for full list)
