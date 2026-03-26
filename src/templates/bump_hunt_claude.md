# Bump Hunt Analysis

{{task_description}}

---

## Orchestration

Use the `/run-analysis` skill to execute the analysis pipeline. The skill
spawns specialist agents (lead-analyst, data-explorer, signal-lead,
background-estimator, systematics-fitter, critical-reviewer, etc.) to plan,
implement, and review the analysis.

The agents and skills are defined in `.claude/agents/` and `.claude/skills/`.

---

## Execution Model

**You are the orchestrator.** You do NOT write analysis code yourself. You
delegate to subagents. Your context stays small; heavy work happens in
subagent contexts.

**All executor subagents start in plan mode.** When spawning an executor,
instruct it to first produce a plan: what scripts it will write, what figures
it will produce, what the artifact structure will be. The subagent executes
only after the plan is set. This prevents agents from diving into code
without thinking.

**Agent profiles:** Detailed role definitions with domain knowledge, mandatory
checklists, and output formats live in `.claude/agents/*.md`. When spawning an
executor or reviewer, instruct it to read its agent profile first. The profile
contains the deep domain expertise (selection philosophy, fit diagnostics,
closure test criteria, etc.) that makes the agent effective. The agent roster
and phase-to-agent mapping is in `orchestration/agents.md`.

**Anti-patterns:**
- Running straight from strategy to results with no intermediate artifacts
- The orchestrator writing analysis scripts itself
- Accepting reviewer PASS too easily — the arbiter should ITERATE liberally
- Spawning subagents without `model: "opus"` — this silently degrades quality
- Subagents reading files with `cat | sed | head` instead of the Read tool
- Skipping plot-validator in review cycles — it catches errors LLMs miss
- Spawning an executor without pointing it to its `.claude/agents/` profile

**What the orchestrator does NOT do:**
- Read full scripts or data files (subagents do this)
- Debug code (subagents do this)
- Produce figures (subagents do this)
- Write analysis prose (subagents do this)

**What the orchestrator MUST do:**
- **Health monitoring.** Check progress every ~5 minutes for long-running
  subagents. Respawn stalled agents if no progress in >10 minutes.
- Ensure review quality. Do NOT conserve tokens by accepting weak reviews
  or rushing past issues. If a reviewer finds problems, have the work redone
  properly — not minimally patched.

{{model_tiers}}

**Subagent file reading:** Instruct all subagents to use the Read tool to
read files in full (no line limits). Never use `cat`, `sed`, `head`, or
`tail` to read files in chunks — the Read tool handles files of any size
and gives the subagent the complete content.

---

## Methodology

Read relevant sections from `methodology/` as needed:

| Topic | File | When |
|-------|------|------|
| Phase definitions | `methodology/03-phases.md` | Before each phase |
| Artifacts | `methodology/04-artifacts.md` | Producing deliverables |
| Review protocol | `methodology/05-review.md` | Spawning reviewers |
| Tools & paradigms | `methodology/06-tools.md` | Coding phases |
| Coding practices | `methodology/07-coding.md` | Coding phases |
| Downscoping | `methodology/08-downscoping.md` | Resource constraints |
| Plotting | `methodology/appendix-plotting.md` | All figure-producing phases |
| Checklist | `methodology/appendix-checklist.md` | Review phases |

---

## Conventions

Read `conventions/search.md` before developing the strategy and during review.

For every required systematic source listed in the conventions, state
"Will implement" or "Not applicable because [reason]." This enumeration is
binding — reviews check against it. Silent omissions are Category A findings
(must resolve before advancing).

---

## Environment

- Run all scripts via `python3 <script>` using the active conda environment.
- Do NOT use pixi, pip install, or any package manager.
- **Available packages**: {{packages}}, plus Python standard library.
- Do NOT use any package not listed above.

---

## Data Files

{{data_description}}

{{physics_hints}}

---

## Review Protocol

See `methodology/05-review.md` for the full protocol. Key rules:

**Classification:** **(A) Must resolve** — blocks advancement. **(B) Must fix
before PASS** — weakens the analysis. **(C) Suggestion** — applied before
next step, no re-review.

The arbiter must not PASS with unresolved A or B items.

**Plot-validator** runs alongside all other reviewers in parallel. It performs
programmatic (not visual) checks on plotting code and output data. Red flags
from the plot-validator are automatic Category A — the arbiter must not
downgrade them. See `.claude/agents/plot-validator.md` for the protocol.

**Iteration limits:** 4-bot: warn at 3, strong warn at 5, hard cap at 10.
1-bot: warn at 2, escalate after 3. All subagents use `model: "opus"`.

---

## Coding Rules

- **Columnar analysis.** Arrays, not event loops. Selections are boolean masks.
- **Prototype on a slice.** ~1000 events first, full data only for production.
- **KISS / YAGNI.** No CLIs, config systems, or plugin architectures. Write scripts.

Standard logging setup:
```python
import logging
logging.basicConfig(level=logging.INFO, format="%(message)s")
log = logging.getLogger(__name__)
```

See `methodology/07-coding.md` for full coding practices.

---

## Scale-Out Rules

**Always estimate before running at full scale.** Check input size, time a
1000-event slice, extrapolate.

| Estimated time | Action |
|---|---|
| < 2 min | Single-core local — just run it |
| 2-15 min | `ProcessPoolExecutor` or equivalent multicore |
| > 15 min | SLURM: `sbatch --wait` (single) or `--array` (per-file) |

---

## Plotting Rules

- **Figure size:** `figsize=(10, 10)`. Subplots: `figsize=(10*ncols, 10*nrows)`.
- **No titles.** Never `ax.set_title()`. Use axis labels and legends instead.
- **Save as PNG.** `bbox_inches="tight"`, `dpi=200`. Close figures after saving.
- **Clear labels.** All axes must have labels with units where applicable.

See `methodology/appendix-plotting.md` for full plotting standards.

---

## Output Requirements

{{output_format}}

---

## Boundary Rules

{{boundary_rules}}
