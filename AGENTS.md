# emilia — agent notes

> CSS-in-bp library for botopink/jhonstart. Pure `.bp`, **zero compiler surface**
> (the core is unaware of emilia — memory: feedback_no_lib_specific_in_core,
> feedback_compiler_unaware_of_jhonstart). Sibling of `jhonstart/html` and
> `erika/erika`: the same `template fn (comptime t: @Expr<string>) -> @ExprCustom<T>`
> mechanism, applied to a CSS sub-language.

## Surface

Three entry points, all bound bare from `from "emilia"`:

1. **`styled.<tag>`** — styled-components form. `styled.div """…"""` returns a
   builder `fn(children: Children) -> Element` that renders the tagged element
   with a stable, content-derived `className` and emits the rule into the
   collected stylesheet.
2. **`css """…"""`** — anonymous, reusable rule fragment. Returns an
   `EmiliaClass` value that an `attr(...)` decoration drops onto any Element.
3. **`tw "…"`** — tailwind-style utility shorthand. Each whitespace-separated
   token (`p-4`, `bg-white`, `text-center`, …) maps to a fixed CSS declaration
   via a comptime token table. Composes into the same `EmiliaClass` value as
   `css """…"""`.

All three carry an `@ExprCustom` overlay so the LSP underlines an unknown
utility token (`tw "p-4 unkwn-token"`) at the offending span, exactly the way
`erika` underlines an unknown collection name.

## How it expands

- **Lexer / parser run at comptime** over the template's `parts()` (same path
  as `html`/`erika`). For `css` / `styled.<tag>`: a CSS declaration scanner
  (`prop : value ;`); for `tw`: a whitespace token split mapped against a
  fixed `tokenTable` (`p-4 → padding: 16px;`, etc.).
- **`${expr}` holes** inside `css""" … """` / `styled.div""" … """` splice
  the caller's typed expression into the right-hand side of a declaration,
  the same way `erika`'s `${threshold}` reaches the SQL operand.
- **Lowering ③ → @Expr<EmiliaClass>:** builds a `record { className, rules }`
  value (`className` derived from a hash of the source; `rules` the parsed
  declarations).
- **Lowering ④ → CustomNode:** a flat overlay with `label: "property"` on
  property names, `label: "string"` / `"number"` on values, `label: "keyword"`
  on `tw` utility heads — feeding the LSP overlay built in
  `sublanguage-lsp`.

## How it integrates with jhonstart

emilia depends only on jhonstart's `Element` shape (`{ tag, value, children }`).
A styled builder is an ordinary `fn(Children) -> Element` that:

1. Wraps the children under its tag.
2. Appends `class="<className>"` to the element's rendered attributes through
   an `attr(EmiliaClass)` child convention (children carry attribute markers
   the renderer pulls out before walking the rest — see the spec for the
   minimal jhonstart change).
3. Registers the rule fragment in a process-wide `Stylesheet` registry (an
   `#[@external]` host cell — fine for SSR; the seam is hidden behind
   `emilia.collect()` / `emilia.flush()`).

The SSR call site composes:

```bp
val markup = renderToString(page);          // jhonstart
val styles = emilia.flush();                // <style>…</style>
val html   = "<html><head>" + styles + "</head><body>" + markup + "</body></html>";
```

`emilia.flush()` returns the collected stylesheet as a `<style>` tag and clears
the registry (per-request semantics; the test harness resets in `beforeEach`).

## What emilia is NOT

- **Not a compiler change.** No new `#[@…]` annotation, no AST node, no codegen
  hook. The entire DSL is comptime template-fn evaluation (the same evaluator
  `erika` / `html` already use).
- **Not a runtime CSS engine.** No selector parsing, no nesting beyond a single
  `:hover` / `&:focus` form (recorded follow-up), no preprocessor pipeline. The
  scope is a flat declaration list per class.
- **Not coupled to a backend beyond commonJS at v1.** The `Stylesheet` host
  cell is a JS `Map`; an erlang/beam port is a clean follow-up (no API change —
  the `#[@external]` body swaps, the surface stays the same).

## Files

- `src/root.bp` — `pub default mod emilia;` (the package handle binds the
  `tw "…"` template; `styled.*` and `css """…"""` are public surface).
- `src/emilia.bp` — the whole lib: the `EmiliaClass` record, the `styled`
  namespace, the `css` + `tw` template fns, the `Stylesheet` host cell, and
  the in-file `test {}` block(s).
- `botopink.json` — `files: ["root.bp", "emilia.bp"]` (`.d.bp` are NOT in the
  module tree — memory: project_libs_module_migration_done).

## Maintainer rules

- **camelCase** all method/fn names (`flushSheet`, not `flush_sheet` — memory:
  feedback_camelcase_naming).
- **No comment inside a closure body** in the comptime-evaluated template body
  (the emitter flattens each block to one line — memory:
  reference_bp_parser_comptime_gotchas).
- **One single `tokens.append` site** in the lexer (the comptime type-checker
  mis-unifies the token array across 3+ branchy append sites — same gotcha
  erika hit; lock the kind in a `val` before the single append).
- **`from "emilia"` only**, never a relative module path: emilia is a
  workspace-external lib and the consumer is jhonstart-based code.
