---
name: production-run
description: Run a full FLAME production — parallel seeds over stage-1 grid training and stage-2 production, then merge grids and combine histograms. Use when asked to run a production, launch many seeds, run stage 1 and/or stage 2, train or merge Kakuhen grids, combine per-seed .top files, restart from a given grid iteration, or interpret the results/ directory layout.
---

# Running a FLAME production

A production is two stages. **Stage 1** trains the Kakuhen integration grids: every
seed writes its own grid, and `merge` folds them into one grid that the next iteration
starts from. **Stage 2** is a single iteration of `nseeds` statistically independent
runs off the frozen grid, whose per-seed histograms `combine_histograms` averages.

`run_parallel_stages.py` drives both.

## The driver

```bash
cd code/process/VJ && ./build/bin/run_parallel_stages.py --runcard RUNCARD.run --nseeds 8 --niter 4 --stage 0 --ncpu 8
```

| Flag | Meaning |
| --- | --- |
| `--runcard` | **required**, passed through to `flame_exe -r` |
| `--nseeds` | **required**, number of seeds; seeds run `1..nseeds` |
| `--niter` | stage-1 grid iterations (default 1); ignored for stage 2, which always runs one |
| `--stage` | `1`, `2`, or `0` = both (default `0`) |
| `--ncpu` | max concurrent processes (default: all CPUs) |
| `--bin-prefix` | prefix containing `bin/` (default `./build`) |
| `--message` | text recorded in the run's `README` |

`--bin-prefix` must contain `bin/flame_exe`, `bin/merge` and `bin/combine_histograms`.
The library install symlinks `merge`, `combine_histograms` and the driver itself into
every process `bin/`, so running from a built process directory needs no extra setup.

**Run it from the process directory, not from `bin/`.** All paths inside are relative
to the current working directory: the driver globs `results/output/top/...`, and the
`merge` call has its `-d results` argument commented out, so it resolves `results/`
from cwd.

Each seed is launched as:

```
flame_exe -r RUNCARD -s <stage> -g <xgrid> -i <seed> -m <message>
```

so per-seed reproducibility comes from `-i`. Per-job logs are written as
`stage<S>_xgrid<I>_seed<N>.log` in cwd and then moved into
`results/stage<S>/iteration<I>/log/`.

## Directory layout

```
results/
  stage<S>/iteration<I>/
      khd_<channel>/        Kakuhen grid state (*.khd, *.khs)
      fld/                  multichannel weight files (*.fld)
      log/                  per-seed job logs
  output/
      top/  histograms_st_<S>_xgrid_<I>_seed_<N>.top
      log/  check_limit_channel_<channel>.log
      histograms_st_2_allseeds.top      <- the final result
```

`output_dir` is set in the runcard's `[output]` section, or overridden with
`flame_exe -o`.

## What the driver does between runs

After each **stage-1** iteration:

```bash
merge -s 1 -g <xgrid> --print-grid
```

`merge` finds the previous iteration automatically, reads dimensionality and channel
counts from the `.fld` files, and loops the `Btilde`, `Remnant` and `Regular`
contributions. Its own flags are `-s/--stage` and `-g/--grid-iteration` (both
mandatory), `-d/--directory` (default `./`), `-p/--print-grid`.

After **stage 2**:

```bash
combine_histograms --output results/output/histograms_st_2_allseeds.top --files <every per-seed .top>
```

Flags: `--mode average|weighted-average|sum` (default `average`), `--output`,
`--files` (accepts `FILE:WEIGHT`), `--digits N` (default 6). The driver passes explicit
paths rather than a glob. Its usage string misprints the binary name as `./merge` —
ignore that.

## Running stages separately

Useful when grid training and production happen on different machines or at different
times:

```bash
./build/bin/run_parallel_stages.py --runcard R.run --nseeds 8 --niter 4 --stage 1 --ncpu 8
```

```bash
./build/bin/run_parallel_stages.py --runcard R.run --nseeds 32 --stage 2 --ncpu 8
```

Stage 2 reads the grid left by the last stage-1 iteration. To **restart from a given
iteration**, the grids for iterations `1..I` must already exist under
`results/stage1/iteration<I>/`; `merge` locates the previous one itself. There is no
`--start-iteration` flag, so restarting means either re-running from iteration 1 or
invoking `flame_exe -s 1 -g <I> -i <seed>` and `merge -s 1 -g <I>` by hand for the
iterations you want.

## Gotchas

- **Known bug**: the pre-flight check for `combine_histograms` reads
  `if (args.stage == 2 or args.stage == 2)` (`run_parallel_stages.py:162`) — the
  second test should be `== 0`. With `--stage 0` a missing `combine_histograms` is not
  caught up front and the run fails only at the end of stage 2, after all the compute.
- Stage 2 with no matching `.top` files prints "No input files found" and exits
  successfully without producing `histograms_st_2_allseeds.top`. Check that the file
  exists before trusting a run.
- `--niter` is silently ignored for stage 2.
- `run_parallel_stages.sh` is the obsolete bash version — it still calls `./bin/flame_dy`
  and does no histogram merge. Do not use it.
- For a single quick run, skip all of this: `flame_exe -r R.run --all-in-one -n N -i 1`
  (see the `build` skill).

## Plotting the result

`results/output/histograms_st_2_allseeds.top` is the input to the `plot-histograms`
skill.
