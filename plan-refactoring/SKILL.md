---
name: plan-refactoring
description: Run a design-only FLAME refactoring through the NOTES to PLAN to REPORT document lifecycle, where the transformation is algebraic and the physics must come out bit-for-bit unchanged. Use when asked to plan or carry out a refactoring, restructure a class family, reorganise code without changing results, write a refactoring plan, or record what a refactoring actually changed.
---

# Planning a design-only refactoring

Refactoring in FLAME means **changing the design, never the physics**. The
transformation is algebraic: code moves, is renamed, is split or merged, but every
cross section and every histogram bin must come out bit-for-bit identical. Anything
that changes a number is not a refactoring — it is a physics change and belongs in its
own branch with its own justification.

Three documents, in order. Each has a different job and none substitutes for another.

## NOTES — the design session

Written **during plan mode**, before anything is applied. It records the reasoning: what
the current structure is, what is wrong with it, which options were weighed, and why the
chosen one won. Open it by saying plainly that nothing here is applied.

Contents:

- **Scope** — what is in, what is out, and the tree scope (`code/` only;
  `code_DEPRECATED_DONOTUSE/` and `toy-code/` are stale copies, never edited or cited).
- **The current structure**, read from the code with `file:line` references.
- **Numbered decisions** (`D1`, `D2`, …) so the plan can cite them instead of
  re-arguing them.
- **What was deferred** and to where.

Existing examples: `NOTES.md` (SectorFunction / RealManager), `NOTES2.md` (the
`RealMapping` family).

## PLAN — the executable specification

Derived from the NOTES, it divides the work into numbered steps. It assumes the NOTES'
decisions rather than restating them. State up front that nothing is applied yet.

Contents:

- **Scope**, with an explicit out-of-scope table giving a reason and a destination for
  each excluded item — dead-code removal, performance work, runcard exposure, renames
  all tend to be deferred, and writing down where they went stops them leaking back in.
- **Target structure** — the class tree or file layout after the change.
- **A guiding principle.** `PLAN2.md`'s is worth reusing: *do not speculatively hoist.*
  Code moves up into an abstract base only when it is common by construction; anything
  whose commonality cannot be established today stays where it is and is lifted later
  when a second implementation exists to compare against. Pushing code up later is
  cheap; guessing wrong now is not.
- **Numbered steps** (`A1`, `A2`, …), each small enough to verify on its own, grouped
  into phases with a **gate** at the end of each.
- **A verification section** — see below.

## REPORT — what was actually applied

Written as the work proceeds and updated **at every gate**, not at the end. It records
what was applied step by step and **every deviation from the plan**, with the reason.
A plan that was followed exactly still needs a report: the verification numbers live in
it.

Open with a header table: branch, starting commit, date, scope, which baseline tarball
was used. Then one section per gate, each carrying the commands run and the results as
a table of actual numbers — not "tests passed".

Existing example: `REPORT2.md`.

## Standing ground rules

State these in every plan, and hold to them:

1. **Behaviour-preserving.** The bar is *numerically identical results*, not "the tests
   still pass".
2. **Physics moved verbatim.** When an expression moves between files, it is moved —
   copied character for character — never retyped, re-derived, simplified, or
   "cleaned up in passing". A re-typed formula is a physics change wearing a
   refactoring's clothes.
3. **Bit-identical regression after every step.** `rel_diff = 0.00e+00`, not "within
   tolerance".
4. **Never re-baseline to hide a change.** If the numbers move, find out why.
   Regenerating the baseline to make a diff go away destroys the only evidence.
5. **No `auto`** except where the type genuinely cannot be named.
6. **Design only.** No new physics, no dead-code removal, no performance work smuggled
   in alongside. Each of those is its own session.
7. Follow `GUIDELINES.md` (the `check-conventions` skill).

## The verification section

Every plan needs one, and it has a shape:

**Before touching anything**, record the baseline — including what already fails.

```bash
cd code/regression_test && python3 run_regression.py --test --verbose -j 4
```

```bash
cmake -S code/lib -B code/lib/build -DENABLE_TESTS=ON && cmake --build code/lib/build -j && ctest --test-dir code/lib/build
```

Write the actual cross sections into the report. A pre-existing failure that is not
recorded before the work becomes indistinguishable from one the refactoring caused.

**At each gate:**

1. Build the library and every live process — `DY`, `VJ`, `ggH`, `dijet`, `epemjj`,
   `hvq` — via `code/scripts/build_process.sh --proc-src code/process/<P> --jobs 8`.
2. Regression suite against the recorded baseline: same passes, same numbers.
3. **Both caching configurations.** Build once with `DISABLE_EQUIV`,
   `DISABLE_CACHE_EQUIV` and `DISABLE_CACHE_PHSP_REAL` set. These paths call the code
   directly instead of through the caches, so a refactoring can break one and not the
   other — and the disabled-cache build is the cross-check that makes the cache
   trustworthy at all.
4. **The staged path** as well as all-in-one, when grids or integrands are touched:
   `python3 run_regression.py --test --mode staged`.
5. Anything with a single consumer that the regression suite does not exercise must be
   run explicitly and named in the plan.

**If a difference appears**, stop and bisect the steps rather than continuing. The
plan's step numbering exists precisely so that the failing step can be named.

## Practical notes

- `**/*.md` is in `.gitignore`, so `NOTES*.md`, `PLAN*.md` and `REPORT*.md` are
  **untracked and invisible to collaborators** — they never show in `git diff` and are
  lost on a fresh clone. Treat them as local working documents and share them
  deliberately.
- Work on a branch named `YYYY-MM-brief-description`, never on `main`, and merge `main`
  in often.
- Baselines are tarballs named after the commit they were made at; unpack the one
  matching the pre-refactoring state and keep it untouched for the duration (the
  `regression-test` skill).
