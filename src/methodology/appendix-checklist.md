## Appendix B: Minimal Artifact Checklist

### Per-phase artifacts

| Phase | Artifact file | Must contain |
|-------|---------------|-------------|
| 1: Strategy | `STRATEGY.md` | Signal/background enumeration, selection approach, systematics categories |
| 2: Execution | `EXPLORATION.md` | Sample inventory, data quality assessment, variable ranking, preselection cutflow |
| 2: Execution | `SELECTION.md` | Selection definition, region definitions, background estimates, closure tests, per-cut distributions |
| 2: Execution | `INFERENCE.md` | Systematic table, fit model, expected + observed results, fit diagnostics |
| 2: Execution | `analysis.py` + `results.json` | Self-contained script, machine-readable results (JSON) |
| 3: Review | `review/` | Review artifacts (physics, critical, constructive, plot-validation, arbiter verdict) |

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
