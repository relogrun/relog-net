# Changelog

## [0.6.1] — 2025-10-04

### Added

- **Decimal type (`dec`)**
  - Fixed-precision decimals: `1.5`, `-0.125`, `1e-3`, `-2.0E+10`. Subtyping: `int ⊆ dec`.
  - `#compute` functions accept mixed `int`/`dec` (auto-promote to `dec`).
  - `#rhai` decimal helpers: `dec_add`, `dec_sub`, `dec_mul`, `dec_div`, `dec_gt`, `dec_ge`, `dec_lt`, `dec_le`, `dec_abs`, `dec_min`, `dec_max`.
  - 
## [0.6.0] — 2025-10-03

### Added

- **Compute directives**
  - `#compute(...)`: pure built-ins for `int | bool | string` (arithmetic, comparisons, booleans, string ops like `concat`, `len`, `clamp`, `between`).
  - `#rhai("...script...", args...)`: inline Rhai script in a sandboxed engine, arguments exposed as `args` (array), returns `int | bool | string`.

## [0.5.0] — 2025-09-26

### Changed

- **Runtime semantics:** pivot to a **Markov** scheduler. Each tick fires **exactly one** enabled instance. Deterministic tie-break: higher `priority` → transition `name` (asc) → `instance_id` (asc) → `id` (asc).

### Breaking

- **Removed DSL/runtime directives:** `forward` and `multi_firing`.

## [0.4.0] — 2025-09-13

### Added

- **Types (optional)**

  - Store type annotations: `store buf: item<sym>`, `store done: pair<int,int>`.
  - Base types: `any | sym | int | bool`; type constructors: `Ctor<T1,...,Tn>`.
  - Static checks for `init`, `in`, `out` terms.

- **Algebra DSL (optional)**

  - New `algebra { ... }` block with **operators** (`operator f { assoc, comm, id(term), rest(let r) }`) and **rewrite rules** (`rule lhs => rhs;`).
  - **Configs:** `max_steps N` and `ac_branch_budget N`.

- **Transition guards**

  - `guard <term>` inside a transition.

## [0.3.0] — 2025-08-31

### Added

- **New `serve` mode**: built-in HTTP + WebSocket API to run/control `.rl` files.

  - WS: `/api` streams `{schema,type,payload}` (`status` | `event` | `error`).
  - HTTP: `POST /api/start`, `/stop`, `/pause`, `/resume`, `/step`, `/set_delay`; `GET /api/status`, `/file?path=<rel>`, `/files`, `/net?path=<rel>`; `POST /api/save`.
  - Flags: `--port <P>`, `--autostart <rel-file.rl>`, `--log <level>`.

### Changed

- **CLI split into subcommands**:

  - `run` — execute a single `.rl` locally:
    `relog-cli run <file.rl> [--log <level>] [--runtime {natural|reactive}] [--max_ticks N] [--delay MS]`
  - `serve` — start the API server for a base directory or a file inside it:
    `relog-cli serve <base-path-or-file> [--port P] [--autostart rel-file.rl] [--log <level>]`

### Breaking

- You must specify a subcommand (`run` or `serve`). Invoking the binary without a subcommand is no longer supported.

## [0.2.0] — 2025-08-30

### Added

- Wildcards: anonymous variable `let _` (matches anything, unbound).
- Net/runtime config in DSL: `forward first_fit | max_cardinality(N)`, `multi_firing unlimited | limited(N) | disabled`, `runtime natural | reactive`, `max_ticks N`, `delay N`.
- Transition config: `grounding strict | skip | default("v")`, `priority N`.
- Store config: `capacity unbounded | N` with capacity checks.
- CLI flags: `--runtime {natural|reactive}`, `--max_ticks N`.

### Optimizations

- Faster instance search (first-fit), fewer allocations in substitution/unification, cheaper atomic-step validation.

## [0.1.0] — 2025-08-24

### Added

- First public preview: toy no-I/O CLI for Relog DSL & algebraic token nets.
- CLI flags: `--help`, `--version`, `--log`, `--delay <ms>` (between successful ticks).
- Runtime: stores, transitions, input/output arcs, multiplicity, greedy concurrent firing.
- DSL basics: `store`, `transition { in/out }`, `init`, terms (`let x`, literals, `foo(a,b)`).
- Builds: macOS universal, Linux x86_64 (musl), Windows x86_64 (gnu).
- Bundled: `samples/`, `LICENSE.md`, `README.md`, `checksums.txt`.

### Known limitations

- No external I/O, persistence, or debugger; `trace` logs are very verbose.
