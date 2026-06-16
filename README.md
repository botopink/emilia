# emilia

> Type-safe CSS-in-bp library for botopink/jhonstart. A `Token` enum
> surface walked at comptime, resolved to flat CSS declarations,
> registered on a per-render `<style>` host cell, and flushed alongside
> `renderToString`.

The CSS sibling of `jhonstart`'s `html` and `erika`'s `erika "…"` — same
mechanism (a typed sub-language consumed at the call site), different
sub-language. Where `html` parses *string markup* and `erika` parses
*string SQL*, emilia consumes an **array literal of typed enum
variants**. Unknown utilities are type errors at the call site, not
runtime fall-throughs.

## Today's surface (v0)

```bp
import { emilia, flush, Token } from "emilia";
import { div, h1, p, renderToString } from "jhonstart";

val cardBg = emilia([
    Token.PadAll4,
    Token.BgWhite,
    Token.ColorBlack,
]);

val titleStyle = emilia([
    Token.TextSizeX3xl,
    Token.TextBold,
    Token.ColorRed500,
]);

val bodyStyle = emilia([
    Token.TextSizeBase,
    Token.ColorGray500,
    Token.Hover([Token.ColorRed500]),         // :hover{color:#ef4444}
    Token.Md([Token.TextSizeLg]),             // @media(min-width:768px){...}
]);

// Render then flush — SSR composition pattern from the spec.
val markup = renderToString(div([h1([]), p([])]));
val styles = flush();
val html   = "<html><head>" + styles + "</head><body>" + markup + "</body></html>";
```

`emilia(tokens)` returns a content-derived class name (`e_<hash>`) and
registers the rule body on the per-render `Stylesheet`. Identical token
lists collapse to one class across the document. `flush()` returns the
`<style>...</style>` block and clears the cell — two consecutive
`flush()` calls emit two independent blocks.

## Token surface (v0)

| Section | Variants |
| --- | --- |
| Text   | `TextBold`, `TextItalic`, `TextUnderline`, `TextLineThrough`, `TextLeft`, `TextCenter`, `TextRight`, `TextSize{Xs, Sm, Base, Lg, Xl, X2xl, X3xl}` |
| Color  | `ColorRed{500, 700}`, `ColorBlue{500, 700}`, `ColorGray500`, `ColorWhite`, `ColorBlack`, `ColorHex(string)` |
| Bg     | `BgRed500`, `BgBlue500`, `BgGray{100, 900}`, `BgWhite`, `BgBlack`, `BgHex(string)` |
| Pad    | `PadX{1, 2, 4, 8, 16}`, `PadY{1, 2, 4}`, `PadAll{1, 2, 4, 8, 16}` (`4` = `1rem`) |
| Modifier | `Hover([Token])`, `Focus([Token])`, `Active([Token])`, `Md([Token])`, `Lg([Token])`, `Xl([Token])` |

The numeric leaves (`500`, `4`, …) live on the variant *name* at v0 —
the nested-section form the spec authors (`.Color.Red.500`, `.Pad.X.4`)
requires the [`enum-sections`](../../tasks/v0.beta.20/specs/frente-a.md)
language extension. v1 (`F1` in the spec phases) re-shapes the enum into
sections; the `tokenToCss` dispatch + the registry/flush mechanism stay
identical.

## What landed at v0.beta.20

- **F0** lib stand-up — `Token` enum (3 sections + 6 modifier variants),
  `emilia(tokens) -> string`, the `Stylesheet` host cell folded into
  `emilia.bp`, two-module package (`tokens` + `emilia`), 9 in-file tests
  green.
- **F2** `tokenToCss` exhaustive `case` over every variant.
- **F3** modifier composition — `Hover`/`Focus`/`Active` produce
  `:pseudo{…}` blocks; `Md`/`Lg`/`Xl` produce `@media(min-width:Xpx){…}`
  blocks; modifiers nest (`Md(Hover(...))` composes both wrappers).
- **F4** `flush()` per-render semantics — registered classes serialise
  into a `<style>` block, the cell clears, the next `flush()` is fresh.
- **F5** runnable [`examples/emilia-card/`](examples/emilia-card/) — a
  small jhonstart page composed with three emilia class names, plus a
  modifier in the body style.

## Deferred (v0.beta.21+)

- **`enum-sections`** — moves the enum to the spec's nested-section
  form so the consumer writes `.Color.Red.500`, `.Pad.X.4`,
  `.Text.Size.X3xl` instead of the v0 flat prefix.
- **F0 second half — jhonstart hooks (generic, no emilia mention):**
  - **Annotation-on-builder** (`#[emilia([…])] div([…])`) — call-site
    decorator that injects the class into the produced `Element`.
  - **`[name]={expr}` html attribute** — html DSL scanner recognises
    `<tag [emilia]={[...]}>` as an annotation attribute and routes to
    the registered handler.
  - Both require an attribute slot on `Element` first (jhonstart spec
    follow-up) and the generic decorator-on-call-site machinery in the
    compiler.
- **`stylesheet.bp` split** — the v0 build folds the `#[@external]`
  surface into `emilia.bp`. The 3-module split (`tokens` /
  `stylesheet` / `emilia`) type-checks but `import { register }` from a
  sibling module resolves at type level only — the runtime symbol is
  undefined. Restoring the split is gated on cross-module-external
  binding parity with the `import { env } from "std"` path.
- **erlang/beam Stylesheet port** — same `register`/`flush` contract,
  swap the `Map` for `persistent_term` or an Agent. Pairs with the
  rakun erlang server port in v0.beta.21.

The full intent, the comptime expansion model, and the jhonstart
integration contract live in
[`tasks/v0.beta.20/specs/ecosystem.md`](../../tasks/v0.beta.20/specs/ecosystem.md).

## License

Same as the parent botopink workspace.
