# Git and sharing

From `GUIDELINES.md`, "Collaborating and sharing" and "Coding conventions".

## Rules

- **Never work on the master/main branch.** Use a dedicated branch and ask to merge once
  it is mature. Delicate work that may break things especially belongs on a branch.
- Branch names take the form **`YYYY-MM-brief-description`**.
- Merge `main` into the branch **as often as possible** unless there is a very good
  reason not to — it keeps conflicts tractable.
- Commit key result files, plot scripts and plots. **Do not commit large files.** If
  large files become necessary, the method (git-lfs or otherwise) is a collaboration
  decision, not an individual one.
- Results and work in progress are not shared outside the collaboration until arXiv.
  Exceptions (job talks, conferences shortly before release) are discussed collectively.

## How to check

```bash
git rev-parse --abbrev-ref HEAD
```

Branch name should match `^\d{4}-\d{2}-[a-z0-9-]+$`. Refuse to commit on `main`.

```bash
git log --oneline main..HEAD --stat | grep -E "\|\s+[0-9]{4,} " 
```

flags files with thousands of changed lines — usually a committed binary, log, or PDF.

## Known state

Branch compliance is roughly 8 of 11. Deviations in use: `heavy-refactoring` (no date
prefix), `origin/2026-07-27/Nicola/Asteria-CMake` (slash-separated with an author
segment), and casing drift (`2026-06-03-Remnants`, `2026-08-15-MECaching`,
`2026-07-26-AsteriaCMake` against the lowercase-dashed norm).

Commit practice is undocumented in `GUIDELINES.md` but consistent in the history:
collaborator initials go in the **subject line** — `with AG:`, `with DL+AG:`,
`Huge refactoring with Claude, AG, DL, MR.` — not in a `Co-Authored-By:` trailer, which
is not used in this repository. Subjects are short and imperative-ish, with no prefix,
scope or issue-ID convention. Frequent `Merge branch 'main' into <branch>` commits,
consistent with the merge-often rule.

There are **no git tags, no releases and no CHANGELOG**. The version is only
`project(FLAMElib VERSION 0.1.0)` in `code/lib/CMakeLists.txt:2`; provenance at runtime
is the git hash, compiled in via `code/lib/version.h.in` and printed at startup. The
de-facto release gate is the regression suite, run by hand — GitLab CI exists but is
switched off (`CI_ENABLED: "false"`).

Note `.gitignore` contains `**/*.md`, so every Markdown working document — `PLAN*.md`,
`NOTES*.md`, `REPORT*.md`, and the missing `3rdPartyCode.md` — is invisible to
collaborators. Only `GUIDELINES.md`, `README.md` and `ONGOING_PROJECTS.md` are tracked,
having been added before that rule.
