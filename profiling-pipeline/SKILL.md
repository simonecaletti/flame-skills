---
name: profiling-pipeline
description: Profile FLAME for CPU time, memory, or instruction-level cost, and produce a structured optimisation report. Use when asked to profile FLAME, find hotspots or bottlenecks, investigate why a run is slow or uses too much memory, produce a flamegraph or callgraph, run perf / heaptrack / valgrind / callgrind / massif / cachegrind on flame_exe, measure the benefit of the caching or equivalence layers, or decide what to optimise next.
---

# Profiling FLAME

Three tracks. Pick by question:

| Question | Track |
| --- | --- |
| Where does wall-clock time go? | **A — perf** |
| Why does it allocate so much / grow? | **B — memory** |
| Exactly how many instructions, and which cache misses? | **C — valgrind** |

Always finish with the **structured report** at the bottom. A profiling run that
ends in a pile of SVGs and no ranked list of actions is not finished.

---

## 0. Prerequisites — do not skip

### Build with debug info

The default build is `Release` = `-O3 -DNDEBUG` with **no `-g`** (`code/lib/CMakeLists.txt:34,40`
and the same lines in each process's `CMakeLists.txt`). Profiling that binary gives
unresolved stacks and an empty `perf annotate`. Verify before profiling anything:

```bash
readelf -S code/process/VJ/build/bin/flame_exe | grep -c debug_info
```

`0` means rebuild. Two options:

```bash
# Preferred: -O2 -g -DNDEBUG. Neither CMakeLists overrides RelWithDebInfo.
cmake -S code/lib -B code/lib/build -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=code/lib/build -DCMAKE_CXX_FLAGS="-fno-omit-frame-pointer"
```

```bash
# Keep production -O3 exactly, just add symbols:
cmake -S code/lib -B code/lib/build -DCMAKE_CXX_FLAGS_RELEASE="-O3 -g -DNDEBUG -fno-omit-frame-pointer" -DCMAKE_INSTALL_PREFIX=code/lib/build
```

Then rebuild the process against that prefix (see the `build` skill). Use the
**same** flags for library and process or the stacks will be half-symbolised.

`-fno-omit-frame-pointer` is worth the ~1% it costs: it enables `--call-graph fp`,
which is dramatically cheaper and smaller than `dwarf` (dwarf copies 8 KB of stack
per sample, inflating `perf.data` to hundreds of MB and distorting the very
measurement you are taking). Use `dwarf` only when frame pointers are missing in a
dependency you care about.

### Make the run measurable

- **Fix the seed and ncall.** `-i 1 -n <N>` with the same values every time, so two
  profiles are comparable. Unseeded MC runs are not.
- **Profile stages separately.** `--all-in-one` conflates stage 1 (grid training)
  and stage 2 (production); they exercise different code in different proportions.
  Use `-s 1` and `-s 2` for attributable numbers, `--all-in-one` only for a first look.
- **Skip startup.** LHAPDF grid loading and OpenLoops initialisation dominate a short
  run and are not what you are optimising. Either use an `-n` large enough to drown
  them out, or `perf record --delay 5000` (ms) to start sampling after init.
- **Pin the CPU.** `taskset -c 2` — this machine shows heavy frequency scaling, and
  migrations smear call-graph attribution.
- **Repeat for timings.** Use `perf stat -r 5` for any before/after claim. A single
  wall-clock number on a scaling CPU is not evidence.

### This machine's limits

Established by direct check — do not report around them:

- `perf_event_paranoid=2` → user-space only, every event reads as `:u`, no kernel
  frames, no system-wide profiling. Fine for FLAME, which is compute-bound in userspace.
- `LLC-loads` / `LLC-load-misses` are **`<not supported>`** on this AMD Ryzen AI 7 PRO 350.
  `perf stat -d` gives L1-dcache only. For last-level cache, use **cachegrind** (Track C).
- `--call-graph lbr` fails: no branch-stack sampling on this PMU. Use `fp` or `dwarf`.

---

## Track A — perf: where the time goes

### A1. Baseline counters

```bash
perf stat -d -r 5 ./build/bin/flame_exe -r RUNCARD --all-in-one -n N -i 1
```

Read: `insn per cycle` (<1.0 → stalled, memory-bound; >2 → compute-bound),
`branch-misses` %, `L1-dcache-load-misses` %. This is the number you will quote as
"before" in the report.

### A2. Record with a call graph

```bash
perf record -F 999 -g --call-graph fp -e cycles:u -o perf.data -- ./build/bin/flame_exe -r RUNCARD --all-in-one -n N -i 1
```

Swap `fp` → `dwarf` if stacks come back truncated; add `--mmap-pages 256` with dwarf
to avoid "lost chunks". `-F 999` (not 1000) avoids lock-stepping with periodic work.

### A3. Read it

```bash
perf report --children --percent-limit 0.5 --stdio       # inclusive: which subsystem
perf report --no-children --percent-limit 0.5 --stdio    # self: which line to fix
```

`--children` finds the *area* (e.g. all of `CrossSection::dsigma`), `--no-children`
finds the *instruction*. Report both — a subsystem at 60 % inclusive but 2 % self
means the cost is in what it calls, not in it.

C++ template symbols are enormous; `--sort symbol` plus `--percent-limit` keeps the
output readable.

### A4. Annotate a hotspot

```bash
perf annotate --stdio -s 'FLAME::MatrixElementVJ<double>::get_squared_me'
```

Needs the `-g` build. Look for: divisions in inner loops, `sqrt`/`pow` calls that
could be hoisted, and reload-after-store patterns that indicate a missed
vectorisation.

### A5. Flamegraph

```bash
perf script -i perf.data > out.perf
stackcollapse-perf.pl out.perf > out.folded
flamegraph.pl out.folded > flame.svg
```

Open `flame.svg` in the in-app browser pane rather than launching `firefox` on the
user's desktop, so the width of each frame can actually be read back to them.

**Differential flamegraphs** are the highest-value trick here and cheap: collapse
two runs and diff them to see exactly what an optimisation moved.

```bash
difffolded.pl before.folded after.folded | flamegraph.pl > diff.svg
```

### A6. Callgraph PDF

```bash
gprof2dot -f perf out.perf -s -z 'FLAME::CrossSection::dsigma*' --depth 10 -n 1 -e 0.5 -o callgraph.dot
dot -Grankdir=TB -Tpdf callgraph.dot -o callgraph.pdf
```

`-z` roots the graph at a function, `-n`/`-e` prune nodes/edges below a percentage,
`-s` strips template arguments (essential for FLAME's symbol names).

### A7. FLAME-specific: price the caching layers

This is the measurement that matters most in this codebase and has no equivalent
elsewhere. Build twice — once normally, once with `-DDISABLE_EQUIV=1
-DDISABLE_CACHE_EQUIV=1 -DDISABLE_CACHE_PHSP_REAL=1` — and compare `perf stat`
and flamegraphs. It tells you what `EquivalenceRelation/` and `CachingSystem/`
are actually buying, and whether the cache lookup itself has become a hotspot.
Toggle them individually to attribute the gain.

---

## Track B — memory

### B1. heaptrack (primary)

```bash
heaptrack -o ht ./build/bin/flame_exe -r RUNCARD --all-in-one -n N -i 1
heaptrack_print ht.zst | head -100
```

Use **`heaptrack_print`**, not the GUI — `heaptrack` auto-launches `heaptrack_gui`
and pops a window on the user's desktop. `heaptrack_print --print-flamegraph f.folded ht.zst`
then feeds `flamegraph.pl` for an allocation flamegraph.

What to look for, in priority order:

1. **Number of allocations**, not bytes. In an MC integrator the killer is a
   `std::vector` allocated per phase-space point. Anything with an allocation count
   proportional to `ncall` is a bug worth fixing.
2. **Temporary allocations** (allocated and freed almost immediately) — heaptrack
   reports these separately; they are pure overhead.
3. **Peak RSS** — matters for cluster job limits.
4. **Leaks** — heaptrack reports these too, and for a run that ends cleanly they
   should be near zero.

Compare two runs directly:

```bash
heaptrack_print -d ht_before.zst ht_after.zst | head -60
```

### B2. valgrind massif — heap over time

```bash
valgrind --tool=massif --massif-out-file=massif.out --time-unit=B ./build/bin/flame_exe -r RUNCARD -n <small N> -i 1
ms_print massif.out | head -80
```

Answers a different question from heaptrack: *when* the heap grows, and whether it
plateaus. Use it when peak RSS or a suspected leak-like growth is the concern.
`--time-unit=B` makes the graph deterministic across machines.

### B3. valgrind dhat — how allocations are used

```bash
valgrind --tool=dhat --dhat-out-file=dhat.out ./build/bin/flame_exe -r RUNCARD -n <small N> -i 1
```

DHAT reports read/write counts per allocation and short-lived vs long-lived blocks.
It is how you prove a heap buffer should have been a stack array or a reused member —
the most common realistic win in this codebase's inner loops.

`massif-visualizer` and `heaptrack_gui`'s companion tools are not installed; the
text output above is complete.

---

## Track C — valgrind: exact counts

Valgrind runs 20–100× slower than native. **Reduce `-n` by at least two orders of
magnitude** and profile stage 2 only. The counts are deterministic and
machine-independent, which is exactly what perf's sampling is not.

### C1. callgrind — instruction counts and a precise call graph

```bash
valgrind --tool=callgrind --callgrind-out-file=cg.out --cache-sim=yes --branch-sim=yes ./build/bin/flame_exe -r RUNCARD -n <small N> -i 1
callgrind_annotate --auto=yes --threshold=95 cg.out | head -120
```

Deterministic `Ir` counts make A/B comparison of two implementations trivial — no
statistical noise, so a 2 % improvement is measurable. `--auto=yes` annotates source
lines directly. `kcachegrind` is not installed; `callgrind_annotate` covers it.

Callgrind output also feeds the same graph pipeline:
`gprof2dot -f callgrind cg.out -s -n 1 -e 0.5 -o cg.dot && dot -Tpdf cg.dot -o cg.pdf`

### C2. cachegrind — the LLC data perf cannot give here

```bash
valgrind --tool=cachegrind --cachegrind-out-file=cache.out ./build/bin/flame_exe -r RUNCARD -n <small N> -i 1
cg_annotate cache.out | head -60
```

Since this machine's PMU does not expose `LLC-*`, this is the **only** way to see
last-level cache behaviour. Read `D1mr`/`DLmr` (data read misses) against the
caching layer's hash lookups — a cache that misses in the CPU cache on every probe
can cost more than the recomputation it avoids.

### C3. memcheck — only when correctness is suspect

```bash
valgrind --tool=memcheck --track-origins=yes ./build/bin/flame_exe -r RUNCARD -n <tiny N> -i 1
```

Not a profiler, but uninitialised reads in phase-space or ME code show up as
irreproducible numbers long before they show up as crashes. Worth one run when a
result looks wrong.

---

## Verify before claiming a win

Any optimisation must be numerically neutral unless it deliberately is not. After
changing code, run the regression suite (see the `regression-test` skill):

```bash
cd code/regression_test && python3 run_regression.py --test --verbose --runcard VJ_WmJ
```

Tolerances are `1e-14` on cross sections. A reordering of floating-point operations
*will* trip them — if it does, say so explicitly in the report rather than quietly
regenerating the baseline.

---

## The report

End every profiling session by writing `profiling_report_<date>_<topic>.md` and
handing it to the user with `SendUserFile`. Structure:

```markdown
# FLAME profiling report — <process>, <date>

## 1. Setup
Binary, build flags, runcard, `-n`/`-i`, stage, machine, tool versions.
State the caveats that apply (paranoid=2, no LLC counters, etc.).

## 2. Headline numbers
Wall time (`perf stat -r 5`, mean ± sd), IPC, branch-miss %, L1-miss %,
peak RSS, total allocations. This is the "before" row of every later comparison.

## 3. Hotspots
Ranked table. For each: symbol, file:line, self %, inclusive %, and one sentence
on *why* it costs — not just that it does.

| # | Symbol | file:line | self % | incl % | Cause |

## 4. Findings
One numbered subsection per actionable finding, most valuable first:
- **What** — the specific code construct, with `file:line`.
- **Evidence** — the measurement that shows it, quoted (a perf line, a heaptrack
  count, an Ir count). Never a hunch.
- **Proposed change** — concretely what to write instead.
- **Expected gain** — estimated % of total runtime, and how that estimate was made.
- **Risk** — numerical (does it reorder FP ops?), structural, and what verifies it.

## 5. Non-findings
What looked hot but is not worth touching, and why. This prevents the next person
re-profiling the same thing.

## 6. Recommended order of work
Ranked by (expected gain / risk), with the cheapest safe win first.
```

Rules for the report:

- **Rank by measured cost, never by how easy the fix looks.** A 0.5 % function with
  an obvious fix goes below a 30 % function with a hard one.
- **Every claim carries its measurement.** If a number is not in the report, the
  finding is a hypothesis and must be labelled as one.
- **Distinguish algorithmic from micro-optimisation.** Removing a redundant call to
  a matrix element beats vectorising it. Look for the former first — in this
  codebase that means the caching, equivalence, and channel-loop structure.
- **State what was not measured.** Multi-threaded behaviour, other processes, other
  runcards, other stages.
