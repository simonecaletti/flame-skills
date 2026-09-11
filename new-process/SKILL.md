---
name: new-process
description: Scaffold a new FLAME process directory under code/process/ with the full file set and the Process contract stubbed out. Use when asked to add, create or set up a new process, start a new channel or final state, or generate the boilerplate for a process that does not exist yet.
---

# Scaffolding a new FLAME process

## First: ask for the folder name

**Ask the user what the process folder should be called before creating anything.**
Directory names in this tree are mixed-case and not derivable from the physics —
`DY`, `VJ`, `ggH`, `dijet`, `epemjj`, `hvq`. Take the name exactly as given; do not
upper-case it, lower-case it, or invent one.

The folder name also fixes the file suffix, which is often **not** the folder name:
`dijet/` uses `_2j` (`process_2j.cc`, `matrix_element_2j.hh`, `analyses_2j/`), `DY/`
uses `_dy`. Ask for the suffix too if the folder name is not an obvious suffix.

## Do not use the generator

`code/scripts/newproc/generate_process.py` is **broken** and its output does not
compile. It was written against an older `Process` interface and emits
`get_allowed_dynamic_scales()` instead of `get_allowed_scale_choices()`,
`Scales`/`Selectors` instead of `ScaleManager`/`PosteriorCuts`, a one-argument
`build_integration_channels`, a `phase_space_<name>.hh` that live processes no longer
have, and a `CMakeLists.txt` referencing a nonexistent
`../../cmake/external_dependencies.cmake`. It never implements `get_n_alphas_born()`
or `check_consistency_beams()` at all, and it writes to `process/<NAME>.upper()`.

**Scaffold from a live process instead** — `DY` for a colour-singlet final state,
`dijet` for a QCD one (it also shows the `powheg_me/` Fortran layout).

## The file set

```
code/process/<NAME>/
  flame_<sfx>.cpp              ~10 lines, the entry point
  process_<sfx>.hh/.cc         the Process subclass and its factories
  scales_<sfx>.hh/.cc          ScaleManager subclass
  selectors_<sfx>.hh           PosteriorCut subclasses + PosteriorCuts
  suppression_factors_<sfx>.hh
  user_section_<sfx>.hh        the [User] runcard section
  matrix_element_<a>.hh        Born-level ME       -> matrix_element_0j
  matrix_element_<b>.hh        one-extra-parton ME -> matrix_element_1j
  analyses_<sfx>/              at least one analysis .hh/.cc
  CMakeLists.txt
  powheg_me/                   optional, for Fortran MEs (VJ, dijet, hvq)
```

The entry point is the whole executable:

```cpp
#include "Main/flame.hh"
#include "process_<sfx>.hh"

int main(int argc, char *argv[]) {
  return FLAME::flame<FLAME::Process<Name>>(argc, argv);
}
```

## The contract

`code/lib/include/Process/process.hh`. **Seven pure virtuals** — the class will not
compile without all of them:

| Member | Signature |
| --- | --- |
| `get_process_name` | `std::string get_process_name() const override` |
| `get_n_alphas_born` | `unsigned int get_n_alphas_born() const override` |
| `get_allowed_scale_choices` | `std::vector<std::string> get_allowed_scale_choices() const override` |
| `create_scales` | `std::shared_ptr<ScaleManager> create_scales() const override` |
| `build_integration_channels` | `IntegrationChannels build_integration_channels(std::shared_ptr<ParticleManager>, std::shared_ptr<GenerationCuts> = nullptr, std::shared_ptr<EWParameters> = nullptr) override` |
| `initialise` | `ProcessObjects initialise(const IntegrationChannels &, const IntegrationChannels &regular, const PhysObjects &, const PhysObjectsProcdep &) override` |
| `check_consistency_beams` | `bool check_consistency_beams(const Beams &) const override` |

Optional overrides with working defaults — add only those the process needs:
`create_selectors`, `create_generation_cuts`, `create_suppression_factors`,
`create_analysis_manager`, `create_user_section`, `apply_user_section`,
`create_remnant_factor`, `build_integration_channels_regular`,
`get_light_particle_defaults`, and the FKS cut accessors (`get_xi_cut_UV`,
`get_y_cut_UV_isr`/`_fsr`, `get_xi_cut_IR_isr`/`_fsr`, `get_y_cut_IR_isr`/`_fsr`,
`get_max_nq_collinear`).

`initialise()` returns a `ProcessObjects` that must be populated with
`phase_space_btilde`, `matrix_element_0j`, `matrix_element_1j` and
`analysis_manager` (plus `phase_space_remnant` / `phase_space_regular` if used).

## How to fill the stubs

**Generate structure, not physics.** Every member whose body depends on the process —
the channel construction, the matrix elements, the scale definitions, the cuts, the
beam check — gets a compiling stub and an explicit marker saying what the user must
supply:

```cpp
unsigned int get_n_alphas_born() const override {
  // TODO(user): number of powers of alpha_s in the Born matrix element.
  //   0 for a colour-singlet final state (DY), 2 for dijet production.
  return 0;
}
```

```cpp
IntegrationChannels build_integration_channels(
    std::shared_ptr<ParticleManager> particles,
    std::shared_ptr<GenerationCuts> generation_cuts,
    std::shared_ptr<EWParameters> ew_parameters) override {
  // TODO(user): declare one integration channel per Born-level subprocess.
  //   See ProcessDY::build_integration_channels in process_dy.cc for the
  //   pattern, and Process2J for a QCD final state.
  throw std::runtime_error("build_integration_channels not implemented for <NAME>");
}
```

Rules for the markers:

- Say **what the value means and how to decide it**, not just "fill me in".
- Point at the live process that shows the pattern, by file and symbol.
- Stubs that cannot return a sensible default `throw`, so an unfinished process fails
  loudly at startup rather than producing a wrong number silently.
- Never guess a physics value. A plausible-looking wrong `get_n_alphas_born()` is worse
  than a throw.

The parts that *are* mechanical — the entry point, the header guards, the class
skeleton, the `[User]` section plumbing, the `CMakeLists.txt` — should be complete and
correct, not stubbed.

## CMakeLists

Copy from the closest live process and change only the source list and the target
name. It is ~300 lines of `FLAME_LIB_MODE` (`AUTO`/`INSTALLED`/`BUNDLED`), FastJet
detection, OpenLoops wiring, RPATH handling and the workflow symlink install — all of
it identical across processes, and none of it worth rewriting.

What changes:

```cmake
set(PROCESS_<SFX>_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/process_<sfx>.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/scales_<sfx>.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/analyses_<sfx>/<analysis>.cc
)
```

The executable target is called `flame_exe` in **every** process — do not rename it;
the regression runner and `run_parallel_stages.py` both look for that name. Every
analysis `.cc` must be listed, or you get a link error. Fortran MEs go in the same list
with `LINKER_LANGUAGE Fortran` and `-ffixed-line-length-none`.

## Verify

The scaffold must build before it is handed over:

```bash
code/scripts/build_process.sh --proc-src code/process/<NAME> --jobs 8
```

Then confirm the runcard template generates, which exercises the process-dependent
runcard sections (scale choices, cut names, `[User]` keys):

```bash
cd code/process/<NAME> && ./build/bin/flame_exe -w && head -30 default.run
```

A scaffold that compiles and can emit its own runcard is done. It will `throw` on an
actual run until the user fills the stubs — that is the intended state, and say so
when handing it over.

Related: `set-runcard` for filling the generated template, `add-observable` for the
analysis, `build` for the build options.
