# CLAUDE.md — ParamIO.jl

**Layer 1 of the infra HPC stack** (ParamIO → DataVault → SweepRunner):
config TOML → `Vector{DataKey}`. Pure parsing + Cartesian enumeration; no IO, no
storage. See [`../CLAUDE.md`](../CLAUDE.md) for how the three layers fit together.

## Role / public API

- `load(path) -> ConfigSpec` — parse TOML (merges `[base] inherit` if present).
- `expand(spec) -> Vector{DataKey}` — Cartesian product of list-valued params × `total_samples`.
- `format_path(key, path_keys) -> String` — legacy compact path; floats use two decimals.
- `format_path(key, path_keys, axis_formats) -> String` — content-aware float path using `build_axis_formats`.
- `build_axis_formats(spec) -> Dict{String,AxisFloatFmt}` — formats distinct float values per path axis without collisions.
- `diagnose(spec) -> DiagnosticReport` — non-writing preflight checks for duplicate points, path collisions, invisible params and ambiguous keys.
- `canonical(key) -> String` — stable, order-/version-independent key identity.
- `param(key, name[, T])` — one parameter, resolved dotted-or-leaf; with `T` it refuses a
  conversion that would change the number.

## Two facts that trip up callers

- **`DataKey.params` keys are DOTTED**: a `[paramsets.system] N=…` block →
  `key.params["system.N"]`, not `["N"]`. Fixed scalars still appear in every key.
- **List value ⇒ swept axis; scalar ⇒ fixed.** Only lists create the product.
- **A `{start, stop, length|step}` table is a grid** — a concise swept axis that expands to a
  list before the product (`length`→linspace `Float64`, integer `step`→`a:s:b` `Int`,
  `scale="log"`→geometric). A namespace that merely *contains* a `start`/`stop` param (plus, say,
  `dt`) is **not** a grid. See README's *Grid axes*.
- **A `{const = X}` table is a fixed value** — the explicit dual of `list ⇒ sweep`: it pins `X`
  (even a list) as one value, never swept. `J = {const = [1, 0.5]}` is one 2-vector; `J = [1, 0.5]`
  is two runs. Implemented as a `_Literal` marker that `expand` unwraps.

## Where to look for usage

- **`examples/inspect.jl`** — run it; prints exactly what `expand` produces (dotted keys, `format_path`, `canonical`). Start here.
- `README.md` — config schema + API.
- Full stack: [`../SweepRunner.jl/examples/`](../SweepRunner.jl/examples/).

## Float path formatting

`[datavault] float_format` defaults to `"fixed2"`. `"auto"` formats distinct float values
losslessly and without collisions within each path axis for the current sweep. Its precision is
chosen from current values, so changing the sweep can change directory names; `canonical()` is
unaffected. Pass `build_axis_formats(spec)` to the three-argument `format_path` in auto mode.

## Source layout

- `src/ParamIO.jl`: exports and include order.
- `src/core/`: `types`, `load`, `expand`, `project`, `artifacts`, `format`, `param`, `canonical`, `diagnose`.
- `src/util/`: `grid`, `flatten`, `resolve`, `path_keys`.

## Invariant when changing this package

**`canonical()`'s output schema is FROZEN** — `SweepRunner`'s `Manifest` uses it
as a directory-safe identity. Changing its format silently orphans every
existing manifest. Run the test suite locally before pushing.
