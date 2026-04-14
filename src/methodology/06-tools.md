## 6. Tools and Paradigms

The agent uses standard scientific Python software. No bespoke analysis
framework is required, but the following tools and paradigms are preferred.
This section is maintained by the analysis team and reflects operational
knowledge about what works well in practice.

### 6.1 Preferred Tools

| Capability | Tool | Notes |
|-----------|------|-------|
| Data I/O | pandas, h5py | `pd.read_hdf()` for HDF5 files. `h5py` for low-level access when needed. |
| Array operations | numpy, pandas | Columnar analysis — no event loops. numpy for numerical arrays, pandas for labeled data. |
| Histogramming | numpy | `np.histogram`, `np.histogram2d`, `np.histogramdd`. Use `np.searchsorted` for bin assignment. |
| Fitting / optimization | scipy.optimize, ROOT | `curve_fit` for simple fits, `minimize` for general optimization. ROOT's `TF1.Fit()` and `RooFit` for complex likelihood fits and signal+background models. |
| Statistical tests | scipy.stats, ROOT | Hypothesis tests, p-values, confidence intervals. ROOT's `RooStats` for CLs limits, profile likelihood, and hypothesis testing. |
| Plotting | matplotlib | Plain matplotlib with clear labels, legends, and axis annotations. No experiment-specific styling packages. See Appendix A for the plotting template. |
| Progress bars | tqdm | For long-running loops. `from tqdm import tqdm`. |
| Logging | logging | Python `logging` module. No bare `print()`. See Section 7 for setup. |

**Tools NOT available** (do not use):
- `uproot`, `awkward-array` — data is HDF5, not ROOT
- `pyhf`, `cabinetry`, `zfit` — use scipy or ROOT for fitting
- `mplhep` — use plain matplotlib
- `coffea`, `fastjet` — not applicable
- `rich` — not in environment
- `pixi` — use `python3` directly
- `xgboost`, `optuna`, `scikit-learn` — not in environment

**Tiered tagging and classification.** When building taggers or classifiers
(b-tagging, tau-ID, quark/gluon, etc.), follow a tiered approach:

1. **Cut-based (cross-check).** Always build a simple cut-based version first
   using the most discriminating variables. This serves as the baseline and
   independent cross-check.
2. **Multivariate (if justified).** More complex methods are warranted only
   when the input space benefits and sufficient data is available for
   validation. For many analyses, cut-based methods are sufficient.

### 6.2 Paradigms

**Prototype on a slice, scale up when it works.** Never run on the full
dataset first. Every new script, selection, or processing step should be
developed and validated on a small subset (~1000 events or a single file)
before scaling to the full sample. This applies at every phase:
- **Exploration:** Load a small slice, check columns and distributions. Do
  not process gigabytes of data to "see what's there."
- **Selection development:** Optimize cuts on a small slice. Only run the
  full cutflow once the logic is validated.
- **Fit development:** Build and test the fit on a few bins / one region
  before scaling to the full model.

The pattern is: get the code right on a small sample where iteration is
cheap (seconds, not hours), then run once at scale. If a step takes more
than a few minutes, ask whether a subset would answer the same question.
The full dataset is for production runs, not for debugging.

**Read the API before working around it.** When a tool or library behaves
unexpectedly, the first action is ALWAYS to read the function's docstring or
documentation — not to hack around the behavior. Most "unexpected" behavior
is a documented feature with a documented parameter to control it.
- Before calling a function with workarounds, run `help(function)` or read
  the source to check if there's a kwarg that does what you want.
- Before writing code to undo a library's default behavior, check if there's
  a configuration option or style parameter.

The 30 seconds spent reading a docstring saves minutes of debugging
cargo-culted workarounds.

**Columnar analysis.** Operate on arrays of events, not event-by-event loops.
Selections are boolean masks applied to arrays. This is faster, more readable,
and less error-prone than loop-based code.

**Immutable cuts.** Express selections as a sequence of named boolean masks.
Never modify the underlying arrays — apply masks to produce filtered views.
This makes cutflows trivial (count `True` values at each stage) and cuts
composable (AND masks for combined selections).

**Prefer ROOT for fitting.** For signal+background fits, likelihood fits,
and any fit involving PDFs, use ROOT's RooFit rather than scipy. RooFit
handles extended likelihood fits, parameter constraints, and error
propagation correctly out of the box. Use scipy only for trivial curve
fits where RooFit would be overkill.

Minimal RooFit pattern:
```python
import ROOT
import numpy as np

# 1. Observable and parameters
obs = ROOT.RooRealVar("obs", "obs", x_lo, x_hi)
mean = ROOT.RooRealVar("mean", "mean", init_val, lo, hi)
sigma = ROOT.RooRealVar("sigma", "sigma", init_val, lo, hi)

# 2. Build PDF
pdf = ROOT.RooGaussian("pdf", "pdf", obs, mean, sigma)

# 3. Binned data from numpy histogram
h = ROOT.TH1D("h", "h", n_bins, bins.astype(np.float64))
for i in range(n_bins):
    h.SetBinContent(i + 1, values[i])
    h.SetBinError(i + 1, errors[i])
data = ROOT.RooDataHist("data", "data", ROOT.RooArgList(obs), h)

# 4. Fit and extract results
pdf.fitTo(data, PrintLevel=-1)
fitted_val = mean.getVal()
fitted_err = mean.getError()
```

For signal+background models, use `RooAddPdf` with extended terms:
```python
mu = ROOT.RooRealVar("mu", "mu", 0, -1e6, 1e6)
B = ROOT.RooRealVar("B", "B", n_total, 0, 1e9)
model = ROOT.RooAddPdf("model", "model",
    ROOT.RooArgList(sig_pdf, bkg_pdf), ROOT.RooArgList(mu, B))
model.fitTo(data, Extended=True, PrintLevel=-1)
mu_val, mu_err = mu.getVal(), mu.getError()
```

**Fit reproducibility.** A human must be able to re-run every fit in the
analysis. Each fit should have its own script (e.g., `python3 fit.py`,
`python3 compute_limits.py`). The input data, fit script, and results
are stored together. A human opening the analysis should be able to run
the script and reproduce the numbers without reading any agent conversation
history.

**Plots are evidence.** Every claim in an artifact should have a corresponding
figure or table. Plots are not decoration — they are the primary evidence that
the analysis is correct. Label axes with units. Include ratio panels for
data/MC comparisons. Use consistent styling throughout.

**Reproducibility by default.** Pin random seeds. Record software versions in
artifact code-reference sections. Scripts should be re-runnable from a clean
state and produce identical outputs.

**Systematic variation naming.** Use the convention `{source}_up` /
`{source}_down` for systematic variation labels or histogram suffixes (e.g.,
`JES_up`, `JES_down`).

**Binning.** Start with uniform binning during exploration. Before fitting,
rebin to ensure adequate statistics — no bin should have fewer than ~5 expected
events (summed over all processes) to avoid fit instabilities. Variable binning
is fine when physically motivated (e.g., finer bins near a mass peak, coarser
in tails).

### 6.3 Scale-Out

Before running any processing script at full scale, the agent must estimate
the resource requirements and choose the appropriate execution mode. Do not
default to single-core local execution — estimate first, then decide.

#### Estimation step

Before running a script on the full dataset, check:
1. **Input size:** `ls -lh` the input files, sum total bytes.
2. **Per-event cost:** Time the script on a 1000-event slice. Extrapolate to
   full dataset: `(total_events / 1000) * slice_time`.
3. **Memory:** Check peak memory on the slice. Multiply by the chunk factor
   if loading in chunks, or by the full dataset factor if loading all at once.

This takes seconds and prevents the agent from sitting on a login node for
an hour processing what could have been a 2-minute SLURM job.

#### Decision thresholds

| Estimated wall time | Input size | Execution mode |
|---------------------|-----------|----------------|
| < 2 minutes | < 1 GB | **Single-core local.** Just run it. |
| 2-15 minutes | 1-10 GB | **Multicore local.** `ProcessPoolExecutor` or equivalent. |
| > 15 minutes | > 10 GB | **SLURM.** `sbatch --wait` or array jobs. |

These are guidelines, not hard rules. If the cluster is idle, SLURM may be
faster even for small tasks. If the task is embarrassingly parallel across
files, prefer SLURM array jobs over multicore local.

#### Pattern 1: Multicore local

For tasks that fit on a single node but benefit from parallelism:

```python
from concurrent.futures import ProcessPoolExecutor

def process_file(path):
    """Process one HDF5 file. Returns a dict of results."""
    import pandas as pd, numpy as np
    df = pd.read_hdf(path)
    # ... processing logic ...
    return result

files = ["file1.h5", "file2.h5", "file3.h5"]
with ProcessPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(process_file, files))
# Merge results
```

Use `os.cpu_count()` or SLURM's `$SLURM_CPUS_PER_TASK` to set `max_workers`.

#### Pattern 2: SLURM single job

For a single script that needs more resources than the login node allows.
The `--wait` flag makes `sbatch` block until done:

```bash
#!/bin/bash
#SBATCH -p shared
#SBATCH -t 01:00:00
#SBATCH -c 4
#SBATCH --mem=8G
#SBATCH -A <account>
#SBATCH --requeue
#SBATCH -o .slurm_%j.out

cd /path/to/analysis
python3 my_script.py
```

Submit and wait: `sbatch --wait job.sh`

#### Pattern 3: SLURM array jobs

For processing multiple files in parallel — each file gets its own job:

```bash
#!/bin/bash
#SBATCH -p shared
#SBATCH -t 00:30:00
#SBATCH -c 1
#SBATCH --mem=4G
#SBATCH -A <account>
#SBATCH --array=0-5
#SBATCH --requeue
#SBATCH -o .slurm_%A_%a.out

cd /path/to/analysis
python3 process_one_file.py --file-index $SLURM_ARRAY_TASK_ID
```

The script reads `$SLURM_ARRAY_TASK_ID` to pick its input file from a list.
Submit and wait: `sbatch --wait array_job.sh`

#### Rules

- **Always estimate before running.** The estimation step is not optional.
  Log the estimate: `log.info("Estimated wall time: %.0f min for %.1f GB",
  est_minutes, total_gb)`.
- **Never wait > 15 minutes on a login node** when SLURM is available.
- **Prefer the simplest pattern that works.** Single-core < multicore <
  SLURM single < SLURM array. Don't use SLURM when local execution suffices.
- **Log the execution mode.** When a script runs, log whether it used
  single-core, multicore, or SLURM, and how long it took.

---
