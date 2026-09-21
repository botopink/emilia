# emilia — user docs

emilia is the CSS-in-bp library for the botopink/jhonstart stack. You
write styles by **listing typed tokens**; emilia turns each token into a
CSS declaration, hashes the resulting body into a stable class name,
and collects every class on a per-render `<style>` registry.

## Install

`repository/emilia/botopink.json` is a **workspace** (decision 75 of
1.0.10-beta): `from "emilia"` resolves to the member `modules/emilia/` —
whose `files` (`root.bp`, `tokens.bp`, `emilia.bp`) is exactly what a
consumer sees — and never to the umbrella, which ships nothing.
`dependencies` is the object form only (decision 76); the string array is a
located error in every tool:

```jsonc
// botopink.json
{
  "dependencies": {
    "jhonstart": { "git": "https://github.com/botopink/jhonstart.git", "branch": "feat" },
    "emilia": { "git": "https://github.com/botopink/emilia.git", "branch": "feat" }
  }
}
```

A sibling member of emilia's own workspace writes `{ "emilia": { "workspace":
true } }` instead — that is what [`examples/emilia-card/`](examples/emilia-card/)
does — and a project elsewhere in the ecosystem `{ "path": "…/modules/emilia" }`.

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
[`modules/emilia/src/tokens.bp`](modules/emilia/src/tokens.bp); the highlights:

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

## The theme

A theme is a **flat list of CSS custom properties**, not nineteen record
fields. That is what CSS itself has, and it is the only shape in which
"clear one namespace" and "clear everything" are ordinary values rather than
nineteen more functions.

```bp
import { Theme, ThemeEntry, Ns, DarkMode, defaultTheme, emptyTheme,
         extendTheme, clearNamespace, namespace, themeValue, themeVar,
         nsPrefix, allNamespaces } from "emilia";
```

| Type | What it is |
| --- | --- |
| `ThemeEntry(name, value)` | one custom property — `name` carries its `--` prefix |
| `Theme(entries, keyframes, darkMode)` | the flat set, the keyframes bodies, and the dark-mode strategy |
| `Ns` | the nineteen namespaces: `Color`, `Font`, `Text`, `FontWeight`, `Tracking`, `Leading`, `Breakpoint`, `Container`, `Spacing`, `Radius`, `Shadow`, `InsetShadow`, `DropShadow`, `Blur`, `Perspective`, `Aspect`, `Ease`, `Animate`, `Keyframes` |
| `DarkMode` | `Media`, `Class(name)`, `Attribute(name, value)` |

| Function | What it answers |
| --- | --- |
| `nsPrefix(ns) -> string` | the namespace's prefix — `--color-`, `--font-`, …, `@keyframes ` |
| `allNamespaces() -> Ns[]` | the nineteen, in declaration order |
| `defaultTheme() -> Theme` | emilia's stock theme |
| `emptyTheme() -> Theme` | a theme with nothing in it — the `--*: initial` reset |
| `extendTheme(th, entries) -> Theme` | add entries; a name already present is **overridden in place** |
| `clearNamespace(th, ns) -> Theme` | drop one namespace — the `--color-*: initial` reset |
| `namespace(th, ns) -> ThemeEntry[]` | the entries of one namespace, in the theme's order |
| `themeValue(th, name) -> string` | the literal value, or `""` when the theme does not carry the name |
| `themeVar(name) -> string` | the reference form, `var(--name)` — what a utility emits |

`--spacing` is a **single variable**, so `nsPrefix(Ns.Spacing)` is `--spacing`
with no trailing dash.

```bp
fn brandTheme() -> Theme {
    val stripped = clearNamespace(defaultTheme(), Ns.Color);
    val brand: ThemeEntry[] = [
        ThemeEntry(name: "--spacing", value: "4px"),
        ThemeEntry(name: "--color-lagoon", value: "oklch(0.72 0.11 221.19)"),
    ];
    return extendTheme(stripped, brand);
}
```

### The name is `extendTheme`, not `extend`

`extend` is a **language keyword** (`Name extend Type { … }`, the type-extension
block), so `fn extend(…)` does not parse. Every front that composes a theme
writes `extendTheme`.

### `extendTheme` refuses an unknown namespace

An entry whose name starts with none of the nineteen prefixes is **refused**,
loudly, naming the offending name. There is no permissive mode and no argument
that relaxes it: an unknown prefix is a typo or a namespace this library does
not have, and both are errors. A variable that is silently absent is debugged as
a cascade bug days later, in a browser.

```bp
extendTheme(defaultTheme(), [ThemeEntry(name: "--gutter", value: "1rem")]);
// emilia theme: '--gutter' is in no known namespace — a theme entry must start
// with one of the nineteen prefixes of `Ns` (…)
```

### The colour palette is not here

`defaultTheme()` carries `--color-black` and `--color-white` and nothing else.
The full palette is front 33's data, handed over as
`paletteEntries() -> ThemeEntry[]`; a project that wants it writes
`extendTheme(defaultTheme(), paletteEntries())`.

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
