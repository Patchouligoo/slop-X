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


---
