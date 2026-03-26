## 7. Coding Practices

Analysis code is exploratory by nature, but it must be correct and
reproducible. The engineering bar is **"would this pass review from a physicist
colleague"** — not enterprise software standards.

### 7.1 Code Quality

**Code style:**
- **KISS.** Obvious numpy/pandas operations over clever metaprogramming. A
  physicist reading the code should be able to follow the analysis flow.
- **DRY.** If multiple steps share the same logic, factor it out.
  But do not prematurely abstract for hypothetical future needs.
- **YAGNI.** Do not build CLIs, config systems, or plugin architectures. Write
  scripts. Refactor when (not before) reuse is actually needed.

**Logging, not printing.** All analysis scripts must use Python's `logging`
module — never bare `print()` statements. Standard setup:

```python
import logging
logging.basicConfig(level=logging.INFO, format="%(message)s")
log = logging.getLogger(__name__)
```

**What NOT to do:**
- Do not write unit tests for every function
- Do not create mock data fixtures when real data is available
- Do not add type annotations to exploratory scripts
- Do not write docstrings for functions that run once
- Do not build frameworks when scripts work
- Do not use dependency injection, abstract base classes, or enterprise patterns

### 7.2 Testing

Testing effort should focus on **structural bugs** — errors in the plumbing
that silently propagate through everything downstream. A bug in the final fit
is cheap to fix (re-run the fit). But cutting on the wrong column, or applying
a weight meant for signal to background — these are catastrophic because they
require re-running the entire analysis and are hard to track down.

**Always:** One **smoke test** per phase — does the full pipeline run on ~100
events without crashing? This catches import errors, broken paths, shape
mismatches, and API changes. Fast to run, high value.

**Always:** One **integration test** for the processing chain — does it produce
output files with the expected structure? Not checking physics values — checking
that the machinery works (correct number of bins, files exist, no NaN yields).

**Focus on structural correctness:**
- Test that variable names map to the right physical quantities
- Test that cut inversions actually invert (CR selection is complement of SR)
- Test that systematic variations go in the expected direction
- Test that event counts are monotonically decreasing through the cutflow

These structural tests are cheap to write and catch the bugs that are most
expensive to debug later.

**Never:** Full test suites, 100% coverage targets, TDD. The analysis result
is the product, not the code. The physics validation (closure tests, signal
injection, post-fit diagnostics) IS the test suite for correctness.

### 7.3 Reproducibility

The analysis must be reproducible by a human who has never seen the code.
Scripts should be self-contained and runnable with `python3 script.py`.

**Script organization:**
- Each major step gets its own script (e.g., `apply_selection.py`,
  `run_fit.py`, `compute_limits.py`, `make_plots.py`).
- Scripts read inputs from agreed paths and write outputs to agreed paths.
  The interface between scripts is the filesystem, not function calls.
- Scripts should be idempotent — running them twice produces the same output.
  Use deterministic seeds and write outputs to fixed paths.
- Task names and script names should be human-readable (`fit.py`, not
  `step4b.py`).

**Script decomposition.** If a script's estimated runtime (extrapolated from
a timing slice) exceeds ~5 minutes, split it into stages with intermediate
outputs saved to disk. Example: separate `build_response.py` (construct the
response matrix) from `unfold.py` (run the unfolding) from `bootstrap.py`
(replicas for stat uncertainties). This makes debugging faster, enables
partial re-runs, and keeps individual tasks within the scale-out thresholds.

### 7.4 Code Reuse

**Within an analysis:** Reusable patterns emerge naturally (data loading,
standard plots, workspace building). When a pattern is used 3+ times, factor
it into a shared utility in the analysis's `scripts/common/` directory. Do not
anticipate reuse — wait until it happens.

### 7.5 Debug and Scratch Code

Debug scripts and throwaway experiments are a normal part of analysis
development, but they must not contaminate the production pipeline:

- **Prefix debug scripts with `debug_`** (e.g., `debug_check_weights.py`,
  `debug_plot_comparison.py`). This makes them visually distinct and
  greppable.
- **Or place them in a `scratch/` directory** within the phase directory.
- **Clean up before review.** Before submitting for review, either delete
  debug scripts that are no longer needed or move them to `scratch/` with
  a note in the experiment log about what they tested.

---
