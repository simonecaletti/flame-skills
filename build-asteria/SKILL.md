---
name: build-asteria
description: Build FLAME and a process on the Asteria SLURM cluster, interactively or via sbatch. Use when asked to build FLAME on Asteria or on a cluster, submit a build job, load the right modules, set up LHAPDF or OpenLoops on the cluster, follow a SLURM job with squeue/sacct/tail, or debug a cluster build that differs from a local one.
---

# Building FLAME on Asteria

**Never build on the login node.** Either take an interactive job or submit the batch
script. Both follow the same recipe, documented in `code/misc/cluster_compilation_info`.

## Batch build (preferred)

From the repository root:

```bash
sbatch code/misc/build_flame_asteria.sbatch
```

Overrides are environment variables set before `sbatch`, and SLURM resources are
command-line flags:

```bash
PROCESS_DIR=code/process/dijet FLAME_INST=/scratch/$USER/flame-install sbatch --cpus-per-task=32 --mem=32G code/misc/build_flame_asteria.sbatch
```

| Variable | Default | Meaning |
| --- | --- | --- |
| `PROCESS_DIR` | `code/process/VJ` | process to build alongside the library |
| `FLAME_INST` | `$PWD/install_prefix` | persistent install prefix — **not** under `$TMPDIR` |

Script defaults: `--job-name=flame-build --cpus-per-task=16 --mem=16G --time=01:00:00
--output=flame-build-%j.log`. Build parallelism binds to `$SLURM_CPUS_PER_TASK`, so
raising `--cpus-per-task` is enough; raise `--mem` with it, since parallel compiler
processes are memory-hungry. Asteria compute nodes have 96 cores (`sinfo`), so 16 is a
quick-to-schedule slice rather than a full node.

## Following the job

`sbatch` prints `Submitted batch job <jobid>`.

```bash
squeue --me
```

```bash
tail -f flame-build-<jobid>.log
```

stdout and stderr are deliberately combined into that one file, so a compiler error
appears next to the step that caused it. `Ctrl-C` stops watching; it does not cancel
the job.

```bash
sacct -j <jobid> --format=JobID,JobName,State,Elapsed,ExitCode
```

`State` gives `COMPLETED` / `FAILED` / `TIMEOUT` / `OUT_OF_MEMORY`; `ExitCode 0:0` is
success. `OUT_OF_MEMORY` on a build means lower `--cpus-per-task` or raise `--mem`.

## Interactive build

```bash
srun --pty --mem=8G --cpus-per-task=8 bash -l
```

```bash
module load gcc/14.2.1
module load pythia8
export LHAPDF_DATA_PATH=/cvmfs/sft.cern.ch/lcg/external/lhapdfsets/current${LHAPDF_DATA_PATH:+:$LHAPDF_DATA_PATH}
```

`pythia8` is what puts `lhapdf-config` and the other HEP libraries on `PATH` — loading
`gcc` alone gives a configure failure on LHAPDF. `LHAPDF_DATA_PATH` must point at the
cvmfs PDF sets or runs fail at PDF load, not at build time.

Library, building in scratch and installing to a persistent prefix:

```bash
FLAME_INST="$PWD/install_prefix"
rm -rf "$TMPDIR/flame-lib-build"
cmake -S code/lib -B "$TMPDIR/flame-lib-build" -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="$FLAME_INST" -DENABLE_OPENLOOPS=ON -DENABLE_LHAPDF=ON
cmake --build "$TMPDIR/flame-lib-build" --parallel "${SLURM_CPUS_PER_TASK:-8}" --verbose && cmake --install "$TMPDIR/flame-lib-build"
```

Then find the installed CMake package — it lands in `lib` or `lib64` depending on the
toolchain, which is why this check exists:

```bash
if [[ -f "$FLAME_INST/lib/cmake/FLAMElib/FLAMElibConfig.cmake" ]]; then FLAMELIB_CMAKE_DIR="$FLAME_INST/lib/cmake/FLAMElib"; elif [[ -f "$FLAME_INST/lib64/cmake/FLAMElib/FLAMElibConfig.cmake" ]]; then FLAMELIB_CMAKE_DIR="$FLAME_INST/lib64/cmake/FLAMElib"; else echo "FLAMElibConfig.cmake not found under $FLAME_INST" >&2; fi
```

And the process, into its own prefix under the library's:

```bash
PROC_NAME=VJ
cmake -S "code/process/$PROC_NAME" -B "$TMPDIR/flame-$PROC_NAME-build" -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="$FLAME_INST/process/$PROC_NAME" -DFLAME_LIB_MODE=INSTALLED -DFLAMElib_DIR="$FLAMELIB_CMAKE_DIR" -DENABLE_OPENLOOPS_PROC=ON
cmake --build "$TMPDIR/flame-$PROC_NAME-build" --parallel "${SLURM_CPUS_PER_TASK:-8}" --verbose && cmake --install "$TMPDIR/flame-$PROC_NAME-build"
```

Result: `$FLAME_INST` contains `bin include lib64 process/$PROC_NAME`.

## OpenLoops caching — the one real trap

Since 2026-07-27 OpenLoops builds out-of-source and its generic and process libraries
are **cached under the install prefix**, because `$TMPDIR` is wiped when a job ends.
The first build against a given `FLAME_INST` pays the full OpenLoops compile; later
jobs against the *same* prefix reuse it — look for `OpenLoops: reusing
already-installed ...` in the configure log.

**This is a plain existence check, not a version check.** If `submodules/OpenLoops` is
updated, the stale install is reused silently and you build against the old OpenLoops.
Force a rebuild by removing the install directory or using a different prefix.

The two-prefix layout matters here: the library goes to `$FLAME_INST` and the
executable to `$FLAME_INST/process/<NAME>`, but OpenLoops is a dependency of
*FLAMElib*, so its cache lives under `$FLAME_INST`. This is handled inside
`code/process/VJ/CMakeLists.txt`, which points `OPENLOOPS_INSTALL_PREFIX` at the
installed FLAMElib's own prefix — no command-line change needed.

After a build, `submodules/OpenLoops` stays clean apart from some
`lib_src/<name>/obj/` residue, whose path is hardcoded in OpenLoops' own `SConstruct`
and deliberately not patched.

## Notes

- Build in `$TMPDIR`, install to a persistent prefix. `$TMPDIR` is per-job and is wiped
  at job end; anything you want to keep must be under `FLAME_INST`.
- `code/scripts/build_process.sh` (the `build` skill) is the local path and does not
  know about modules or SLURM. On Asteria use this recipe instead.
- In both `build_flame_asteria.sbatch` and `cluster_compilation_info`, the final cmake
  argument is followed by a trailing `\` and then a commented-out line. Bash treats the
  joined `#` as a comment so it works today, but uncommenting the line above it breaks
  the continuation. Edit that block carefully.
- `code/misc/laptop_compilation_info` is the equivalent local recipe, fully subsumed by
  `build_process.sh`.
