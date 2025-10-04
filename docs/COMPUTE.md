# Compute & Rhai

Inline evaluation for guards/outputs.

- **Directives:** `#compute(...)` (built-ins), `#rhai("...script...", args...)` (sandboxed Rhai).
- **Where:** inside `guard <term>` and output terms.
- **Ground-only:** all args must be constants after substitution.
- **Result type:** one constant: `int | dec | bool | string`.

## Semantics

- **Order:** substitute vars → evaluate (`#compute` / `#rhai`) → algebra normalize (where applicable).
- **Guards:** result must normalize to the constant `true`. Any error/non-ground ⇒ guard is `false`.
- **Outputs:** on error or non-ground, apply transition grounding:

  - `strict` ⇒ step fails,
  - `skip` ⇒ drop this output,
  - `default("v")` ⇒ use `"v"` (must be a simple string; no `? , ( )`).

---

## `#compute(...)` — built-ins

- Integers are 64-bit; decimals are fixed-precision. Overflow/div-by-zero are errors.
- Numeric functions accept both `int` and `dec`; mixed operations promote `int` to `dec`.
- Functions (arity):

**Math:**
`add(a,b)`, `sub(a,b)`, `mul(a,b)`, `div(a,b)`, `mod(a,b)`, `abs(x)`, `min(a,b)`, `max(a,b)`, `clamp(x,lo,hi)` (requires `lo ≤ hi`)

**Compare (→ bool):**
`gt(a,b)`, `ge(a,b)`, `lt(a,b)`, `le(a,b)`, `eq(a,b)`, `ne(a,b)`, `between(x,lo,hi)` (inclusive, `lo ≤ hi`)

**Bool:**
`and(a,b)`, `or(a,b)`, `not(a)`

**Strings:**
`concat(a,b)`, `len(s)` (UTF-8 chars)

**Example**

```relog
guard #compute(gt(#compute(add(2, 5)), 6))
out res(val(#compute(sub(10, #compute(add(4, 3))))))
```

**Sample**: [samples/compute.rl](../samples/compute.rl)

---

## `#rhai("...script...", args...)` — sandboxed script

- Signature: first arg is a string script; extra args are ground constants.
- Inside script: args available as `args` (array).
  Mapping: int→INT, dec→STRING (canonical form), bool→BOOL, string→STRING.
- Must return INT/BOOL/STRING.

**Sandbox profile**

- No I/O/time/random registered. `print`/`debug` are ignored.
- Limits: `max_operations = 20_000`, `max_call_levels = 32`, strict variables.

**Decimal arithmetic**

Rhai doesn't have native decimal types, so decimal values are passed as strings in canonical form (e.g., `"1.5"`, `"0.0"`).
Use these helper functions for decimal operations:

- **Math:** `dec_add(a,b)`, `dec_sub(a,b)`, `dec_mul(a,b)`, `dec_div(a,b)`, `dec_abs(x)`, `dec_min(a,b)`, `dec_max(a,b)`
- **Compare:** `dec_gt(a,b)`, `dec_ge(a,b)`, `dec_lt(a,b)`, `dec_le(a,b)`

All helpers accept and return strings. Example:

```relog
guard #rhai("dec_gt(args[0], \"10.5\")", let price)
out total(#rhai("dec_add(args[0], args[1])", let a, let b))
```

**Quoting**

- Strings use `"` with standard escapes: `\" \\ \n \t \u1234 \u{1F600}`.

**Example**

```relog
guard #rhai("let s = args[0]; s.len() >= 3", "alex")
out   out(val(#rhai("let n = args[0]; format!(\"user:{}\", n)", "ann")))
```

**Sample**: [samples/rhai.rl](../samples/rhai.rl)

---

## Rules

- Arguments must be **ground**; otherwise:

  - in guards ⇒ **false**,
  - in outputs ⇒ **grounding policy** applies.

- Only `int | dec | bool | string` results are accepted.
- Algebra can rewrite results; guards must end up as the constant `true`.
- `#compute` is pure/deterministic.
- `#rhai` runs in a sandbox; no external effects; fuel-limited.
- Keep scripts small; they execute per firing attempt.
- `_` is an anonymous variable in patterns; use `"_"` if you need a literal underscore token.
