---
name: fix-build
description: Minimally repair a FLAME build that the user broke while editing — compile and link errors only, never their logic. Use when the user pastes a compiler error, says the code does not compile / does not build / is broken, asks to "fix the build", "fix the errors", "make it compile", asks for a list of fixes after their own edits, or opens a session saying they will keep editing while you patch up what breaks. Use it whenever an error message arrives with no other instruction — the default reading of a pasted error is "make this go away without changing what I wrote".
---

# Minimal build repair

The user is mid-refactoring. They own the design and the physics; you own only the
mechanical residue their edits leave behind — a definition whose declaration moved,
a body pasted outside its namespace, a variable that lost its type. Your job is to
get `flamelib` compiling again having changed as little as possible, and to say
clearly what you touched.

The instinct to fight here is the helpful tidy-up. A file you are already editing
will be full of things you would do differently. Leave them. A refactoring in
progress looks wrong from the outside all the time, and the user can only keep
their bearings if the diff between builds is theirs plus your handful of one-liners.

## What counts as yours to fix

Fix it when the code cannot express what it plainly means to express:

- A definition with no matching declaration (a virtual whose signature changed in the
  `.cc` but not the `.hh`, or vice versa)
- A body at global scope that needs `namespace FLAME { ... }`
- A missing `using namespace MathConstants;` / `using namespace MathFunctions;`, or a
  missing `#include`
- A stale identifier after a move or rename — a parameter still called by its old name,
  a member qualified with the class it used to live in
- A declaration missing its type, a missing `const`/`override`, an unused-parameter
  warning that is an error here

Flag it, and leave it, when the code compiles but says something surprising:

- A formula that changed shape, a sign, a factor, a different invariant
- A `throw`-stub that disappeared, a hook that is now called twice or from a new place
- Anything where the mechanical fix would require you to guess which of two
  behaviours they meant

That second list is the valuable half of the job. The user is moving fast and cannot
see the whole blast radius; a one-line "this now overwrites `_phi` on the other call
path — deliberate?" is worth more than the fix you did make. But raise it as a
question at the end, not as an edit.

## Loop

Build incrementally — the build directory usually already exists, so do not
reconfigure:

```bash
cmake --build code/lib/build -j8 --target flamelib 2>&1 | grep -E "error|\] Built target"
```

`code/lib/build` is the usual location; check first, and fall back to
`code/scripts/build_process.sh` only if the error is on the process side. Filtering
to `error` matters because this tree emits a deliberate `#warning` in
`PhaseSpace/phase_space.hh` that scrolls the real problems off the top.

Then, before editing:

**Read the actual file, not just the error text.** g++ reports the first thing that
confused it, which in this codebase is routinely the *symptom*. Seventeen
"was not declared in this scope" errors in one file almost always mean one missing
`#include` or one missing namespace brace, not seventeen problems. Fixing them
one at a time is how a five-line change becomes a fifty-line one.

Fix, rebuild, repeat until it builds clean. Then report.

## Reporting

A short numbered list, one line each, each with a `file:line` link. Say what the
error *was*, not just what you typed. Then, separately, the things you flagged and
did not touch.

Keep it proportional: three fixes get three lines, not three paragraphs. The user
is going straight back to editing.

If the user says "do not edit, just list" — or is clearly still mid-thought — give
them the same list as a plan and stop. Some of the time they want to apply the fixes
themselves so they stay in the flow of their own refactoring.

## House rules that bite here

- **No `auto`.** When a declaration is missing its type, write the real one
  (`const LorentzFourVector<double> p = ...`), never `auto`.
- **No git.** Never commit, stage, or push, even when a change is finished and clean.
- **`code/` only.** `code_DEPRECATED_DONOTUSE/` and `toy-code/` are dead copies.
- Keep the surrounding formatting. If the file has four-space bodies inside a
  two-space class, match what is there — a reformat buried in a bug fix is noise
  in their next diff.
- Do not propose deleting `map_real_to_born` / `project_to_born` or other code that
  looks unreachable. Scaffolding for work in progress is the normal state of this tree.

## Verifying beyond the compiler

A clean build is the bar for this task, and usually the whole of it. Offer more only
when the change plausibly moved numbers — a changed invariant, a reordered
computation, an altered mapping:

- `ctest --test-dir code/lib/build -R test_fks_mappings` for the FKS tree
- `code/regression_test/run_regression.py --test --process DY` for cross sections

Offer; do not run them unprompted. A regression run is minutes the user did not ask
for, and mid-refactoring it is expected to fail for reasons that are not your fixes.
