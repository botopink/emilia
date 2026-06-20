# emilia — agent notes

> Type-safe CSS-in-bp library for botopink/jhonstart. **Zero compiler
> surface** at v0 — the core is unaware of emilia (memory:
> `feedback_no_lib_specific_in_core`,
> `feedback_compiler_unaware_of_jhonstart`). Sibling of `jhonstart/html`
> and `erika/erika`: a typed sub-language consumed at the call site, but
> the sub-language is an **array literal of typed enum variants**, not a
> string.

## Surface (v0)

Three named imports from `from "emilia"`:

1. **`emilia(tokens: Token[]) -> string`** — the entry point. Walks
   the token list at runtime (v0 — no comptime expansion yet), maps
   each variant to its CSS declaration, joins on `;`, hashes the body
   into a stable class name, registers the `(name, body)` pair on the
   per-render `Stylesheet`, returns the class name.
2. **`flush() -> string`** — serialises the `Stylesheet` into a
   `<style>...</style>` block AND clears the cell. Per-render
   contract — two consecutive calls emit two independent blocks; the
   second is `<style></style>` if no `register` happened in between.
3. **`Token` enum** — the typed authored surface (see `tokens.bp`).
   Sections (v0 flat): Text, Color, Bg, Pad + modifier variants
   (`Hover`/`Focus`/`Active`/`Md`/`Lg`/`Xl`) carrying a nested
   `Token[]`.

The spec authors a richer surface (a `#[emilia(...)]` decorator on a
builder call + a `[emilia]={...}` attribute inside the `html """…"""`
DSL); both forms need the two generic jhonstart hooks (`F0` second
half — deferred). v0 ships the underlying mechanism without the two
attachment syntaxes.

## How it expands

- **Walk at runtime, not comptime.** v0's `emilia(tokens)` is an
  ordinary `pub fn`: the token array reaches the body as a runtime
  value (not an `@Expr<Token[]>`); the `case t { … }` dispatcher fires
  per element. Promoting to comptime — so the consumer writes
  `#[emilia([.Bg.White])] div([])` and the class name + registration
  side-effect lower at compile time — needs the jhonstart
  annotation-on-builder hook (a generic mechanism) and the decorator
  body must be one of the new "call-site decorator" shapes that pass
  the call's argument list as `@Expr<Token[]>`.
- **`Token[]` for the modifier payload.** The modifier variants
  (`Hover([Token]), Md([Token])`, …) recurse through `tokensToCss` to
  produce a single wrapped string (`":hover{...}"`,
  `"@media(min-width:768px){...}"`), which is concatenated into the
  outer class body just like any flat declaration.
- **Lowering to JS.** v0 lowers to plain `function emilia(tokens) {
  ... }` + a host expression per `#[@external]` declaration. There is
  **no compiler change** beyond what was already in `feat` (`@external`
  templates, `case`/exhaustiveness, enum payload destructuring with
  named fields).

## How it integrates with jhonstart (today and tomorrow)

- **Today.** A consumer does
  `import { emilia, flush, Token } from "emilia"`, calls `emilia(...)`
  to GET a class name, then composes the SSR output as
  `renderToString(page) + flush()`. v0's `Element` has **no attribute
  slot**, so the class name lives in the `<style>` block but is not
  yet inlined onto the rendered tag. The recorded follow-up there is
  the `Element` attribute carrier (jhonstart spec — there is already
  a tracked task), independent of emilia.
- **Tomorrow (deferred — `F0` second half + jhonstart spec):**
  - **Annotation-on-builder.** A generic jhonstart hook that recognises
    `#[<name>([…])] tag(children)` on a builder call and transforms it
    to `tag(children, attrs: [class: <name>(tokens)])`. emilia is one
    consumer (`<name>` = `emilia`); a future ecosystem lib (e.g.,
    data-attribute helpers) plugs into the same hook with no
    additional compiler change.
  - **`[name]={expr}` html attribute.** The html scanner sees
    `<tag [name]={expr}>`, looks up `name` in the registered
    annotation handlers (`emilia` here), passes `<expr>` to it, and
    replaces the attribute with `class={name(expr)}`. Generic — the
    html DSL knows nothing about emilia.

Both hooks are **emilia-agnostic** — jhonstart owns the mechanism.

## What emilia is NOT

- **Not a compiler change.** No new `#[@…]` annotation in the core, no
  AST node, no codegen hook. The entire DSL is `pub fn` + `case` +
  `#[@external]` — already shipped primitives.
- **Not a runtime CSS engine.** No selector parsing, no nested
  selectors beyond the modifier wrappers (`Hover`, `Focus`, `Active`,
  `Md`, `Lg`, `Xl`), no preprocessor pipeline. The scope is a flat
  declaration list per class with the modifier wrappers nested ONE
  level deep (and themselves nestable, e.g. `Md(Hover(...))`).
- **Not coupled to a backend beyond commonJS at v1.** The `Stylesheet`
  host cell is a JS `Map`; an erlang/beam port is a clean follow-up
  (same `register`/`flush` contract, swap the cell type).

## Files

- `src/root.bp` — `pub mod tokens; pub default mod emilia;` (the
  v0 build folded the `stylesheet` module into `emilia.bp` — see the
  "Deferred" section of the README for the restoration follow-up).
- `src/tokens.bp` — the `Token` enum: all v0 variants + the modifier
  variants. Section headers live in the docblock; v0 NEVER puts a
  line comment inside the enum body (parser gotcha).
- `src/emilia.bp` — the public `emilia(tokens) -> string` +
  `flush() -> string` + the `tokenToCss`/`tokensToCss` dispatchers +
  the `#[@external(node, …)]` host-cell expressions (`register`,
  `flushSheet`).
- `botopink.json` — `files: ["root.bp", "tokens.bp", "emilia.bp"]`
  (`.d.bp` are NOT in the module tree — memory:
  `project_libs_module_migration_done`).
- `examples/emilia-card/` — runnable smoke (4 in-file `test {}`)
  composing three emilia class names + a Hover/Md modifier on a small
  jhonstart page; depends on `jhonstart` + `emilia`.

## Maintainer rules

- **camelCase** all method/fn names (`tokenToCss`, `flushSheet`,
  `hashHex` — never `token_to_css` — memory:
  `feedback_camelcase_naming`).
- **Enum payload destructuring uses the NAMED FIELD** —
  `ColorHex(value: string)` is matched as `ColorHex(value) -> …`, not
  `ColorHex(v) -> …`. The positional binding parses but lowers to
  `undefined` (codegen relies on the field name to project the payload
  from the runtime variant record).
- **No line comment inside an enum body.** Section headers go in the
  module docblock; the parser trips on a `//` between variant arms.
- **Array type spelling is postfix** — `Token[]`, NOT `[Token]`. The
  prefix bracket parses only as an array literal (`[Token.PadX4]`).
- **`from "emilia"` only**, never a relative module path. emilia is a
  workspace-external lib and the consumer is jhonstart-based code.
- **Cross-module `#[@external]` symbol imports don't lower at v0** —
  `import { register };` from a sibling module resolves the type but
  the runtime symbol is `undefined`. Until the codegen path closes
  that gap, keep all `#[@external]` host-cell expressions in the
  module that USES them (currently `emilia.bp`).

## Test surface

- `botopink test` inside `repository/emilia/` runs `src/emilia.bp`'s
  11 in-file `test {}` blocks (Text.Bold / Text.Size.Lg / Color.Black
  / Bg.White / Layout.Flex / Border.Rounded.Full / Effect.Shadow.Md +
  Hover/Md modifier composition + multi-token `tokensToCss` + nested
  modifier chain). The full `emilia(tokens)/flush()` public surface +
  the `examples/emilia-card/` runtime smoke remain deferred — they
  belong to the V1 re-author follow-up (the `register`/`flushSheet`/
  `hashHex` declares + the V1 example migration `Token.X` →
  `.X.Y.Z`).
- `botopink test` inside `examples/emilia-card/` reds at v0.beta.22
  because the example still uses V0 token names (`Token.PadAll4`,
  `Token.BgWhite`, …) — migration to V1 paths is part of the same
  follow-up.

## Spec / phase status

| Phase | Status |
| --- | --- |
| F0 — lib stand-up | DONE-then-undone (V0 surface dropped during V1 WIP) |
| F1 — fill out `Token` | DONE (V1 nested-section landed `3f77623`) |
| F2 — `tokenToCss` exhaustive | DONE for V1 (re-pinned under v0.beta.22 ecosystem-and-snap-tail F2) |
| F3 — modifier composition | DONE for V1 (re-pinned alongside F2) |
| F4 — `flush()` per-render | DEFERRED (re-author after V1 example migration) |
| F5 — example + docs sweep | DEFERRED (V1 example migration first) |

Spec lives in
[`tasks/v0.beta.20/specs/ecosystem.md`](../../tasks/v0.beta.20/specs/ecosystem.md);
the V1 re-author follow-up rides on
[`tasks/v0.beta.22/specs/05-ecosystem-and-snap-tail.md`](../../tasks/v0.beta.22/specs/05-ecosystem-and-snap-tail.md)
F2's deferred half.
