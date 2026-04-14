## Automation

> See `agents.md` for the literal prompt templates used in `run_agent` calls.
> See `sessions.md` for the directory layout this script populates.

The following pseudocode illustrates the orchestration logic. It is not a
runnable script — helper functions like `find_latest_artifact`, `extract_decision`,
etc. are orchestrator responsibilities whose implementation depends on the agent
system. The logic and control flow are what matter.

```bash
# --- Configuration ---

max_review_iterations=${MAX_REVIEW_ITER:-10}

# --- Session naming ---

pick_session_name() {
  echo "$(shuf -n1 names_pool.txt)"
}

# --- Review function ---

# 4-bot review: physics + critical + constructive + plot-validator → arbiter.
# Returns 0 on PASS, 1 on max-iterations/escalation.
run_4bot_review() {
  dir=$1
  i=0
  while [ $i -lt $max_review_iterations ]; do
    i=$((i + 1))
    if [ $i -gt 3 ]; then
      echo "WARNING: review iteration $i for $dir"
    fi
    if [ $i -gt 5 ]; then
      echo "STRONG WARNING: review iteration $i for $dir"
    fi

    # Physics, critical, constructive, and plot-validator run in parallel
    run_agent --name "$(pick_session_name)" \
      --output "$dir/review/physics" "physics review" &
    run_agent --name "$(pick_session_name)" \
      --output "$dir/review/critical" "critical review" &
    run_agent --name "$(pick_session_name)" \
      --output "$dir/review/constructive" "constructive review" &
    run_agent --name "$(pick_session_name)" \
      --output "$dir/review/plot-validation" "plot validation" &
    wait

    # Arbiter reads all reviews and the artifact
    run_agent --name "$(pick_session_name)" \
      --output "$dir/review/arbiter" "arbitrate"
    decision=$(extract_decision "$dir/review/arbiter")

    case $decision in
      PASS)
        return 0
        ;;
      ITERATE)
        exec_name=$(pick_session_name)
        write_iteration_inputs "$dir" "$i" "$exec_name"
        run_agent --name "$exec_name" \
          --output "$dir/exec" "iterate v$((i+1))"
        ;;
      ESCALATE)
        echo "ESCALATED: $dir — review could not resolve issues"
        return 1
        ;;
    esac
  done

  echo "ERROR: review reached $max_review_iterations iterations for $dir"
  return 1
}

# --- Main pipeline (5-phase) ---

# Phase 1: Strategy
run_agent --name "$(pick_session_name)" \
  --output "phase1_strategy/exec" "execute strategy (lead-analyst)"
run_agent --name "$(pick_session_name)" \
  --output "phase1_strategy/exec" "survey data (data-explorer)" &
wait
run_4bot_review "phase1_strategy" || exit 1

# Phase 2: Execution
# 2.1-2.2: Selection and background in parallel
run_agent --name "$(pick_session_name)" \
  --output "phase2_execution/exec" "implement selection (signal-lead)" &
run_agent --name "$(pick_session_name)" \
  --output "phase2_execution/exec" "estimate backgrounds (background-estimator)" &
wait

# 2.3-2.4: Fit and final results (sequential — needs selection + background)
run_agent --name "$(pick_session_name)" \
  --output "phase2_execution/exec" \
  "build fit, evaluate systematics, produce analysis.py + results.json (systematics-fitter)"

# Phase 3: Review
run_4bot_review "phase2_execution" || exit 1

# Phase 4: Unblinding
# Blinding is lifted — this agent may access SR events from the measurement data
run_agent --name "$(pick_session_name)" \
  --output "phase4_unblinding/exec" \
  "unblind: run analysis.py on full data including SR, produce observed results (unblinding-analyst)"
run_4bot_review "phase4_unblinding" || exit 1

# Phase 5: Summary
run_agent --name "$(pick_session_name)" \
  --output "phase5_summary/exec" \
  "produce final summary, update STRATEGY.md with Phase 4/5 results (summary-writer)"
run_4bot_review "phase5_summary" || exit 1

# On PASS: verify results.json exists with observed results
if [ -f "results.json" ] && [ -f "SUMMARY.md" ]; then
  echo "Analysis complete."
else
  echo "ERROR: results.json or SUMMARY.md not found after review PASS"
  exit 1
fi
```
