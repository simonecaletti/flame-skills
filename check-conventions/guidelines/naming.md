# Naming

From `GUIDELINES.md`, "Coding conventions".

## Rules

- Source files are named `.cc` and `.hh`. Not `.cpp`/`.h` — with the sole exception of
  the process entry points (`flame_dy.cpp`, `flame_vj.cpp`, …) and the unit tests
  (`test_*.cpp`), which are established as `.cpp`.
- Class names are `CamelCase`.
- Function and variable names use `snake_case`.
- Class members must be distinguishable from local variables. The convention for newly
  written code is a **leading underscore for class members**.
- Private and protected class members **including methods** take a leading underscore
  (`_variable_name`, `_function_name`). Public variables and methods take none.

## How to check

The underscore rule is the one that is actually violated. For a touched header, check
that every member under `private:`/`protected:` starts with `_`, and that nothing under
`public:` does:

```bash
awk '/^\s*(public|private|protected):/{s=$1} /\(|;/{print s, FILENAME":"FNR": "$0}' path/to/file.hh | grep -E "private:|protected:" | grep -vE "\s_[a-z]"
```

Treat that as a hint, not a verdict — it does not parse C++. Read the header.

## Known state

Consistently followed in `FKS/` and `RealInfrastructure/`. Older headers are mixed.
Only flag violations introduced by the change under review.
