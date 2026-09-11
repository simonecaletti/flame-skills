---
name: plot-histograms
description: Turn FLAME .top histogram output into comparison plots — overlay several runs with a ratio panel and a scale-variation band. Use when asked to plot histograms, compare two runs or two branches, make a ratio plot, show scale-variation uncertainty, combine per-seed .top files, or produce a PDF of FLAME results.
---

# Plotting FLAME histograms

FLAME writes histograms as `.top` text files. The pipeline is: combine the per-seed
files into one, then overlay however many combined files you want to compare.

## 1. Combine per-seed files

A production leaves one `.top` per seed in `results/output/top/`. Merge them first —
plotting a single seed is plotting noise.

```bash
./build/bin/combine_histograms --output results/output/histograms_st_2_allseeds.top --files results/output/top/histograms_st_2_xgrid_1_seed_*.top
```

| Flag | Meaning |
| --- | --- |
| `--mode` | `average` (default), `weighted-average`, `sum` |
| `--output` | output file |
| `--files` | input files, each optionally `FILE:WEIGHT` |
| `--digits N` | output precision (default 6) |

`average` is the right mode for seeds of the same run: they are statistically
independent samples of the same integral. `weighted-average` weights by `1/sigma^2`
per bin and is for combining runs of *different* statistics. `sum` is for genuinely
disjoint contributions.

`run_parallel_stages.py` already does this step at the end of stage 2, so a completed
production run has `results/output/histograms_st_2_allseeds.top` ready.

The usage string inside the binary misprints its own name as `./merge`. Ignore it —
`merge` is the *grid* merger, a different program.

## 2. Overlay and compare

```bash
python3 code/scripts/plotting/plot-top-files.py --files A.top B.top --out compare.pdf
```

| Flag | Meaning |
| --- | --- |
| `--files F1 F2 ...` | **required**, one or more `.top` files |
| `--out PDF` | **required**, multi-page output PDF |
| `--ratio-ref N` | index of the reference file for ratios (default `0` = first listed) |
| `--ratio-ylim YMIN YMAX` | ratio panel limits (default `0.5 1.5`) |
| `--weight-index K` | which weight to plot when several exist (default `0`) |
| `--tol` | bin-edge matching tolerance (default `1e-12`) |
| `--dense` | densify ND histograms, helps ordering |

One PDF page per histogram name common to the inputs. 1D pages are a step plot with
error bars, a shaded scale-variation envelope, and a ratio panel underneath. 2D
histograms render as `pcolormesh`. Histograms with more than 2 dimensions are skipped.

### Weights and the scale-variation band

Index `0` is the **central** weight; indices `1..n` are the scale variations requested
by `[scales] scale_variations` in the runcard (`7point`, `5point`, or `none`). The
envelope band is built from all of them, so it only appears if the run actually
requested variations. `--weight-index K` plots variation `K` as the central curve
instead — use it to see one specific variation rather than the envelope.

### Comparing two branches or two code versions

Order matters: `--ratio-ref` picks the denominator. Put the reference first and the
candidate second, which is the default:

```bash
python3 code/scripts/plotting/plot-top-files.py --files baseline.top new.top --out check.pdf --ratio-ylim 0.98 1.02
```

Tighten `--ratio-ylim` for a refactoring that is supposed to change nothing — with the
default `0.5 1.5` a 5 % discrepancy is invisible. For a change that must be *exactly*
neutral, do not plot at all: run the `regression-test` skill, which compares at `1e-14`.

## Bin edges must match

Files are matched histogram-by-histogram by name, and bins by edge value within
`--tol`. Two runs with different binning produce either empty pages or a crash, not a
warning. Binning is fixed in the analysis source (`_histogram_manager.book(...)`), so
comparing across a binning change means re-running, not re-plotting.

## Other comparison scripts in the tree

- `code/scripts/plotting/plot_flame_vs_nnlojet.py` — FLAME vs NNLOJET. **No CLI**: file
  paths, histogram name, axis ranges and labels are module-level constants hardcoded to
  one author's layout. Treat it as a template to copy and edit, not a tool to run.
- `code/scripts/plotting/plot-kh-grids.py` — **dead**. It reads
  `*_status_st_*_xgrid_*.json` grid dumps that the library no longer writes (grid state
  is now `khd_<channel>/*.khd` plus `*.khs`), so it always fails with
  `FileNotFoundError`. Do not suggest it.
