---
name: ir-limits
description: Read and use the FKS IR-limit check that validates the real-emission subtraction — the soft, collinear and soft-collinear scans written to check_limit_channel_*.log. Use when asked whether the subtraction cancels, to check IR limits, to debug a real-radiation or counterterm problem, to read or plot check_limit_channel logs, or after changing FKS mappings, sector functions, subtraction terms or matrix elements.
---

# The FKS IR-limit check

For each integration channel the harness generates one random Born point, then walks the
radiation variables into each singular limit and prints

```
ratio-1   where   ratio = -(real) / (counterterm)
```

per **emitter leg** and per **real flavour configuration**. If the subtraction is
correct, the real emission and its counterterm cancel, so `ratio → 1` and `ratio-1 → 0`
for every configuration that is actually singular in that limit.

## It runs by itself

There is **no CLI flag and no runcard key.** `build_real_equivalence_relations()`
(`cross_section_btilde.hh:110`) loops every channel and calls the checker for all three
limits, and `flame.hh:575` calls that whenever

```
[nlo_mode] perturbative_order = NLO
[nlo_mode] realsub = on
```

So every NLO run with `realsub` on already did the scan at startup. To produce a log,
just do a short run — `-n` can be tiny, the scan does not use the integration
statistics:

```bash
cd code/process/VJ && ./build/bin/flame_exe -r RUNCARD.run --all-in-one -n 1000 -i 1
```

The method name is misleading: it builds no equivalence relations, it only runs the
check and then clears the cache.

## Where the log lands

```
<output_dir>/output/log/check_limit_channel_<channel_name>.log
```

One file per integration channel. The Soft pass truncates the file; Collinear and
SoftCollinear append — so a complete log holds all three sections in that order, and a
log with only a soft section means the run died partway. If the file cannot be opened
the output goes to stdout with a warning.

Do not confuse these with `born_equivalences.log`, `real_equivalences.log` and
`regular_equivalences.log` in the same directory — different machinery.

## The scans

| Limit | Scanned | Values |
| --- | --- | --- |
| Soft | `xi` | `1e-1 … 1e-13`, with `omy` fixed at a random value in `[0.1, 1.9)` |
| Collinear | `omy` | same 13 values, `xi` from `make_collinear_xi()` |
| SoftCollinear | `(xi, omy)` pairs | products `1e-6 … 1e-14` |

The Born point is reproducible: the RNG is seeded `987654 + channel`, so the same
channel always gets the same Born point and two logs are directly comparable. Soft
limits are only scanned for gluon emission, collinear limits only for massless emitters,
and Born flavours equivalent under `is_born_flavour_equivalent` are skipped.

## Reading a series

A healthy series, from a real `VJ_WmJ` log:

```
pdgs_real=(1,-2,11,-12,21,21) ratios:
 (small var=  1e-01, ratio-1=   0.002995336933771)
 (small var=  1e-02, ratio-1=  -0.000087663885554)
 (small var=  1e-03, ratio-1=  -0.000007347655272)
 (small var=  1e-04, ratio-1=  -0.000000718347214)
 (small var=  1e-05, ratio-1=  -0.000000071675571)
 (small var=  1e-06, ratio-1=  -0.000000007151128)
 (small var=  1e-07, ratio-1=  -0.000000000212188)   <- best
 (small var=  1e-08, ratio-1=  -0.000000002105972)
 (small var=  1e-09, ratio-1=  -0.000000001138426)
 (small var=  1e-10, ratio-1=   0.000000371345892)
 (small var=  1e-11, ratio-1=   0.000002607588268)
 (small var=  1e-12, ratio-1=   0.000051563805551)
 (small var=  1e-13, ratio-1=  -0.000179458319466)
```

Two things to read, in this order:

1. **The descent.** `ratio-1` should fall roughly one order of magnitude per decade of
   the limit variable. That linear convergence is the actual evidence the subtraction is
   right.
2. **The turnaround.** Below the minimum the numbers get *worse* again. That is
   double-precision cancellation between two large, nearly equal terms — **not a bug**,
   and not something to fix. Judge a series by its minimum (here `2e-10` at `1e-7`), not
   by its last row.

A series that never turns around and reaches `~1e-13` … `1e-15` is ideal; the DY
`photon` and `Z_boson` channels do exactly that.

### What failures look like

| Symptom | Meaning |
| --- | --- |
| Plateau at a constant ≠ 0 | real and counterterm differ by a finite factor — either a genuine subtraction bug, or a configuration that is **not singular** in this limit (see below) |
| `nan` | the counterterm evaluated to exactly 0 (`ratio` is set to NaN by construction) |
| constant `-1.000000000000000` | `ratio` is 0, i.e. the **real term** evaluated to 0 |
| Descent that stops early, e.g. flattens at `1e-4` | a subleading term is not being subtracted |
| Sign flip mid-descent | usually the roundoff floor, benign; before the minimum, suspicious |

### Most series do not converge — that is normal

The check prints a series for **every** (emitter leg × real flavour configuration) pair,
but only the pairs that are genuinely singular in that limit are supposed to tend to
zero. In the current regression output roughly 75–85 % of points sit above `1e-2`:
726/960 for `DY_Z_gamma`'s `photon` channel, 2397/2810 for `VJ_WmJ`'s `W_boson`. A
plateau such as

```
pdgs_real=(-2,1,11,-12,21,21) ratios:
 (small var=  1e-09, ratio-1=  -0.475642622198798)
```

is the normal appearance of a non-singular combination, not evidence of a broken
subtraction.

**Consequence: never judge the log by an aggregate statistic.** Counting bad points, or
computing a mean, says nothing. Either identify which (emitter, flavour, limit)
combination is supposed to be singular — which needs the physics, not the log — or use
the differential method below.

## The reliable method: diff two logs

Because the Born point is seeded per channel, a log is reproducible run to run. So the
robust check after touching FKS mappings, sector functions, subtraction terms or matrix
elements is **before/after**, not absolute:

```bash
cd code/process/VJ && ./build/bin/flame_exe -r R.run --all-in-one -n 1000 -i 1 && cp -r results/output/log /tmp/ir_before
```

```bash
# ... make the change, rebuild ...
cd code/process/VJ && ./build/bin/flame_exe -r R.run --all-in-one -n 1000 -i 1 && diff -r /tmp/ir_before results/output/log
```

For a refactoring that must be behaviour-preserving the logs should be **identical**.
For a change that improves the subtraction, the converging series should reach a lower
minimum and the non-singular plateaus should be unchanged.

## Plot it

```bash
python3 code/scripts/plotting/plot-limits-check.py results/output/log/check_limit_channel_W_boson.log -o limits.pdf --legs 1 2 5 --aggregate max
```

Log-log `|ratio-1|` scatter, one page per emitter leg. `--legs` defaults to `1 2 5`, so
pass the legs you actually care about. `--aggregate` is `max`, `mean` or `none`.

**Always pass the logfile explicitly** — its default path is resolved relative to the
*script's* directory, not yours, so the bare invocation reads someone else's stale log
or fails.

## The check is meaningless in two modes

`nlo_contribution` changes what the counterterm is, and both alternatives destroy the
test (`check_IR_limits.cc:492-502`):

| `nlo_contribution` | Counterterm | Effect on the log |
| --- | --- | --- |
| `standard` | the real subtraction term | **the only meaningful setting** |
| `p2b` | forced to `-xsec_real` | every `ratio-1` is exactly 0 — trivially, regardless of correctness |
| `sector_function` | forced to `0` | every ratio is `nan` |

A log full of clean zeros is therefore not good news until you have checked that the
runcard says `nlo_contribution = standard`. Check it first, every time.

## Getting more detail

The per-point breakdown — real term, soft/collinear/soft-collinear counterterms, sector
functions and all four jacobians — is already computed but sits behind `if (false)` at
`code/lib/src/CrossSection/check_IR_limits.cc:563`. Flip it to `true` and rebuild when a
series misbehaves and you need to see which piece is wrong. Remember to flip it back;
the output is very large.

Also useful when a limit fails: rebuild with `-DDISABLE_EQUIV=1 -DDISABLE_CACHE_EQUIV=1
-DDISABLE_CACHE_PHSP_REAL=1` so every term is recomputed from scratch. `check_IR_limits`
is the only consumer of the `prepare_refs=true` path through
`map_born_and_phirad_to_real`, so a bug there shows up here and nowhere else — the
regression suite does not cover it.
