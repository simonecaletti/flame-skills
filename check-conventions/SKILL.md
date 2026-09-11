---
name: check-conventions
description: Check FLAME code against the project conventions in GUIDELINES.md — naming, underscores, doxygen, the CachingSystem single-access-path rule, logbook citations, borrowed-code marking, and branch naming. Use when asked to review a diff or file for style, check conventions before a merge, verify naming or the cache rule, or when writing new code that must match the house style.
---

# Checking FLAME conventions

The rules live in `guidelines/` next to this file, one file per category. **Read the
relevant category file before judging code against it** — the rules there are the
authority, not this summary, and they are updated as conventions evolve.

| File | Category |
| --- | --- |
| `guidelines/naming.md` | file extensions, CamelCase/snake_case, leading underscores |
| `guidelines/documentation.md` | doxygen comments |
| `guidelines/caching.md` | the CachingSystem single-access-path rule |
| `guidelines/physics.md` | logbook citations, borrowed code and copyright |
| `guidelines/git.md` | branches, merging, what may be committed |

Source of truth upstream is `GUIDELINES.md` at the repo root. When it changes, update
the corresponding file here rather than restating the change in prose.

## How to run a check

Scope to what actually changed — a whole-tree sweep produces hundreds of pre-existing
violations and buries the ones that matter.

```bash
git diff --name-only main...HEAD -- '*.cc' '*.hh'
```

```bash
git diff -U0 main...HEAD -- '*.cc' '*.hh'
```

For each touched file, read the category files that apply and report violations as
`file:line` with the rule that is broken and the minimal fix. Do not reformat, rename,
or "tidy" anything that was not part of the change under review.

## Reporting

Separate **introduced** from **pre-existing**. A file that already violated a rule
before the diff is not this change's problem; say so and move on. The tree is
inconsistent by area — `FKS/` and `RealInfrastructure/` follow the conventions closely
because they were recently refactored, older code much less — so a strict reading of
the whole file will mostly produce noise.

Order findings by consequence, not by rule number:

1. **Cache rule violations** — these silently break the `DISABLE_CACHE_*` cross-check
   and can let physics drift between two code paths. Always report these first.
2. **Missing logbook citation on new physics code** — unrecoverable knowledge loss once
   the author forgets.
3. **Borrowed code without origin and copyright** — a licensing problem, not a style one.
4. Naming and doxygen — real but cosmetic.

State plainly when a file is clean. Do not invent findings to fill a report.
