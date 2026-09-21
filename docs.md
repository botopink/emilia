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

### Color — the palette

Twenty-six families of eleven shades, and **the token emits a reference, not a
value**:

```bp
.Color.Red.500              // color:var(--color-red-500)
.Color.Sky.100              // color:var(--color-sky-100)
.Color.Taupe.950            // color:var(--color-taupe-950)
.Color.White                // color:var(--color-white)
.Color.Black                // color:var(--color-black)
.Color.Transparent          // color:transparent
.Color.Current              // color:currentColor
.Color.Inherit              // color:inherit
```

The families are the seventeen chromatic (`Red Orange Amber Yellow Lime Green
Emerald Teal Cyan Sky Blue Indigo Violet Purple Fuchsia Pink Rose`) and the
nine neutral (`Slate Gray Zinc Neutral Stone Mauve Olive Mist Taupe`); every
one of them answers all eleven shades `50 100 200 300 400 500 600 700 800 900
950`.

`var(--color-red-500)` is byte-equal with what upstream's own `.text-red-500`
rule emits, and it means a project that overrides `--color-red-500` moves every
rule that names it. The numbers live in the theme, not in the rule — see
[The colour palette](#the-colour-palette).

**Six cells are declared and unreachable.** `.Color.Red.100`, `.Color.Red.500`,
`.Color.Red.700`, `.Color.Gray.100`, `.Color.Gray.500` and `.Color.Gray.700`
do not compile in any spelling: the compiler resolves a leading-dot section
path by scanning every registered enum — the synthesised section enums
included — and returning the first whose tree carries the path, without ever
consulting the expected type. `Token` carries `Color.Red.500` and so does
`Token.Border.Color`, whose Red and Gray also run 100/500/700, so the winner is
decided by hash order. It reds at the call site (`type mismatch: expected
Token, got __Token__Border`) rather than emitting the wrong CSS. Until the
resolver is fixed, reach those six shades through `.Bg.Color.<Family>.<shade>`
(whose head segment `Bg` is unique) or pick a neighbouring shade.

### Bg.Color — the same palette on `background-color`

```bp
.Bg.Color.Red.500           // background-color:var(--color-red-500)
.Bg.Color.Sky.100           // background-color:var(--color-sky-100)
.Bg.Color.Slate.900         // background-color:var(--color-slate-900)
.Bg.Color.Taupe.950         // background-color:var(--color-taupe-950)
.Bg.Color.White             // background-color:var(--color-white)
.Bg.Color.Black             // background-color:var(--color-black)
.Bg.Color.Transparent       // background-color:transparent
.Bg.Color.Current           // background-color:currentColor
.Bg.Color.Inherit           // background-color:inherit
```

Same 26 families, same eleven shades, one property along — and **all 286 cells
resolve**, because `Bg` is a head segment no other enum carries. It is the way
to reach the six shades `.Color.Red` / `.Color.Gray` cannot.

The property is `background-color` and not `background`. Upstream's `bg-*`
colour utilities set the longhand; the shorthand the v0 stub emitted resets
every other background property of the element as a side effect.

The four **legacy** `Bg` leaves keep what they emitted before the palette
landed, so nothing that compiled changed meaning; front 39 folds them in when
it lands:

```bp
.Bg.White                   // background:#ffffff
.Bg.Black                   // background:#000000
.Bg.Red.500                 // background:red
.Bg.Gray.500                // background:gray
```

### `Color.Hex("#abc")` is declared and unconstructible

The `Hex(value: string)` leaf is kept — it is the proof that a string payload
splices into the emitted CSS, and `colorTokenToCss` still matches it — but
**no caller can build one**. Measured against the compiler, on this enum:

```text
Token.Color.Hex("#abc")
  error: 'Hex' is not declared in any behavior implemented for 'Token'
val h: Token = .Color.Hex("#abc");
  error: unbound variable 'Color'
```

A payload leaf nested inside a section has no constructible spelling; the
variant type-checks in a `case` pattern, so the surface looks complete and is
not. The shape that builds is a **top-level** variant with builtin-typed
fields — which is what `Alpha(percent: i32, inner: Token[])` is, and what an
arbitrary-value escape hatch will have to be.

Nothing in the palette depends on the enum, which is why that gap costs this
front nothing: `paletteVar(family, shade)` and `alphaWrap(percent, css)` take
plain strings and are `pub`, and the property name (`color:`,
`background-color:`) lives in the dispatcher rather than in the value. A
colour emilia's enum does not carry formats through the same functions.

### Alpha — opacity, upstream's `/N` suffix

```bp
val red: Token[] = [.Bg.Color.Red.500];
Token.Alpha(percent: 50, inner: red)
// background-color:color-mix(in oklab, var(--color-red-500) 50%, transparent)

val blue: Token[] = [.Color.Blue.600];
Token.Alpha(percent: 80, inner: blue)
// color:color-mix(in oklab, var(--color-blue-600) 80%, transparent)
```

`bg-red-500/50` is a **wrapper**, not a leaf. Opacity cannot hang under the
family — the shade is already the leaf, and a second numeric level would
multiply the grid to 26 × 11 × 21 — and a payload leaf nested inside a section
has no constructible spelling, so `Alpha` is a top-level variant with a
builtin-typed field, the shape `Hover(inner: Token[])` already has.

It rewrites the VALUE of every declaration its inner tokens produce, rule by
rule, so the selector and at-rule of a modifier inside it survive and it
composes in either order:

```bp
Token.Hover([Token.Alpha(percent: 50, inner: red)])   // same rule
Token.Alpha(percent: 50, inner: [Token.Hover(red)])   // as this one
```

A **non-colour** token is rewritten too, and the result is meaningless CSS
(`font-weight:color-mix(in oklab, bold 50%, transparent)`). That is
deliberate: dropping the declaration silently would hide the mistake, and one
a browser discards is visible in devtools the moment it is looked for.

### Pad and Margin

**Nine directions each** — `All`, `X`, `Y`, `T`, `R`, `B`, `L`, and the logical
pair `S`/`E` that follows the writing direction where left and right do not.

```bp
.Pad.All.4     // padding:calc(var(--spacing) * 4)
.Pad.X.4       // padding-left:calc(var(--spacing) * 4);padding-right:calc(var(--spacing) * 4)
.Pad.T.4       // padding-top:calc(var(--spacing) * 4)
.Pad.S.4       // padding-inline-start:calc(var(--spacing) * 4)
.Pad.E.4       // padding-inline-end:calc(var(--spacing) * 4)
```

CSS has **no `padding-x` property**, so an axis token is two declarations, not
one. Until this front emilia emitted `padding-x:`, `padding-y:` and `margin-y:`
— property names no browser knows, silently discarded — and the `Margin.X`
ladder emitted `m-0.25`, `m-1` and `margin-auto`, which are Tailwind class
fragments rather than declarations. Those eight spellings are gone, and a test
asserts each of them is.

#### The scale

Every direction answers the same 35 leaves: the thirty multipliers
`0 1 2 3 4 5 6 7 8 9 10 11 12 14 16 20 24 28 32 36 40 44 48 52 56 60 64 72 80 96`,
the pixel step `Px`, and `Half { 0, 1, 2, 3 }`.

```bp
.Pad.All.0        // padding:0            — `p-0` is a bare 0, not a calc of zero
.Pad.All.1        // padding:calc(var(--spacing) * 1)
.Pad.All.96       // padding:calc(var(--spacing) * 96)
.Pad.All.Half.1   // padding:calc(var(--spacing) * 1.5)   — upstream's `p-1.5`
.Pad.All.Px       // padding:1px
```

`0.5` cannot be an enum leaf — a numeric leaf is a run of digits — so `Half.1`
reads "one and a half" and `Half { 0, 1, 2, 3 }` covers `0.5`, `1.5`, `2.5` and
`3.5`. **A spacing value is never resolved here**: every leaf answers
`spacing(n)` or `spacingHalf(n)`, so `--spacing` stays the one place the length
is decided (§ Spacing — `spacing(n)`).

#### `Auto` and `Neg`, on margin

`Auto` is a value on the ladder, so it is on **every** margin direction and gets
the same expansion every multiplier gets. `Neg` is upstream's `-mt-4`, the one
family of utilities with no alternative spelling:

```bp
.Margin.All.Auto    // margin:auto
.Margin.X.Auto      // margin-left:auto;margin-right:auto
.Margin.T.Neg.4     // margin-top:calc(var(--spacing) * -4)
.Margin.X.Neg.2     // margin-left:calc(var(--spacing) * -2);margin-right:calc(var(--spacing) * -2)
.Margin.T.Neg.Px    // margin-top:-1px
```

`Neg` carries no `0` and no `Auto` — a negative zero and a negative auto are not
utilities — and its `Half` is `{ 1, 2, 3 }`, not `{ 0, 1, 2, 3 }`. The missing
rung is `-0.5`: `spacingHalf` takes an `i32`, an `i32` has no negative zero, and
emilia will not write a second `calc(var(--spacing) * …)` of its own to reach
one value. Closing it is front 54's (a signed half step); until then `-mt-0.5`
has no token, and no wrong token either.

### Size

Thirteen sub-sections: `W`, `H`, `Both`, `MinW`, `MaxW`, `MinH`, `MaxH`, and the
six logical forms `Inline`, `Block`, `MinInline`, `MaxInline`, `MinBlock`,
`MaxBlock`. Four kinds of leaf.

```bp
.Size.W.64            // width:calc(var(--spacing) * 64)   — the spacing ladder again
.Size.W.Px            // width:1px
.Size.W.Frac.Half     // width:50%
.Size.W.Frac.Third    // width:33.333333%
.Size.W.Full          // width:100%
.Size.W.Min           // width:min-content
.Size.W.Auto          // width:auto
```

`1/2` is neither an identifier nor a run of digits, so a fraction cannot be a
leaf; `Frac` carries the eleven upstream fractions by name — `Half`, `Third`,
`TwoThirds`, `Quarter`, `ThreeQuarters`, `Fifth`, `TwoFifths`, `ThreeFifths`,
`FourFifths`, `Sixth`, `FiveSixths`.

**The viewport unit differs by axis** and the tests say so:

```bp
.Size.W.Screen        // width:100vw
.Size.H.Screen        // height:100vh
.Size.H.Dvh           // height:100dvh
.Size.MinH.Screen     // min-height:100vh
```

`Both` is upstream's `size-*` — two declarations from one leaf:

```bp
.Size.Both.12         // width:calc(var(--spacing) * 12);height:calc(var(--spacing) * 12)
.Size.Both.Full       // width:100%;height:100%
```

#### The named widths are the theme's, not emilia's

`max-w-md` is `max-width:var(--container-md)` upstream and here. The `rem`
behind each name is written once, in the theme, so a project that redefines
`--container-md` moves every `max-w-md` in the build — which a literal ladder in
the dispatcher would have made impossible.

```bp
.Size.MaxW.Md            // max-width:var(--container-md)        (--container-md is 28rem)
.Size.MaxW.X3xl          // max-width:var(--container-3xl)        (48rem)
.Size.MaxW.Screen.X2xl   // max-width:var(--breakpoint-2xl)       (96rem)
.Size.MaxInline.Md       // max-inline-size:var(--container-md)
```

`MaxW` carries all thirteen container names, `X3xs` and `X2xs` through `X7xl`,
and `MaxW.Screen` the five breakpoints. No `Size` leaf emits a `rem` of its own;
a test walks all 566 of them and asserts it.

### Modifiers — state + breakpoint variants

Each modifier carries a `Token[]` payload. A modifier is **not** a block
nested inside the class body — it is a `Variant`, and the tokens it carries
become **sibling rules** with their own selector and their own at-rule, hoisted
out of the class. The table below is the pre-front-56 shape and is kept only
because the token spellings are still current; the emitted CSS is in
§ The cascade and the output.

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
  - Maps each token through `tokenToSheet`, merges the sheets in list order,
    encodes the result, content-hashes the encoding, registers
    `(e_<hash>, payload)` on the per-render `Stylesheet`, returns the class
    name.
- `emiliaWith(tokens: Token[], th: Theme) -> string`
  - The same, under an explicit theme. `emilia(tokens)` is
    `emiliaWith(tokens, defaultTheme())`. The theme is an input: a `Md`
    modifier carries the theme's `--breakpoint-md`, so two themes give two
    classes for the same token list.
- `flush() -> string`
  - Drains the `Stylesheet` and renders the **document** — the `@layer`
    statement, the theme layer, the base layer, the components layer, the
    utilities layer and the `@keyframes` blocks — AND clears the cell.
    Per-render contract: two consecutive `flush()` calls emit two
    independent documents; the second has no `@layer utilities` body if no
    `register` happened between them.
- `flushWith(o: Options) -> string`
  - The same, under explicit options. `flush()` is
    `flushWith(defaultOptions())`. See § The cascade and the output.

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

### Spacing — `spacing(n)`

| Function | What it answers |
| --- | --- |
| `spacing(n) -> string` | `calc(var(--spacing) * n)`; `spacing(0)` is `0` |
| `spacingHalf(n) -> string` | `calc(var(--spacing) * n.5)` |

emilia **never resolves a spacing value**. Every spacing utility is a multiplier
of `var(--spacing)`, resolved by the browser, so the same token list renders
differently under a theme whose `--spacing` is `4px` and one whose `--spacing`
is `0.25rem`. Emitting `1rem` instead would make that override silently
ineffective.

```bp
assert spacing(0) == "0";
assert spacing(4) == "calc(var(--spacing) * 4)";
assert spacing(-4) == "calc(var(--spacing) * -4)";
assert spacingHalf(1) == "calc(var(--spacing) * 1.5)";
```

Negative steps need no second function. `spacing` takes an `i32`, not an `f64`,
so the emitted string never depends on a backend's float formatting; the
fractional steps are a closed set that `spacingHalf` covers exactly.

### Rendering a theme

| Function | What it answers |
| --- | --- |
| `themeCss(th) -> string` | the `:root` body — `name:value` joined with `;` |
| `keyframeCss(th) -> ThemeEntry[]` | the `@keyframes` bodies, which cannot sit inside `:root` |
| `darkAtRule(th) -> string` | the at-rule a dark rule nests inside, or `""` |
| `darkSelector(th) -> string` | the selector a dark rule is written against |
| `withDarkMode(th, mode) -> Theme` | swap the strategy, change nothing else |

The block is **always the whole theme**. Upstream emits only the variables a
build used, and `@theme static` emits all of them; emilia's behaviour is the
second, because tree-shaking would need a whole-program pass over every
`emilia()` call site and emilia hashes per call site.

Entry order is the theme's own, and it matters: emilia's class names are content
hashes, so a reordered theme is a different document for the same input.

A keyframes entry carries the bare animation name and a brace-balanced body, so
one block is written `nsPrefix(Ns.Keyframes) + name + value` —
`@keyframes spin{to{transform:rotate(360deg)}}` — and nobody spells the at-rule
by hand.

| Strategy | `darkAtRule` | `darkSelector` |
| --- | --- | --- |
| `DarkMode.Media` | `@media (prefers-color-scheme: dark)` | `&` |
| `DarkMode.Class(name: "dark")` | `""` | `&:where(.dark, .dark *)` |
| `DarkMode.Attribute(name: "data-theme", value: "dark")` | `""` | `&:where([data-theme=dark], [data-theme=dark] *)` |

`Class` and `Attribute` carry the whole strategy in the selector, which is why
their at-rule is empty. The `Dark` token that consumes these is front 34's.

### The colour palette

`defaultTheme()` carries `--color-black` and `--color-white` and nothing else.
The 286 numeric values are front 33's data, handed over as
`paletteEntries() -> ThemeEntry[]` and composed by the consumer:

```bp
val th = extendTheme(defaultTheme(), paletteEntries());
val doc = await flushWith(withTheme(defaultOptions(), th));
assert themeValue(th, "--color-red-500") == "oklch(63.7% 0.237 25.331)";
```

It is a **list, not a rendered block**, and that is the point: a project that
already imports upstream's own theme leaves it out and still uses every colour
token, because a token emits `var(--color-red-500)` and whoever declares that
variable is the consumer's call. `emilia(...)` emits no `@theme` block of its
own, at any size of token list.

The values are transcribed from upstream `tailwindcss` **4.3.2**'s
`theme.css`. The reference prints exactly two of them
(`--color-red-500`, `--color-blue-500`) in the decimal-lightness spelling
`oklch(0.637 0.237 25.331)`; upstream writes the same colour as
`oklch(63.7% 0.237 25.331)`. Both are the same lightness in OKLCH; emilia
takes upstream's, so the emitted `@theme` block is byte-equal with the one a
project would otherwise import.

### A theme is a module

Upstream shares a theme between projects by importing a CSS file. In botopink a
theme is a **function in a module**, so sharing one is an ordinary package
dependency: a package exports `brandTheme() -> Theme` or a bare
`entries() -> ThemeEntry[]`, and the consumer composes it with `extendTheme`.
[`examples/emilia-theme/`](examples/emilia-theme/) is the worked example.

```bp
val composed = extendTheme(defaultTheme(), vendorEntries());
assert themeValue(composed, "--radius-pill") == "9999px";   // the vendor's
assert themeValue(composed, "--radius-lg") == "0.5rem";     // emilia's
```

## The cascade and the output

A token used to lower to a **declaration string**, and a modifier wrapped that
string in braces, so a whole class was one nested block. That shape can only
express what fits inside one class body. A token lowers to a **`Sheet`** now, and
a sheet is a list of rules that can sit anywhere in the document.

```bp
import { Rule, Block, Sheet, Variant, Options,
         emptySheet, declSheet, staticSheet, blockSheet, mergeSheet,
         declarationsOf, layerNames, nestVariant, markImportant,
         encodeSheet, decodeSheet, carriesSeparator,
         defaultOptions, withTheme, withBase, withPrefix, withImportant, withLayers,
         renderRule, renderDocument } from "emilia";
```

| Type | What it is |
| --- | --- |
| `Rule(layer, atRules, selector, declarations, important)` | one style rule — `atRules` outermost-first, `declarations` `;`-joined with no braces around it |
| `Block(header, body)` | a rule that is **not** a style rule: `@keyframes spin` plus its brace-balanced body |
| `Sheet(rules, blocks)` | what a token list produces |
| `Variant(atRule, selector)` | what a modifier is — an at-rule and a selector template, either of which may be empty |
| `Options(theme, base, prefix, important, layers)` | the build-level knobs |

### A selector is a nesting template

`selector` carries **exactly one `&`**, and `&` stands for the class. That one
field covers every variant there is, because each is a template with one `&`:

```bp
Variant(atRule: "@media (hover: hover)", selector: "&:hover")     // hover
Variant(atRule: "", selector: "[dir=\"rtl\"] &")                    // rtl
Variant(atRule: "@media (width >= 48rem)", selector: "&")         // md
Variant(atRule: "", selector: "&:is(:where(.group):hover *)")     // group-hover
```

A selector with **no** `&` is a **literal** selector — that is how the theme
writes `:root` and how a reset writes `html` — and the class name, and therefore
the prefix, never reaches it. A selector with **two or more** `&` is **refused**,
naming the selector. There is no permissive mode and no argument that relaxes it:
with none the variant replaces the rule it was meant to wrap, and with two it
duplicates it, and both are stylesheets that are silently wrong in a browser.

### Nesting runs inner-`&`-first

`nestVariant(s, v)` wraps a sheet in a variant, exactly as CSS nesting does:
the **inner** rule's `&` is what the **outer** variant's selector replaces, and
the outer variant's at-rule is prepended, so the outer modifier is outermost on
both halves.

```bp
val hover  = Variant(atRule: "", selector: "&:hover");
val before = Variant(atRule: "", selector: "&::before");

nestVariant(nestVariant(declSheet("content:\"\""), before), hover);  // &:hover::before
nestVariant(nestVariant(declSheet("content:\"\""), hover), before);  // &::before:hover
```

Blocks pass through untouched — an at-rule does not wrap a keyframes rule.

### Building a sheet

| Function | What it answers |
| --- | --- |
| `emptySheet()` | no rules, no blocks |
| `declSheet(decls)` | one `utilities` rule on the bare `&`; **`declSheet("")` is `emptySheet()`** |
| `staticSheet(layer, selector, decls)` | one rule whose selector is literal |
| `blockSheet(header, body)` | one `@keyframes` block and no rule |
| `mergeSheet(a, b)` | rules then blocks, order preserved — token order is class identity |
| `declarationsOf(s)` | every rule's declaration string, in order |
| `markImportant(s)` | set `important` on every rule; rendering appends `!important` per **declaration** |
| `layerNames()` | `["theme", "base", "components", "utilities"]` — the cascade order, written once |

### Options, and what wins

`renderRule(className, r, o)` renders one rule and `renderDocument(raw, o)` the
whole `<style>` document. The last rule in the stylesheet wins, so the order is
pinned rather than inherited from a `Map`:

1. `@layer theme, base, components, utilities;` when `layers` is on;
2. the **theme** layer — the custom-property block as a `:root` rule;
3. the **base** layer — `o.base`, front 55's reset. It is **opt-in**: the
   default is `withBase(o, [])`, a deliberate inversion of upstream's default;
4. the **components** layer;
5. the **utilities** layer — registered classes in **registration** order;
   within one class, the tokens in the order they were listed, and within one
   class the rules with **no at-rules before** the rules with at-rules, so a
   `md:` override beats the unconditioned utility on a mobile-first read;
6. the `@keyframes` blocks, **outside every layer**, deduplicated by header.
   Keyframes are not subject to the cascade, so their placement is a formatting
   choice; it is written down so it is not re-litigated.

`withLayers(o, false)` emits **no `@layer` token at all** and the same rules in
the same order, so the cascade then rests on document order alone.

`withPrefix(o, "tw_")` renders `.tw_e_1{…}`. emilia's class names are content
hashes over `[a-z0-9_]`, so a prefix is a plain concatenation: upstream's escaped
`.tw\:text-red-500` form has no analogue here, because emilia has no literal
class names to escape. That divergence is intentional.

`withImportant(o, true)` appends `!important` to every declaration of every rule,
the same thing `markImportant` does to one sheet.

[`examples/emilia-cascade/`](examples/emilia-cascade/) is the worked example:
one card whose hover is a sibling rule, whose breakpoint is a hoisted `@media`
read from the theme, whose reset arrives through `Options`, and whose document
is layered.

### The codec

A host cell stores one string per class, and a `Sheet` is a record tree, so
`encodeSheet`/`decodeSheet` carry it through: records joined by `"\n"` and tagged
`R`/`B`, fields by `"\t"`, the `atRules` list by `"\r"`. The encoding is also what
gets hashed, so a class name is a pure function of its sheet.

None of the three characters can appear in a rendered declaration. That is an
assumption, so it is **checked** and not assumed: `carriesSeparator(s)` is an
ordinary value, and the suite walks the dispatcher output and the theme's own
strings through it.

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
