# Drug release curve fitting

I built a tool for comparing drug release curves because I wanted to know whether a model really explained the data, or just drew a convincing line through it.

I give it a table of time and the fraction of drug released. It fits zero-order, first-order, Higuchi, Korsmeyer-Peppas and Weibull curves, checks predictions against points left out of fitting, and shows how uncertain the fitted parameters are. It warns when a model is outside its useful range or when the points cannot distinguish two models.

**[Read the saved example report](reports/ketoprofen/report.md)** without installing anything. It includes plots and the limitations of the source data. For an offline copy, download [report.html](reports/ketoprofen/report.html) and open it in a browser; the plots are embedded.

## Try it

I use Python with [uv](https://docs.astral.sh/uv/getting-started/installation/) to install the locked dependencies:

```bash
git clone https://github.com/jackaI1/drug-release-kinetics-fit.git
cd drug-release-kinetics-fit
uv sync --frozen --extra dev
uv run release-fit data/ketoprofen_f15.csv \
  --metadata data/ketoprofen_f15.json \
  --output reports/ketoprofen --bootstrap 500 --seed 2026
```

Open `reports/ketoprofen/report.html`, or read `report.md` on GitHub. `fit.json` saves the parameters, confidence intervals, exact training and held-out points, predictions and input hashes for further analysis.

For my own curve, I use a CSV headed `time,fraction`, with increasing times and released fractions between zero and one. I copy the example metadata JSON, replace the citation, time unit, extraction method, error description and notes, then pass both filenames to the same command. The error half-width is in fraction units. I keep the known total drug load as the denominator, rather than scaling the last point to complete release. [The method notes](docs/methods.md) explain the input checks and equations.

## What the example tells me

I used a published ketoprofen pellet release table. The full-curve comparison below comes directly from:

```bash
uv run python scripts/summary.py
```

Observations: 12; bootstrap draws per fit: 500

| Model | Held-out error (percentage points) | AICc |
|---|---:|---:|
| zero order | 7.10 | -48.24 |
| first order | 12.06 | -40.04 |
| higuchi | 17.02 | -33.64 |
| weibull | 5.99 | -67.20 |

A smaller held-out error means better predictions at untrained sampling times. AICc is a score that balances closeness of fit against the number of adjustable parameters; lower is better on the same points. Here Weibull scores best numerically. The held-out set is small, though, and the report still flags unresolved comparisons. I would not treat the table as proof of a release mechanism.

Korsmeyer-Peppas is in the report's separate early-release comparison. I only fit it through the first 60% of release, and compare all models on those same points. I do not compare its early-release score against another model's full-curve score.

## Where the data comes from

I transcribed the time and release columns from the second table labelled Table 4, the F15 kinetics table, in [Ghosh, Mittal, Gaur and Sahoo, *Formulation and evaluation of a pH dependent multi-particulate colon targeted drug delivery system using powder layering technology of ketoprofen* (2020)](https://ijpsr.com/bft-article/formulation-and-evaluation-of-a-ph-dependent-multi-particulate-colon-targeted-drug-delivery-system-using-powder-layering-technology-of-ketoprofen/?view=fulltext). The CSV converts the reported percentages to fractions. This is public experimental data, with no synthetic substitute.

I did not digitize a graph. The metadata states that no graph-reading error was introduced, and gives a rounding allowance of half the displayed precision. That allowance is used in resampling; it is not a measured experimental error. Time uncertainty and raw replicate errors were unavailable. The source also contains a conflicting value in its comparative table and changes the surrounding medium during release. I preserve the selected table and show those caveats in every generated report.

I used [Zhang et al., *DDSolver* (2010), Table I](https://pmc.ncbi.nlm.nih.gov/articles/PMC2895453/) for model definitions and [Heredia et al., *Comparative statistical analysis of the release kinetics models* (2022)](https://pmc.ncbi.nlm.nih.gov/articles/PMC8912140/) for discussion of model comparison and the early-release restriction. These are method references, not extra experimental datasets. The bundled data and derived reports retain the source's [CC BY-NC-SA 3.0 license](https://creativecommons.org/licenses/by-nc-sa/3.0/); my code is MIT licensed.

## How I handle uncertainty

I fit the original released fractions with nonlinear least squares, which adjusts parameters to minimize squared deviations. A fixed split leaves out sampling times before fitting. I then refit all points within each comparison range for the reported parameters and AICc.

Bootstrap confidence intervals come from repeatedly resampling fitted residuals, adding the stated extraction or rounding allowance, and refitting. The report shows conditional 95% intervals and successful resample counts. It also flags close AICc scores and unresolved held-out errors. Cumulative measurements can have correlated errors, so these intervals are conditional descriptions of this curve, not uncertainty across independent experiments.

Higuchi's assumptions about diffusion and the surrounding medium need independent evidence. Predictions beyond complete release, shape parameters at numerical bounds, too few early points and inappropriate baselines are flagged or rejected. None of these fits establishes a physical mechanism or predicts patient outcomes. I explain the formulas, thresholds and limitations in [docs/methods.md](docs/methods.md).

## Code and checks

The package separates model equations, comparison and resampling, input provenance, and report generation under `src/drug_release/`. It runs offline after installation. I check known synthetic parameters independently for every model, noisy parameter recovery, the range restriction, held-out calculations, information criteria, bootstrap reproducibility and the command-line workflow.

```bash
uv run pytest -q
uv run ruff check src tests scripts
uv run ruff format --check src tests scripts
uv build
```

[CI](https://github.com/jackaI1/drug-release-kinetics-fit/actions) runs these checks on the supported Python versions and regenerates the example as a downloadable artifact. The dependency lockfile makes the setup repeatable.
