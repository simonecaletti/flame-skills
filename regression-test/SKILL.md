---
name: regression-test
description: Run and interpret the FLAME regression test suite in code/regression_test — cross sections and histograms compared against a stored baseline. Use when asked to run regression tests, check that a refactoring did not change numbers, create or regenerate a baseline, unpack a baseline tarball, investigate a regression failure or tolerance mismatch, or run the staged (stage-1/merge/stage-2) path instead of all-in-one.
---

# FLAME regression tests

The suite builds the processes, runs each runcard, and compares cross sections and
histograms against a stored baseline at very tight tolerances. It is the check to
run after any refactoring that is supposed to be numerically neutral.

## Baselines are tarballs

`baseline/` is **not** in the repo, and is never committed. Baselines are exchanged
as tarballs sitting in `code/regression_test/`, named
`baseline_<label>_<commit-hash>.tar` — the short hash of the commit the baseline was
generated at, e.g. `baseline_simone_3c80f33.tar`, `baseline_OL_4ee8128.tar`. The
hash is the whole point: it says which code state the numbers came from.

### Unpacking one to run against

Unpack the tarball for the commit you want to compare against and rename the
unpacked directory to exactly `baseline/`:

```bash
cd code/regression_test && tar xf baseline_simone_3c80f33.tar && mv <unpacked-dir> baseline
```

Without it, `--test` exits with "No baseline found."

### Creating one

After `--create-baseline` finishes, always tar up the fresh `baseline/` directory
with the current commit hash in the name, so it can be handed to someone else and
so it stays identifiable later:

```bash
cd code/regression_test && tar cf "baseline_$(git rev-parse --short HEAD).tar" baseline/
```

Add a short label before the hash when the baseline is specific to a configuration
rather than to plain HEAD (e.g. `baseline_OL_<hash>.tar` for an OpenLoops build).
A baseline directory that was never tarred is lost the moment `--clean` runs.

## Running

```bash
cd code/regression_test && python3 run_regression.py --test --verbose
```

Exactly one mode is required: `--create-baseline`, `--test`, or `--clean`.

| Flag | Meaning |
| --- | --- |
| `--test` | run and compare against `baseline/`, writing to `current/` |
| `--create-baseline` | regenerate references (prompts before overwriting) |
| `--clean` | wipe all test directories |
| `--process NAME` | restrict to one process — only `DY`, `VJ`, `ggH` are accepted |
| `--runcard NAME` | restrict to one runcard, e.g. `DY_Z_gamma` |
| `-v, --verbose` | per-comparison output |
| `-j, --parallel N` | run up to N tests in parallel (default 1) |
| `--mode all-in-one\|staged` | default `all-in-one`; `staged` exercises stage 1 → merge → stage 2 |
| `--ci` | non-interactive, auto-overwrites the baseline |

The runner shells out to `code/scripts/build_process.sh` itself — no separate build
step needed. Runcards live in `runcards/` and are named `<PROCESS>_<testname>.run`;
the process is taken from the prefix. Note `dijet_minimal.run` exists as a runcard
even though `--process` will not accept `dijet`; reach it with
`--runcard dijet_minimal`.

Runcards with no corresponding baseline directory are skipped with a warning, not
failed — check the skip list before declaring a green run.

## Tolerances

From `compare_results.py`: cross-section values `1e-14`, MC errors `1e-12`,
histogram bins `1e-12` — all relative. These are bit-level-ish; a genuine
refactoring that reorders floating-point operations will trip them. A diff of
`~1e-15` is noise, `~1e-6` or larger is a real physics change.

## When a test fails

1. Confirm the baseline tarball matches the commit you meant to compare against —
   a mismatched baseline is the most common false alarm.
2. Re-run the single failing runcard with `--runcard <name> --verbose` to see the
   baseline vs current values.
3. Rebuild with `DISABLE_EQUIV`, `DISABLE_CACHE_EQUIV`, `DISABLE_CACHE_PHSP_REAL`
   set on the library configure. If the number changes, the bug is in the
   Born-equivalence or caching layer rather than the matrix element.
4. Compare `current/<runcard>/result.txt` against `baseline/<runcard>/result.txt`
   directly; `compare_results.py` can be run standalone with `--tolerance` /
   `--hist-tolerance` overrides to gauge how far off the result is.

## CI

GitLab CI runs `--test --ci` but is gated off — `CI_ENABLED: "false"` in
`code/.gitlab-ci.yml`. Do not assume CI covered a change.
