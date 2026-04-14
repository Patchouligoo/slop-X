## Appendix B: Minimal Artifact Checklist

### Per-phase artifacts

| Phase | Artifact file | Must contain |
|-------|---------------|-------------|
| 1: Strategy | `STRATEGY.md` | Signal/background enumeration, selection approach, systematics categories |
| 1: Strategy | `DATA_SURVEY.md` | Sample inventory, data quality assessment, variable ranking, preselection cutflow |
| 2: Execution | `SELECTION.md` | Selection definition, region definitions, per-cut distributions |
| 2: Execution | `BACKGROUND.md` | Background estimates, closure tests with uncertainties |
| 2: Execution | `INFERENCE.md` | Systematic table, fit model, expected + observed results, fit diagnostics |
| 2: Execution | `analysis.py` + `results.json` | Self-contained script with `--blinded` support, machine-readable expected results (JSON from `--blinded` run) |
| 3: Final Review | `review/` | Review artifacts (physics, critical, constructive, plot-validation, arbiter verdict) |
| 4: Unblinding | `UNBLINDING.md` | Observed mu_val/mu_err, expected vs observed comparison, anomaly assessment, post-fit diagnostics with SR data |
| 4: Unblinding | Updated `results.json` | Observed results replacing expected results |
| 5: Summary | `SUMMARY.md` | Full analysis chain summary, lessons learned, potential improvements |
| 5: Summary | Updated `STRATEGY.md` | Phase 4 and Phase 5 results appended |

### Final results completeness checklist

The final results must satisfy ALL of these. Each is a Category A
review finding if absent:

- [ ] Machine-readable `results/` directory (JSON with spectrum, parameters)
- [ ] Self-contained `analysis.py` script that reproduces results
- [ ] All figures saved as PNG with clear labels and units
- [ ] Experiment log is non-empty
- [ ] All intermediate phase artifacts exist on disk

### Experiment log minimum content

The experiment log must contain entries for at least:
- [ ] Data format discovery (branches, trees, event counts)
- [ ] Key parameter choices and their reasoning
- [ ] Failed approaches and why they were abandoned
- [ ] Any bugs encountered and how they were resolved

---
