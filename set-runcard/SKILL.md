---
name: set-runcard
description: Create a FLAME runcard by generating the process's own default template with flame_exe -w and filling in the requested values. Use when asked to write, create, set up or modify a runcard, change beam energy, PDF set, scales, cuts, number of calls, seed, stage, analysis or NLO settings, or when a run fails runcard validation.
---

# Writing a FLAME runcard

**Never hand-write a runcard and never copy a stale one.** Generate the template from
the process executable itself, then substitute the requested values. The template is
process-specific — it lists that process's allowed scale choices, its cut names, its
available analyses and its `[User]` keys — so a template from another process will
fail validation.

## 1. Generate the template

```bash
cd code/process/DY && ./build/bin/flame_exe -w
```

This writes **`default.run` in the current working directory**, always under that
name, overwriting whatever is there. Run it in a scratch directory, or rename the
result immediately:

```bash
cd code/process/DY && ./build/bin/flame_exe -w && mv default.run myrun.run
```

The repo-root `default.run` is exactly such a dump (for DY) and is the reference for
the format, but it is not a substitute for generating a fresh one for the process at
hand.

## 2. Substitute the requested values

Edit the generated file in place, replacing defaults with what the user asked for.
Keep the comments — they document the accepted values for each key, and the template
regenerates them anyway.

Mandatory keys are sentinel-initialised (`-1`, `undefined`) and rejected by
`validate()` at startup, so every one of these must be set before the runcard runs:

| Key | Sentinel |
| --- | --- |
| `[beam] energy_A`, `energy_B` | `-1.000000` |
| `[beam] beam_A`, `beam_B` | `undefined` |
| `[beam] pdf` | commented out — mandatory if any beam is a hadron |
| `[run_options] stage` | `-1` |
| `[run_options] random_seed` | `-1` |
| `[run_options] ncall1..4` | `-1` (mandatory for the current stage) |
| `[scales] mu_f`, `mu_r` | `-1` (only when `scale_choice = Fixed`) |

## The ten sections

| Section | Commonly set |
| --- | --- |
| `[beam]` | `energy_A`, `energy_B` (GeV), `beam_A`/`beam_B` (`proton`, `electron`, `positron`, …), `pdf` |
| `[particles]` | `max_nq_light`, `mass[pdg]`, `width[pdg]` — note `mass_phsp` is what stages 1 and 2 use |
| `[ew_couplings]` | `input_scheme` (`GFMZMW`, `AlphaMZMW`, `AlphaGFMZ`), `complex_scheme`, `one_over_alpha`, `GF`, CKM angles |
| `[qcd]` | `alphas_running_type`, `pdf_running_provider` (`hoppet`/`lhapdf`), `nlight_max`, `leading_colour` (`false`/`CATwoCF`/`CFHalfCA`), `mu_ref_running`, `alphas_ref_running`, `nloops` |
| `[scales]` | `scale_choice` (or `scale_choice_mu_f`/`_mu_r` independently), `mu_f`, `mu_r`, `scale_factor_mu_*`, `scale_variations` |
| `[selectors]` | generation cuts (`s_min`, `mll_min`, `mll_max`) and process posterior cuts (`pt_em`, `pt_ep`, …) |
| `[run_options]` | `stage` (1–4), `random_seed`, `ncall1..4`, `grid_iteration` |
| `[output]` | `output_dir` (default `./results`), `analysis` |
| `[nlo_mode]` | see below |
| `[User]` | per-process, e.g. DY's `subprocess = Z_gamma\|Z_only\|Wp\|Wm` |

### PDF syntax

```
pdf = NNPDF40_nnlo_as_01180[0]                                  single member
pdf = NNPDF40_nnlo_as_01180[*]                                  all members
pdf = [NNPDF40_nnlo_as_01180[0], NNPDF40_nnlo_as_01180[1]]      explicit list
pdf = [NNPDF40_nnlo_as_01180[*], NNPDF31_nlo_as_0118[0]]        mixed sets
```

### Scales

`scale_choice` accepts `Fixed` plus the dynamic choices that process declares — the
generated template prints the list under "Available options for this process" (DY
offers `DileptonMass`, `DileptonTransverseMass`). `mu_f`/`mu_r` apply only when the
choice is `Fixed`.

Scale-like keys accept particle-mass references: `mu_f = m[Z_boson]` or `m[23]`.

`scale_variations` is `7point`, `5point` or `none`, and determines how many weights
each histogram carries (index 0 central, the rest variations — see the
`plot-histograms` skill). Custom sets instead:

```
variation[central]  = 1.0, 1.0
variation[muf_up]   = 2.0, 1.0
```

### `[nlo_mode]`

```
perturbative_order = LO | NLO
nlo_contribution   = standard | p2b | sector_function
born = on   softvirt = on   collremn = on   realsub = on   remnant = off   regular = on
remnant_factor = smooth | sharp | none | <process-defined>
h_damp = off   b0_damp = off   mu_h = 20   k_b0 = 5
scale_factor_mu_h = 1   scale_factor_k_b0 = 1
```

The six contribution switches read `on`/`true`; anything else is off. Defaults are all
on except `remnant`. `p2b` and `sector_function` are rejected when `realsub = off`.
`mu_h` accepts the same values as `mu_r`/`mu_f`, mass references included. `h_sharp`
and `h_b0` apply only when `remnant_factor = sharp`.

**Typo trap:** any key in `[nlo_mode]` that is not recognised is silently treated as a
remnant parameter rather than rejected. Writing `real_sub` instead of `realsub` does
not error — it leaves `realsub` at its default and quietly changes nothing. Check
spelling against the generated template rather than from memory.

## 3. Run it

```bash
./build/bin/flame_exe -r myrun.run --all-in-one -n 100000 -i 1
```

CLI overrides beat the runcard for a subset of keys: `-s/--stage`, `-i/--iseed`,
`-n/--ncall`, `-g/--grid-iteration`, `-a/--analysis`, `-o/--output`, `-m/--message`.
Use them for values that change per job — seed and stage especially — and keep the
runcard for the physics. `--all-in-one` is incompatible with `-s`.

For a multi-seed production, leave `stage` and `random_seed` in the runcard and let
`run_parallel_stages.py` override them per job (the `production-run` skill).

## Validation failures

`validate()` runs at startup and names the offending key. The usual causes: a
sentinel left unset, a `scale_choice` that this process does not declare, an
`analysis` name not registered by this process (the error lists the registered ones),
or a `[User]` key belonging to a different process. All four mean the template came
from the wrong executable — regenerate with `-w` from the right one.
