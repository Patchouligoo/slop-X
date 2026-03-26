## Appendix A: Plotting Template

All plotting code must follow this template. This is the reference for any
agent producing figures — whether the executor itself or a dedicated plotting
subagent.

### Base template

```python
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(42)

# --- Single plot ---
fig, ax = plt.subplots(figsize=(10, 10))
# For MxN subplots, scale to keep ratio: 2x2 -> (20, 20), 1x3 -> (30, 10)

# --- Ratio plot ---
# fig, (ax, rax) = plt.subplots(
#     2, 1, figsize=(10, 10),
#     gridspec_kw={"height_ratios": [3, 1]},
#     sharex=True,
# )
# fig.subplots_adjust(hspace=0)  # REQUIRED — no gap between main and ratio

# --- Your plotting code ---

# For histograms: ax.hist(...) or ax.bar(bin_centers, heights, width=bin_widths)
# For 2D histograms: ax.pcolormesh(X, Y, Z, cmap="viridis")
# For error bars: ax.errorbar(x, y, yerr=yerr, fmt="o")
# For data/MC comparisons: overlay on main axes, ratio on rax

# --- Labels (required) ---
ax.set_xlabel(r"$m_{jj}$ [GeV]")
ax.set_ylabel("Events / bin")
ax.legend(fontsize="x-small")

fig.savefig("output.png", bbox_inches="tight", dpi=200)
plt.close(fig)
```

### Rules

- **Figure size is LOCKED at `figsize=(10, 10)`.** Do not use any other
  figure size for single plots. For ratio plots, use `figsize=(10, 10)` with
  `height_ratios=[3, 1]`. For 2x2 subplots, use `figsize=(20, 20)`. The
  rule is: 10 inches per subplot column, 10 inches per subplot row.
  **Any script that uses a custom figsize is a Category A review finding.**
- **No titles.** Never `ax.set_title()`. Use axis labels and legends to
  convey information. Additional info can go into `ax.legend(title="...")`.
- **Axis labels with units.** Always `ax.set_xlabel(...)` and
  `ax.set_ylabel(...)` with units in brackets, e.g. `r"$p_T$ [GeV]"`.
- **Labels on every axes.** In multi-panel figures, every axes must have
  axis labels.
- **Legend font size.** Always pass `fontsize="x-small"` to `ax.legend(...)`.
- **Save as PNG only.** Always `bbox_inches="tight"`, `dpi=200`. No PDF
  output needed.
- **Close figures.** `plt.close(fig)` after saving to prevent memory leaks
  in long scripts.
- **Ratio plot hspace.** `fig.subplots_adjust(hspace=0)` is non-negotiable
  for ratio plots. Any visible gap between the main panel and ratio panel
  is a Category A review finding.
- **Log scale.** Use `ax.set_yscale("log")` when the y-axis range spans
  more than 2 orders of magnitude. Linear scale is appropriate otherwise.
- **Deterministic.** `np.random.seed(42)` if any randomness is involved.
- **Aspect for 2D plots.** For 2D heatmaps with colorbars, use
  `fig.colorbar(im, ax=ax, fraction=0.046, pad=0.04)` or similar to keep
  reasonable proportions.

### Error propagation for derived quantities

When plotting derived quantities (ratios, normalized distributions,
efficiencies), uncertainties must be propagated manually — matplotlib does
not do this automatically.

**Common formulas:**
- **Normalized distribution** `(1/N) dN/dx`: `yerr[i] = sqrt(n[i]) / (N * dx[i])` where `N = sum(n)` and `dx[i]` is the bin width. For Poisson counts, `sqrt(n[i])` is the per-bin uncertainty.
- **Ratio** `R = A/B`: `sigma_R = R * sqrt((sigma_A/A)^2 + (sigma_B/B)^2)` (uncorrelated errors)
- **Efficiency** `e = k/n`: use Clopper-Pearson (binomial) intervals, not Gaussian propagation. `scipy.stats.binom` provides these.
- **Bin-width-normalized** `dN/dx`: `yerr[i] = sqrt(n[i]) / dx[i]`

Always pass `yerr=` explicitly to `ax.errorbar()` or `ax.bar(..., yerr=...)`
for derived quantities. Relying on auto-errors for derived quantities is a
Category A review finding.

### Captions

See Section 4 for caption requirements. Captions must be self-contained:
state what is plotted, identify all curves/markers/bands, and state the key
conclusion. Sparse captions are Category A.

### Subfigures and figure grouping

Group related figures into grids rather than presenting them as separate
figures. Use letter labels `(a)`, `(b)`, etc. with `ax.text(0.05, 0.95,
"(a)", transform=ax.transAxes, fontsize="large", va="top")` in each panel.
Write a single caption describing all sub-panels. This keeps the output
compact and makes comparisons easier for the reader.

**Grid sizing:** Selection cut distributions can be grouped into a 3x3 grid
with a single caption. Related comparisons (e.g., data/background for
multiple variables) should be side-by-side. A 2x2 grid uses
`figsize=(20, 20)`, a 3x3 uses `figsize=(30, 30)` — following the
10-inches-per-subplot rule.

### Correlation and covariance visualizations

Correlations between variables, bins, or systematic sources must be shown
as **matrix heatmaps** (using `ax.pcolormesh` with a diverging colormap
like `RdBu_r`, centered at 0 for correlations). For the correlation
matrix specifically:
- Use `vmin=-1, vmax=1` with a diverging colormap
- Annotate cells with values if the matrix is small enough (< 10x10)
- For large matrices, show the heatmap without annotations but with a
  clear colorbar

### Systematic breakdown plots

When a systematic breakdown shows any single source with relative uncertainty
>100% in a bin, investigate — this typically indicates a bug in the variation
processing or an edge effect in a low-stats bin. Clip or flag such bins rather
than letting them dominate the y-axis scale. If the large variation is genuine
(e.g., a very low-stats bin), document the explanation in the artifact.

### Delegation to plotting subagent

Plotting should be delegated to a dedicated subagent. When
spawning a plotting subagent, the parent agent must include in the prompt:

1. **This entire appendix** (copy the template and rules above into the
   subagent prompt so it has the style reference in context)
2. The data to plot (file paths or serialized arrays)
3. What kind of plot (histogram, ratio, 2D, overlay)
4. Axis labels and ranges
5. Output path

The plotting agent applies this template and produces the figure. It does not
make physics decisions about what to plot or how to interpret the result.

---
