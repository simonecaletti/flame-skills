---
name: build
description: Build the FLAME library and a process executable, and launch flame_exe runs. Use when asked to build, compile, rebuild, or configure FLAME or one of its processes (DY, VJ, ggH, dijet, diboson, epemjj, hvq), when a CMake/link error appears in a FLAME build, when choosing CMake options (OpenLoops, LHAPDF, hoppet, kakuhen, DISABLE_EQUIV, DISABLE_CACHE_*, VERBOSE_LEVEL, DEBUG_PHASESPACE_GENERATE), when running flame_exe with a runcard, or when setting up parallel/staged production runs on a cluster.
---

# Building and running FLAME

## Two artefacts, one script

FLAME builds in two steps: `code/lib` produces the shared `flamelib` and installs a
`FLAMElib` CMake package; each `code/process/<PROC>` builds a `flame_exe` that links
against that installed package. `code/scripts/build_process.sh` does both.

```bash
code/scripts/build_process.sh --proc-src code/process/DY --jobs 8
```

Process directory names are case-sensitive: `DY`, `VJ`, `ggH`, `dijet`, `diboson`,
`epemjj`, `hvq`. `--proc-src` defaults to the current working directory, so from
inside `code/process/dijet` a bare `../../scripts/build_process.sh` works.

Before a first build ever, the submodules must exist:

```bash
git submodule init && git submodule update
```

### build_process.sh flags

| Flag | Meaning |
| --- | --- |
| `--proc-src DIR` | process source dir (default: cwd; must contain `CMakeLists.txt`) |
| `--lib-build DIR` | library build dir (default `code/lib/build`) |
| `--lib-prefix DIR` | library install prefix (default: the library build dir) |
| `--proc-build DIR` | process build dir (default `PROC_SRC/build`) |
| `--proc-prefix DIR` | process install prefix (default: the process build dir) |
| `--lib-openloops` | OpenLoops in the library only |
| `--proc-openloops` | OpenLoops in process **and** library (implies `--lib-openloops`) |
| `-j, --jobs N` | parallel jobs (default 1 — always pass this) |

Relative paths are resolved against the invocation directory, not the script.
The script locates the installed package by globbing `*/lib*/cmake/FLAMElib` under
the library prefix (the `lib` vs `lib64` split is why); "FLAMElib CMake package was
not found" means the install step did not land where the prefix says.

## Library-only build

When only `code/lib` changed and no executable is needed:

```bash
cmake -S code/lib -B code/lib/build -DCMAKE_INSTALL_PREFIX=code/lib/build && cmake --build code/lib/build -j && cmake --install code/lib/build
```

## CMake options

Library (`code/lib`):

- `ENABLE_OPENLOOPS` — OFF by default.
- `ENABLE_LHAPDF` / `ENABLE_HOPPET` / `ENABLE_KAKUHEN` — ON by default. LHAPDF is a
  hard error if requested and missing; pass
  `-DLHAPDF_CONFIG_EXECUTABLE=$(command -v lhapdf-config)`.
- `ENABLE_TESTS` — doctest unit tests, OFF by default.
- `DEBUG_PHASESPACE_GENERATE=0|1|2`, `VERBOSE_LEVEL` — diagnostics.
- `DISABLE_EQUIV`, `DISABLE_CACHE_EQUIV`, `DISABLE_CACHE_PHSP_REAL` — turn off the
  Born-equivalence and caching layers so every ME call is recomputed from scratch.

Process side: `ENABLE_OPENLOOPS_PROC`, `ENABLE_FASTJET`,
`FLAME_LIB_MODE` (`AUTO`/`INSTALLED`/`BUNDLED` — the script forces `INSTALLED`).

**When a result looks wrong, rebuild with the three `DISABLE_*` switches first.**
`build_process.sh` does not forward them, so configure the library by hand with
them set, then point the process build at that prefix.

## Running

```bash
./build/bin/flame_exe -r runcard.run --all-in-one -n 100000 -i 1
```

Flags: `-r/--run` runcard, `-s/--stage`, `-i/--iseed`, `-n/--ncall`,
`-g/--grid-iteration`, `-a/--analysis`, `-o/--output`, `-m/--message`,
`-w/--write` (dumps a fully commented `default.run` template — the root
`default.run` is that dump and is the reference for the runcard format).
`--all-in-one` runs stage 1 then stage 2 in one process and is incompatible with `-s`.

Runcard sections: `[beam] [particles] [ew_couplings] [qcd] [scales] [selectors]
[run_options] [output] [nlo_mode] [User]`. `[run_options]` carries `stage` (1–4),
`random_seed`, `ncall1..4`, `grid_iteration`. `[nlo_mode]` selects
`perturbative_order`, `nlo_contribution` (`standard`/`p2b`/`sector_function`) and
the individual singular contributions. `[User]` is per-process, parsed by that
process's `user_section_*.hh`.

Parallel production:

```bash
code/lib/workflow/run_parallel_stages.py --runcard R --nseeds N --niter I --stage S --ncpu C
```

It fans out seeds and then calls the `merge` and `combine_histograms` executables
that the library build installs into each process `bin/`.

## Cluster

Do **not** build on the Asteria SLURM login node. Use
`code/misc/build_flame_asteria.sbatch`, or read `code/misc/cluster_compilation_info`.
`code/misc/laptop_compilation_info` has the manual local recipe.

## Unit tests

```bash
cmake -S code/lib -B code/lib/build -DENABLE_TESTS=ON && cmake --build code/lib/build -j && ctest --test-dir code/lib/build
```

One test: `ctest --test-dir code/lib/build -R test_phase_space`, or run the binary
from `code/lib/build/bin/test/`. `test_phase_space` runs real MC integrations with a
600 s timeout; every other test is capped at 30 s.

## Scope

`code/` is the only live tree. Never build, edit, or cite
`code_DEPRECATED_DONOTUSE/` or `toy-code/`.
