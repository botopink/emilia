# emilia — user docs

emilia is the CSS-in-bp library for the botopink/jhonstart stack. You
write styles by **listing typed tokens**; emilia turns each token into a
CSS declaration, hashes the resulting body into a stable class name,
and collects every class on a per-render `<style>` registry.

## Install

```jsonc
// botopink.json
{
  "dependencies": ["jhonstart", "emilia"]
}
```

Bare `import { emilia, flush, Token } from "emilia"` resolves at v0;
the `import emilia from "emilia"` package-handle form is a recorded
follow-up (binds the handle but doesn't currently lower the bare
`emilia(…)` call).

## Quickstart

```bp
import { emilia, flush, Token } from "emilia";
import { div, h1, p, renderToString } from "jhonstart";

val titleStyle = emilia([
    Token.TextSizeX3xl,
    Token.TextBold,
    Token.ColorRed500,
]);

val bodyStyle = emilia([
    Token.TextSizeBase,
    Token.ColorGray500,
    Token.Hover([Token.ColorRed500]),
    Token.Md([Token.TextSizeLg]),
]);

// SSR composition — markup first, then the registered stylesheet.
val markup = renderToString(div([h1([]), p([])]));
val styles = flush();
val html   = "<html><head>" + styles + "</head><body>" + markup + "</body></html>";
```

## The `Token` enum

Every utility is a typed enum variant. Unknown variants are type errors
at the call site, not runtime fall-throughs. The full v0 surface (with
the values each variant emits) lives in
[`src/tokens.bp`](src/tokens.bp); the highlights:

### Text

```bp
Token.TextBold              // font-weight:bold
Token.TextItalic            // font-style:italic
Token.TextUnderline         // text-decoration:underline
Token.TextLineThrough       // text-decoration:line-through
Token.TextLeft              // text-align:left
Token.TextCenter            // text-align:center
Token.TextRight             // text-align:right
Token.TextSizeXs            // font-size:0.75rem
Token.TextSizeSm            // font-size:0.875rem
Token.TextSizeBase          // font-size:1rem
Token.TextSizeLg            // font-size:1.125rem
Token.TextSizeXl            // font-size:1.25rem
Token.TextSizeX2xl          // font-size:1.5rem
Token.TextSizeX3xl          // font-size:1.875rem
```

### Color (text color) / Bg (background)

Fixed palette plus a `Hex(string)` escape per family:

```bp
Token.ColorRed500           // color:#ef4444
Token.ColorRed700           // color:#b91c1c
Token.ColorBlue500          // color:#3b82f6
Token.ColorBlue700          // color:#1d4ed8
Token.ColorGray500          // color:#6b7280
Token.ColorWhite            // color:#ffffff
Token.ColorBlack            // color:#000000
Token.ColorHex("#abc")      // color:#abc

Token.BgRed500              // background:#ef4444
Token.BgBlue500             // background:#3b82f6
Token.BgGray100             // background:#f3f4f6
Token.BgGray900             // background:#111827
Token.BgWhite               // background:#ffffff
Token.BgBlack               // background:#000000
Token.BgHex("#abc123")      // background:#abc123
```

### Pad

Three axes (`X`, `Y`, `All`); the numeric scale is `1 = 0.25rem`,
`2 = 0.5rem`, `4 = 1rem`, `8 = 2rem`, `16 = 4rem`.

```bp
Token.PadX4                 // padding-left:1rem;padding-right:1rem
Token.PadY2                 // padding-top:0.5rem;padding-bottom:0.5rem
Token.PadAll4               // padding:1rem
```

### Modifiers — state + breakpoint wrappers

Each modifier carries a `Token[]` payload that emilia recursively
resolves and wraps in the matching CSS prefix:

```bp
Token.Hover([Token.BgRed700])           // :hover{background:#ef4444}        (BgRed700 maps if added)
Token.Focus([Token.ColorBlue500])       // :focus{color:#3b82f6}
Token.Active([Token.TextUnderline])     // :active{text-decoration:underline}
Token.Md([Token.TextSizeLg])            // @media(min-width:768px){font-size:1.125rem}
Token.Lg([Token.PadX8])                 // @media(min-width:1024px){padding-left:2rem;padding-right:2rem}
Token.Xl([Token.PadX16])                // @media(min-width:1280px){padding-left:4rem;padding-right:4rem}
```

Modifiers nest:

```bp
Token.Md([Token.Hover([Token.BgBlack])])
// @media(min-width:768px){:hover{background:#000000}}
```

## The runtime — `emilia(tokens)` and `flush()`

- `emilia(tokens: Token[]) -> string`
  - Maps each token through `tokenToCss`, filters empty declarations,
    joins on `;`, content-hashes the body, registers
    `(e_<hash>, body)` on the per-render `Stylesheet`, returns the
    class name.
- `flush() -> string`
  - Serialises the `Stylesheet` into `<style>.e_<hash>{...} ...</style>`
    AND clears the cell. Per-render contract: two consecutive
    `flush()` calls emit two independent blocks; the second is
    `<style></style>` if no `register` happened between them.

### Stable hashes — sites collapse

Identical token lists on different elements share one class:

```bp
val a = emilia([Token.BgWhite]);
val b = emilia([Token.BgWhite]);
assert a == b;                          // same content-derived hash
// The <style> block contains exactly ONE `.<a>{background:#ffffff}` rule.
```

## What's coming (v0.beta.21+)

The spec authors a richer surface that v0 does not yet ship:

- `#[emilia([.Pad.All.4, .Bg.White])] div([…])` — annotation on
  builder, generic jhonstart hook.
- `<div [emilia]={[.Pad.All.4, .Bg.White]}>` — `[name]={expr}`
  attribute inside the `html """…"""` DSL, generic jhonstart hook.
- Nested section paths (`.Color.Red.500`, `.Pad.X.4`) — gated on the
  `enum-sections` language extension. v0's flat variant names
  (`ColorRed500`, `PadX4`) re-shape into the section paths when
  `enum-sections` lands.
- erlang/beam port for the `Stylesheet` host cell — same surface, swap
  the `Map` for `persistent_term` or an Agent.

The intent + the comptime expansion model live in the spec at
[`tasks/v0.beta.20/specs/ecosystem.md`](../../tasks/v0.beta.20/specs/ecosystem.md);
the AGENTS.md tracks the deferred items and the maintainer rules.
