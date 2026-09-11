# Documentation

From `GUIDELINES.md`, "Coding conventions".

## Rule

> classes, and any non-trivial function, should have doxygen-style comments (`///`)
> explaining what they are intended to do

## How to check

For each new or substantially rewritten class or non-trivial function in the diff,
confirm there is a comment above it saying **what it is for**, not what it does
line-by-line. A one-line `/// @brief` is enough for a small function; a class deserves
a sentence of purpose.

## Known state — read before reporting

The tree does not settle on one comment syntax: roughly 557 `///` lines against 836
`/** … */` blocks in `code/lib/include`, and around twenty headers carry neither,
including `Process/process.hh`, `PhaseSpace/multichanneling.hh` and `Beams/beams.hh`.

There is **no Doxyfile anywhere in `code/`** and no doc-generation step — the only
Doxyfiles in the repository belong to the `kakuhen` and `hoppet` submodules. So this is
a readability convention with no build-time enforcement and no generated output.

Practical consequence: `/** … */` on new code is not a violation worth reporting, since
it is the majority style in the file it sits next to. **Missing** documentation on a new
public class or a non-obvious function is worth reporting. Match the file you are in.
