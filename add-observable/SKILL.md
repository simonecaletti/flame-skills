---
name: add-observable
description: Add a new histogram (observable) to an existing FLAME analysis. Use when asked to add, book or plot a new distribution, histogram a new variable such as a pT, rapidity, invariant mass or angle, add a 2D histogram, or change the binning of an observable in a process's analyses_* directory.
---

# Adding an observable to an existing analysis

Scope: adding **one histogram to an analysis that already exists**. Creating a new
analysis class is a different, larger job (it needs registration in
`create_analysis_manager()` and a new `.cc` in the process CMakeLists).

Two edits, both in the same analysis `.cc`: **book** it in the constructor, **fill** it
in `fill()`. Nothing else — no CMake change, no registration, no runcard change.

## Where the analyses live

`code/process/<PROC>/analyses_<proc>/`. Pick the analysis the user actually runs — the
one named by `[output] analysis` in their runcard, or by `flame_exe -a`. The canonical
example with both a 1D and a 2D histogram is
`code/process/DY/analyses_dy/neutral_current.cc`.

## 1. Book it in the constructor

```cpp
histogram_name = "pt_ll";
n_dimensions = 1;
_histogram_manager.book(histogram_name, n_dimensions,
                        {std::make_tuple(0.0, 100.0, 20)});
```

One `std::make_tuple(xmin, xmax, nbins)` per dimension. A 2D histogram is two tuples
and `n_dimensions = 2`:

```cpp
histogram_name = "mll_pt_ep";
n_dimensions = 2;
_histogram_manager.book(
    histogram_name, n_dimensions,
    {std::make_tuple(20., 140., 20), std::make_tuple(0.0, 70., 20)});
```

Follow the surrounding style: the existing files reuse the local `histogram_name` and
`n_dimensions` variables for each booking rather than passing literals.

**Variable binning** uses a second overload taking explicit edges per dimension
(`Histogram/histogram.hh:379`) instead of the uniform tuple form:

```cpp
_histogram_manager.book("mll", 1, std::vector<std::vector<double>>{{20., 40., 60., 80., 91., 100., 140.}});
```

## 2. Fill it in `fill()`

```cpp
_histogram_manager.fill("pt_ll", {pt_ll(event)}, dsigma);
```

The coordinate vector has one entry per dimension, in the same order as the bookings:

```cpp
_histogram_manager.fill("mll_pt_ep", {mll(event), pt_ep(event)}, dsigma);
```

`dsigma` is passed **straight through, unmodified**. It is the weight vector: index 0
is the central weight and the rest are the scale variations requested by
`[scales] scale_variations`. Never index into it, scale it, or pass `dsigma[0]` — that
silently drops every variation and the scale band disappears from the plots.

The name in `fill` must match the name in `book` exactly. A mismatch is an `assert`
failure at runtime (`histogram.hh:403`), not a compile error — and in a `-DNDEBUG`
Release build the assert is compiled out, so the fill lands nowhere silently. Copy the
string rather than retyping it.

### Conditional filling

Guard with a plain `if` when the observable is only defined for some events:

```cpp
const auto* final_parton = event.find_first_final(SM::is_parton);
if (final_parton and final_parton->p4().perp() > 30.) {
  _histogram_manager.fill("cross_section_real", {.5}, dsigma);
}
```

A booked histogram that is never filled still appears in the `.top` output, empty —
useful to distinguish "cut removed everything" from "I forgot the fill".

## 3. If the observable needs a helper

Compute it inline when it is a one-liner. For anything longer, follow the existing
pattern: a member function taking the event, declared in the analysis `.hh` and defined
in the `.cc`.

```cpp
// in neutral_current.hh, inside the class
double pt_ep(const Event<double> &event);
```

```cpp
// in neutral_current.cc
double AnalysisNeutralCurrent::pt_ep(const Event<double> &event) {
  for (auto &particle : event.final_particles()) {
    if (particle.particle()->pdg_id() == -11) {
      return particle.p4().perp();
    }
  }
  ...
}
```

Useful event accessors: `event.final_particles()`, `event.find_first_final(SM::is_parton)`,
`particle.p4()` (a `LorentzFourVector` with `perp()`, `rapidity()`, `pseudorapidity()`,
`m()`, …), `particle.particle()->pdg_id()`.

## Flavour blindness

The analysis constructor sets `_is_flavour_blind`. Leave it alone unless the new
observable genuinely depends on the initial- or final-state flavour assignment — in
which case the analysis is no longer flavour blind and the flag must be `false`, or
results will be wrong in a way no test catches. Note that under a
`DISABLE_CACHE_PHSP_REAL` build `is_flavour_blind()` returns `false` regardless, so
that build is the cross-check.

## Verify

Rebuild and run; the new histogram must appear in the `.top` output.

```bash
code/scripts/build_process.sh --proc-src code/process/DY --jobs 8
```

```bash
cd code/process/DY && ./build/bin/flame_exe -r RUNCARD.run --all-in-one -n 10000 -i 1 && grep -c "pt_ll" results/output/top/histograms_st_2_*.top
```

Then plot it with the `plot-histograms` skill.

Adding a histogram is additive and must not change any existing number. If the process
has a regression baseline, confirm that: `cd code/regression_test && python3
run_regression.py --test --runcard <name>` — cross sections compare at `1e-14`, so any
change means the edit touched more than the new histogram. (Histogram comparison will
flag the new observable as absent from the baseline; that part is expected.)
