# Relog DSL

## Syntax

### Terms

- **Variables:** `let x`. Reusing a var enforces equality, e.g. `pair(let x, let x)`.
- **Literals:** identifiers (`hello`), strings (`"hello world"`), numbers (`123`, `1.5`, `-2.0E+10`).
  - **Strings** use double quotes and standard escapes: `\"`, `\\`, `\n`, `\t`, `\u1234`, `\u{1F600}`.
  - **Integers:** `[-+]?[0-9]+`
  - **Decimals:** either plain decimal `[-+]?[0-9]+"."?[0-9]+` or scientific notation  
    `[-+]? <mantissa> ("e"|"E") [+-]? <exponent>`, where `<mantissa>` is e.g. `123.45` or `123`.  
    Examples: `1.5`, `-0.125`, `1e-3`, `-2.0E+10`.
  - The parser/marking **preserves the original literal spelling** (e.g., `1e-3` is not rewritten to `0.001`).
- **Applications:** n-ary terms, e.g. `foo(bar, baz)`.
- **Multiplicity:** `* N`. Inputs need **N distinct** matching tokens. Outputs produce **N** tokens. Examples:
  - `in buffer(let x) * 3` — consumes 3 distinct tokens
  - `out done(result) * 2` — produces 2 tokens
  - `init { free slot * 3 }` — creates 3 initial tokens
- **Guards:** `guard <term>` after input matching; the term is algebra-normalized.
  The transition fires only if it becomes `true` (multiple guards allowed).

### Arc modes

Input arcs support:

- **consume** (default) — removes the token
- **read** — keeps the token (non-destructive read)
- **inhib** — transition enabled only if token is **absent**

Output arcs support:

- **consume** (default) — adds the token normally
- **reset** — clears the entire store first, then adds tokens

### Grounding policy

When output terms contain unbound variables after substitution:

- **strict** (default) — step fails with error
- **skip** — drop this output arc silently
- **default("value")** — substitute the literal string value (no special chars: `? , ( )`)

### Priority

When multiple transitions are enabled, highest `priority` fires first (default: 0). Ties break by name, then instance order.

### Algebra & Types (experimental)

> ⚠️ **Experimental**: surface and semantics may change.

- **Algebra** is an optional rewrite system with operator properties:
  - `operator f { assoc, comm, id(_), rest(let r) }`
    - Canonicalization: flattens `assoc`, drops `id(_)`, sorts `comm` args.
    - Rules: `rule lhs => rhs`. Normalization runs until fixpoint or `max_steps`.
    - AC matching uses a backtracking budget `ac_branch_budget`; if exhausted, the rule doesn't apply.
  - **All stored tokens are normalized**. Changing algebra re-normalizes the current marking.
- **Types** (optional) annotate stores: `any | sym | int | dec | bool | head<T...>`.
  - `any` — no constraints
  - `sym` — identifiers and string literals
  - `int` — integers
  - `dec` — fixed-precision decimals
  - `bool` — boolean values (`true`, `false`)
  - **Subtyping:** `int ⊆ dec` — an integer value is accepted wherever `dec` is expected.

### Compute directives

- **`#compute(...)`** — safe built-ins (math, comparisons, booleans, strings).  
  Ground-only; returns `int | dec | bool | string`.
  - Errors in **guards** → guard = `false`. Errors in **outputs** → grounding policy applies.
- **`#rhai("...script...", args...)`** — inline Rhai script in a sandbox.  
  Ground-only; args are available as `args` (array); returns `int | dec | bool | string`.
  - Same error semantics as above.

Full compute reference: [COMPUTE.md](./COMPUTE.md)

## Samples

### Minimal net (stores + transitions)

```relog
// Stores
store produced
store free
store buf
store done

// Transitions
transition push {
  in  produced(item(let x))
  in  free(slot)
  out buf(item(let x))
}

transition pop_pair {
  in  buf(item(let y)) * 2
  out done(pair(let y, let y))
  out free(slot) * 2
}

// Init
init {
  produced item(hello)
  produced item(world) * 2
  free slot * 2
}
```

### Guards & algebra

```relog
// Stores
store produced
store free
store buf
store done

// Algebra (optional)
algebra {
  operator set { assoc, comm, id(_), rest(let r) };

  rule member(let x, set(let x, let r)) => true;
  rule member(let _, let _) => false;
}

// Transitions
transition push {
  in  produced(item(let x))
  in  free(slot)
  guard member(let x, set(hello, world))  // must normalize to `true` (if no algebra → use literal `true`)
  out buf(item(let x))
}

transition pop_pair {
  in  buf(item(let y)) * 2
  out done(pair(let y, let y))
  out free(slot) * 2
}

// Init
init {
  produced item(hello)
  produced item(world) * 2
  free slot * 2
}
```

### Typed stores

```relog
// Built-in types: any | sym | int | dec | bool
// Parametric types: head<T1, T2, ...>

// Stores (typed)
store produced: item<sym>
store prices: dec
store free: sym
store buf: item<sym>
store done: pair<sym, sym>

// Transitions (type-checked)
transition push {
  in  produced(item(let x))
  in  free(slot)
  guard member(let x, set(hello, world))
  out buf(item(let x))
}

transition check_price {
  in  prices(let p)
  guard #compute(gt(let p, 10.0))
  out expensive(let p)
}

// Init (type-safe)
init {
  produced item(hello)
  produced item(world) * 2
  prices 15.99
  prices 5.50
  free slot * 2
}
```

### Compute / Rhai

```relog
store names
store out

transition greet_long {
  in  names(val(let n))
  guard #rhai("let s = args[0]; s.len() >= 3", let n)
  out  out(val(#compute(concat(let n, "!"))))
}

init {
  names val("ann")
  names val("bo")
}
```

### Configs

```relog
// Runtime config
runtime natural            // natural | reactive; default: natural
max_ticks 1000             // N; default: none
delay 0                    // N(ms); default: 0

algebra {
  max_steps 10000          // default: 10000
  ac_branch_budget 64      // default: 64
  // operator f { assoc, comm, id(_), rest(let r) };
  // rule lhs => rhs;
}

// Stores
store produced {
  capacity unbounded       // unbounded | N; default: unbounded
}
store free
store buf {
  capacity 8
}

// Transitions
transition push {
  grounding strict         // strict | skip | default("v"); default: strict
  priority 1               // N; default: 0

  // Arc in
  in produced(item(let x)) {
    mode consume           // consume | read | inhib; default: consume
  }

  // Arc in
  in free(slot)

  // Arc out
  out buf(item(let x)) {
    mode consume           // consume | reset; default: consume
  }
}

// Init
init {
  produced item(hello)
  produced item(world) * 2
  free slot * 2
}
```
