# Orchestration Guide

> See `agents.md` for the agent prompt templates that populate this layout.

How to execute the methodology specification using Claude Code or any
multi-session LLM agent system.

## Core Principle: Session Isolation

Every agent invocation — execution, review, arbitration — is a **separate,
isolated session** with explicitly defined inputs and outputs. No shared
conversation history, no shared memory, no implicit state. Each session
reads files, writes files, and exits. The files are the interface.

```
┌─────────────┐     ┌──────────┐     ┌──────────────┐
│   inputs/   │────►│  agent   │────►│   outputs/   │
│  (read-only)│     │ session  │     │  (new files) │
└─────────────┘     └──────────┘     └──────────────┘
                         │
                         ▼
                    session.log
                (full conversation transcript)
```

**Exception: the experiment log.** Each phase has an `experiment_log.md` that
persists across executor sessions within that phase. Every executor session
reads the existing log and appends to it. This is the only mutable shared state
within a phase — it prevents agents from re-trying failed approaches and gives
humans visibility into decision-making.

## Agent Session Identity

Every agent session is assigned a **session name** — a random human first name
(e.g., "Gerald", "Margaret", "Tomoko"). The orchestrator draws from a pool of
names and never reuses a name within an analysis run. This serves two purposes:

1. **Traceability.** Every file produced by a session includes the session name
   and timestamp. When reading artifacts, agents and humans can trace who
   produced what and when.

2. **No clobbering.** Iteration produces new files rather than overwriting
   previous versions.

**Naming convention for handoff files:**
```
{ARTIFACT}_{session_name}_{YYYY-MM-DD}_{HH-MM}.md
```

Examples:
- `STRATEGY_gerald_2026-03-13_14-30.md`
- `STRATEGY_CRITICAL_REVIEW_florence_2026-03-13_15-00.md`
- `STRATEGY_ARBITER_hiroshi_2026-03-13_15-30.md`

The orchestrator tells each agent its assigned session name in the input
prompt. The agent uses this name when naming its output files. Downstream
agents discover the current artifact by finding the most recent file matching
the artifact type pattern, sorted by timestamp.

## Directory Layout

```
analysis_name/
  CLAUDE.md
  conventions/
  methodology/ → (symlink)
  orchestration/ → (symlink)
  experiment_log.md

  phase1_strategy/
    experiment_log.md
    scripts/
    figures/
    exec/
      STRATEGY_gerald_2026-03-13_14-30.md
      DATA_SURVEY_alice_2026-03-13_14-45.md
    review/
      physics/
        STRATEGY_PHYSICS_REVIEW_florence_2026-03-13_15-00.md
      critical/
        STRATEGY_CRITICAL_REVIEW_bob_2026-03-13_15-00.md
      constructive/
        STRATEGY_CONSTRUCTIVE_REVIEW_tomoko_2026-03-13_15-00.md
      plot-validation/
        STRATEGY_PLOT_VALIDATION_carol_2026-03-13_15-00.md
      arbiter/
        STRATEGY_ARBITER_hiroshi_2026-03-13_15-30.md

  phase2_execution/
    experiment_log.md
    sensitivity_log.md
    scripts/
    figures/
    exec/
      SELECTION_david_2026-03-14_10-00.md
      BACKGROUND_eva_2026-03-14_10-00.md
      INFERENCE_frank_2026-03-14_12-00.md
    review/
      physics/
      critical/
      constructive/
      plot-validation/
      arbiter/

  review/                              # Phase 3: top-level review directory
    physics/
    critical/
    constructive/
    plot-validation/
    arbiter/

  phase4_unblinding/
    experiment_log.md
    scripts/
    figures/
    exec/
      UNBLINDING_name_timestamp.md
    review/
      physics/
      critical/
      constructive/
      plot-validation/
      arbiter/

  phase5_summary/
    experiment_log.md
    exec/
      SUMMARY_name_timestamp.md
    review/
      physics/
      critical/
      constructive/
      plot-validation/
      arbiter/

  analysis.py                          # Final deliverable (from Phase 2)
  results.json                         # Final deliverable (observed, from Phase 4)
```
