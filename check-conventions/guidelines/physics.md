# Physics code: logbook citations and borrowed code

From `GUIDELINES.md`, "Coding conventions".

## Logbook citation

> if a piece of code implements some specific piece of physics, equation, etc., refer
> explicitly to the logbook or other document where this is explained and specified
> (and specify which variants / equations are being used).

Both halves matter: the **entry** and the **which equation**. A bare directory name is
half a citation.

Entries live in `logbook/`, one directory per topic, named `YYYY-MM-DD-Topic-In-Dashes`
(note the day — branches use `YYYY-MM` without it). Each holds a LaTeX master file named
after the directory, plus optional chapter `.tex` files.

The established citation style is a doxygen comment naming the directory and, when the
derivation spans several files, the sub-`.tex`:

```cpp
/// See logbook/2026-02-26-New-FKS-map.
```

```cpp
/**
 * Altarelli-Parisi variable z, see logbook/2026-02-26-New-FKS-map and
 * logbook/2026-04-20-FKS-Subtracted-Real (unintegrated_ctr.tex).
 */
```

### Known state

Only **nine** headers in the live tree cite the logbook, all under `FKS/` and
`RealInfrastructure/`: `subtraction_scheme_enumerate.hh`, `fks_phase_space_real.hh`,
`real_mappings_FKS.hh`, `real_mappings_FKS_isr_plain.hh`,
`real_mappings_FKS_fsr_plain.hh`, `fks_sector_function.hh`, `fks_sector_distances.hh`,
`fks_real_table_builder.hh`, `fks_cuts.hh`. The convention is followed in the recently
refactored area and nowhere else — so require it on **new** physics code, and do not
flag the absence across untouched older files.

## Borrowed code

> if we borrow code from somewhere else, in the code itself, we mark very explicitly
> where it's borrowed from and with what copyright. It's probably a good idea to inform
> everyone about it … and we add it to the file `3rdPartyCode.md`

Three obligations: an **in-place marker**, a **copyright statement**, and an entry in
`3rdPartyCode.md`.

### Known state — all three are partly unmet

`3rdPartyCode.md` **does not exist and never has** (`git log --all` for it is empty),
despite being referenced by both `GUIDELINES.md` and `CLAUDE.md`. It would also be
gitignored by `**/*.md` if created. What exists instead is
`3rdPartyPermissions/`, one `YYYY-MM-Topic.txt` per permission — currently a single
placeholder file.

In-place markers exist only in the Fortran matrix elements, as free-text comments with
no fixed template:

- `code/process/dijet/powheg_me/virtual.f:92` — "taken from Ellis-Sexton code"
- `code/process/dijet/powheg_me/Born.f:219` — Kunszt & Soper, Phys.Rev.D46,192
- `code/process/VJ/powheg_me/{Born_cc.f,virtual_cc.f,real_cc.f}:1` — "adapted from"
- `code/process/VJ/powheg_me/virtual_nc.f:874`, `virtual_cc.f:924` — Duplancic & Nizic,
  hep-ph/0006249v2

**None carries a copyright statement**, so the "with what copyright" half of the rule is
unmet everywhere it applies. Report this for newly borrowed code; for the existing
blocks, it is a known debt, not a finding against the current change.
