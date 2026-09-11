---
name: unit-tests
description: Build and run the FLAME doctest unit tests in code/lib/test — all of them, or one test binary, test case, or subcase. Use when asked to run unit tests, run ctest, check that a change did not break the tests, run a named test (test_phase_space, test_fks_mappings, test_cache, test_histogram, test_lorentz_four_vector, test_configuration, test_dynamic_particle), filter to one TEST_CASE or SUBCASE, debug a failing or timing-out unit test, or add a new test file to the suite.
---

# FLAME unit tests

Seven doctest binaries built from `code/lib/test/*.cpp`, registered with CTest under
the label `unit`. They are **off by default** — `ENABLE_TESTS` is `OFF`
(`code/lib/CMakeLists.txt:253`).

| Test | Covers |
| --- | --- |
| `test_lorentz_four_vector` | four-vector algebra, boosts, rapidity, conservation |
| `test_configuration` | runcard parsing and validation |
| `test_dynamic_particle` | `DynamicParticle` accessors, kinematics, parent indices |
| `test_histogram` | binning, filling, weights, under/overflow |
| `test_cache` | `CachingSystem/` operations and `PdfCacheKey` |
| `test_phase_space` | **slow** — real MC integrations, 600 s timeout |
| `test_fks_mappings` | FKS ISR/FSR mappings, including the round-trip inverse |

## Build with tests on

```bash
cmake -S code/lib -B code/lib/build -DENABLE_TESTS=ON && cmake --build code/lib/build -j
```

Check whether the existing build already has them before reconfiguring:

```bash
grep ENABLE_TESTS code/lib/build/CMakeCache.txt
```

Binaries land in `code/lib/build/bin/test/`. No install step is needed to run them.

## Run everything

```bash
ctest --test-dir code/lib/build --output-on-failure
```

`--output-on-failure` is not optional in practice — without it a failure prints only
the test name, and the doctest assertion that actually failed is hidden.

Useful additions:

| Flag | Effect |
| --- | --- |
| `-j N` | parallel; safe, the tests share no state |
| `-N` | list tests without running |
| `--rerun-failed` | re-run only what failed last time |
| `-L unit` | select by label (all of them carry `unit`) |
| `--timeout N` | override the per-test timeout |
| `-VV` | full verbose output |

## Run only what was asked for

Three levels of granularity. Use the narrowest one that answers the question.

### One binary, via CTest

`-R` is a regex over test names:

```bash
ctest --test-dir code/lib/build -R test_fks_mappings --output-on-failure
```

```bash
ctest --test-dir code/lib/build -R 'test_(cache|histogram)' --output-on-failure
```

`-E` excludes instead. To run everything *except* the slow MC test:

```bash
ctest --test-dir code/lib/build -E test_phase_space --output-on-failure
```

### One test case, via the doctest binary

CTest can only select whole binaries. To go finer, run the binary directly — it
takes doctest's own CLI (doctest 2.4.12) and needs no ctest wrapper:

```bash
code/lib/build/bin/test/test_fks_mappings --list-test-cases
```

```bash
code/lib/build/bin/test/test_fks_mappings -tc='FKS mappings: ISR*'
```

Filters are comma-separated and accept `*` wildcards, so `-tc='*ISR*'` works.

### One subcase

Most tests are built from `SUBCASE` blocks (22 in `test_lorentz_four_vector`, 36 in
`test_histogram`, 25 in `test_dynamic_particle`, 11 in `test_fks_mappings`, 12 in
`test_configuration`, 6 in `test_phase_space`; `test_cache` uses none):

```bash
code/lib/build/bin/test/test_histogram -tc='Histogram - Multiple weights' -sc='*overflow*'
```

### doctest flags worth knowing

| Flag | Effect |
| --- | --- |
| `-ltc` / `--list-test-cases` | list names (respects the current filters) |
| `-tc=` / `-tce=` | include / exclude test cases by name |
| `-sc=` / `-sce=` | include / exclude subcases |
| `-sf=` / `-sfe=` | filter by source file |
| `-s` | print successful assertions too — use when a test passes but you doubt it |
| `-d` | print per-test duration; the first step on a timeout |
| `-aa=1` | abort after the first failed assertion |
| `-m` | minimal output, failures only |
| `-ob=rand -rs=N` | randomise order to expose inter-test coupling |
| `-nt` | skip exception-related assertions |

All flags also accept a `dt-` prefix (`--dt-test-case=`) when the program under test
has conflicting options of its own.

## When a test fails

1. **Re-run just that binary directly** with `-s -d`. The CTest layer adds nothing
   at this point and hides the assertion output.
2. **Check it is not a timeout.** Non-slow tests are capped at **30 s**,
   `test_phase_space` at **600 s** (`code/lib/test/CMakeLists.txt`, `SLOW_TESTS`).
   A test that got slower rather than wrong reports as a failure. `-d` distinguishes
   them; `ctest --timeout` confirms.
3. **Rebuild with the caching layers off** —
   `-DDISABLE_EQUIV=1 -DDISABLE_CACHE_EQUIV=1 -DDISABLE_CACHE_PHSP_REAL=1`. If the
   test then passes, the bug is in `EquivalenceRelation/` or `CachingSystem/`, not in
   the physics.
4. **For `test_fks_mappings`**: the mapping round-trip depends on
   `TRANSVERSE_DEGENERATE_SIN2 = 1e-16` in
   `code/lib/include/PhaseSpace/phase_space_utils.hh:102`. Do not loosen it to make a
   test pass — that threshold is tuned against a dijet regression. The non-plain
   (`Local`/`Global`) mapping leaves are unimplemented scaffolding whose methods
   throw by design; a test asserting that they throw is asserting correct behaviour.
5. **A unit test passing is not a regression test passing.** These check units, not
   cross sections. For numerical neutrality run the `regression-test` skill.

## Adding a test

`TEST_SOURCES` in `code/lib/test/CMakeLists.txt` is an **explicit list, not a glob** —
a new `test_*.cpp` dropped into the directory is silently ignored until it is added
there. If it runs MC integrations, add its name to `SLOW_TESTS` in the same file so
it gets the 600 s timeout instead of 30 s.

Each source becomes its own executable linked against `flamelib` (plus fmt, kakuhen,
LHAPDF, hoppet when those targets exist), compiled with `-Wall -Wextra -pedantic` at
C++20, and registered via `add_test` with the label `unit`. Follow the existing
files: `doctest.h` is vendored in the same directory.
