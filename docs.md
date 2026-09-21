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

**All 286 cells are reachable.** Six of them — `.Color.Red.{100,500,700}` and
`.Color.Gray.{100,500,700}` — used to compile in no spelling, because the
compiler resolved a leading-dot section path by scanning every registered enum
(the synthesised section enums included) and returning the first whose tree
carried the path, without consulting the expected type: `Token` carries
`Color.Red.500` and so does `Token.Border.Color`, whose Red and Gray also run
100/500/700, so the winner was decided by hash order. It red at the call site
rather than emitting the wrong CSS. The resolver now prefers the enum the
expected type names, accepts the fully qualified `Token.Color.Red.500`, and
refuses an ambiguous path instead of guessing.

`Token.Border.Color` has since been widened to this same full grid (front 40),
so the two sections now carry the same 286 cells and the same eleven shades.
That is **not** what fixed the collision, and it would not have: the resolver
fix is what makes both reachable, and the standing rule is that a collision
like this is never resolved by adding or renaming a type to flip a hash.

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
landed, so nothing that compiled changed meaning. Front 39 did **not** fold
them in: they still emit the `background` shorthand, and changing what a
compiling path means is not something a front does to another front's surface.
Reach for `.Bg.Color.*` in new code.

```bp
.Bg.White                   // background:#ffffff
.Bg.Black                   // background:#000000
.Bg.Red.500                 // background:red
.Bg.Gray.500                // background:gray
```

### Bg — the rest of the background, `§ 10.1`–`§ 10.8`

Seven keyword sub-sections sit beside `Bg.Color`. Each is one declaration, and
none of them resolves a length or reads the theme.

```bp
.Bg.Attachment.Fixed        // background-attachment:fixed
.Bg.Attachment.Local        // background-attachment:local
.Bg.Attachment.Scroll       // background-attachment:scroll

.Bg.Clip.Border             // background-clip:border-box
.Bg.Clip.Padding            // background-clip:padding-box
.Bg.Clip.Content            // background-clip:content-box
.Bg.Clip.Text               // background-clip:text

.Bg.Origin.Border           // background-origin:border-box
.Bg.Origin.Padding          // background-origin:padding-box
.Bg.Origin.Content          // background-origin:content-box

.Bg.Pos.Bottom              // background-position:bottom
.Bg.Pos.Center              // background-position:center
.Bg.Pos.Left                // background-position:left
.Bg.Pos.LeftBottom          // background-position:left bottom
.Bg.Pos.LeftTop             // background-position:left top
.Bg.Pos.Right               // background-position:right
.Bg.Pos.RightBottom         // background-position:right bottom
.Bg.Pos.RightTop            // background-position:right top
.Bg.Pos.Top                 // background-position:top

.Bg.Repeat.Repeat           // background-repeat:repeat
.Bg.Repeat.None             // background-repeat:no-repeat
.Bg.Repeat.X                // background-repeat:repeat-x
.Bg.Repeat.Y                // background-repeat:repeat-y
.Bg.Repeat.Round            // background-repeat:round
.Bg.Repeat.Space            // background-repeat:space

.Bg.Size.Auto               // background-size:auto
.Bg.Size.Cover              // background-size:cover
.Bg.Size.Contain            // background-size:contain

.Bg.Image.None              // background-image:none
```

Three names are worth reading twice:

- **`Pos`, not `Position`.** `.Bg.Pos.Center` sets `background-position` and
  `.Layout.Position.Fixed` sets `position`; Tailwind spells both with the same
  English word and they are unrelated properties. The short head keeps the path
  four segments and keeps the two apart at a glance.
- **`Repeat.None`, not `Repeat.NoRepeat`.** The CSS value keeps its `no-`
  prefix; the token does not repeat the word its own section already says.
- **`Clip.Text` is the one clip value that is not a `*-box`.** The suffix is
  written per arm rather than derived from the leaf name, because a derived
  rule would emit `text-box` — a different value that also exists.

`.Bg.Size.*` and the top-level `Size` section are two different things:
`background-size` and the element's own `width`/`height` ladder. The same
shape as `.Bg.Color` beside the top-level `Color`.

Nothing else in `§ 10.4` is a token: `bg-[url(…)]`, `bg-size-[…]` and
`bg-position-[…]` are arbitrary values and belong to the escape-hatch front.

### Gradient — `§ 10.4`

Gradients are their own top-level section, not a sub-section of `Bg`. The stop
colours are not `background-image` at all — they are custom properties the
gradient reads — so nesting them under `Bg.Image` would put three properties
under one name; and a stop is a four-segment path already.

**`To` is the direction. `Stop` is the terminal colour.** Tailwind spells both
with the word `to` (`bg-gradient-to-r` and `to-pink-500`) while they set
unrelated things; emilia does not.

```bp
.Gradient.To.T              // background-image:linear-gradient(to top, var(--tw-gradient-stops))
.Gradient.To.Tr             // … linear-gradient(to top right, …)
.Gradient.To.R              // … linear-gradient(to right, …)
.Gradient.To.Br             // … linear-gradient(to bottom right, …)
.Gradient.To.B              // … linear-gradient(to bottom, …)
.Gradient.To.Bl             // … linear-gradient(to bottom left, …)
.Gradient.To.L              // … linear-gradient(to left, …)
.Gradient.To.Tl             // … linear-gradient(to top left, …)
```

A corner is **two keywords** — `to top right`, never `to top-right` — and there
is exactly one space after the comma. The eight phrases are spelled in one
place in `emilia.bp`, so adding a direction is one arm.

#### The stops

`From`, `Via` and `Stop` each carry the **whole of front 33's grid** — the 26
families over eleven shades, plus `White`, `Black`, `Transparent`, `Current`
and `Inherit`. The colour half is `paletteVar(family, shade)`, the same
function `.Bg.Color.*` calls, so a stop and a background reference ONE custom
property by construction:

```bp
.Gradient.From.Indigo.500
// --tw-gradient-from:var(--color-indigo-500);
// --tw-gradient-stops:var(--tw-gradient-from), var(--tw-gradient-to, transparent)

.Gradient.Via.Purple.500
// --tw-gradient-via:var(--color-purple-500);
// --tw-gradient-stops:var(--tw-gradient-from, transparent), var(--tw-gradient-via), var(--tw-gradient-to, transparent)

.Gradient.Stop.Pink.500
// --tw-gradient-to:var(--color-pink-500)
```

A full gradient is a direction and two or three stops in one list:

```bp
emilia([.Gradient.To.R, .Gradient.From.Indigo.500, .Gradient.Stop.Pink.500])
```

**Token order is load-bearing here**, and deliberately so. `Via` writes a
three-stop list and `From` a two-stop one; whichever is listed LAST wins, which
is how a `From` + `Via` + `Stop` triple ends up with the three-colour list.
`Stop` writes no list at all — if it did, it would overwrite `Via`'s.

#### What the stop shape does and does not take from upstream

The three custom-property names are upstream's, checked against it:
`--tw-gradient-from`, `--tw-gradient-via`, `--tw-gradient-to`, read by
`var(--tw-gradient-stops)` in the direction.

The **composition** is simpler than upstream's on purpose. Tailwind v4 threads
four position variables through the list and gives `via-*` its own
`--tw-gradient-via-stops`, and both rest on `@property` registration for their
defaults. emilia emits no `@property` block, and colour-stop positions
(`from-10%`) are not tokens here, so copying that shape would emit a list that
is invalid at computed-value time in every browser.

What is kept is the part that makes the simplification correct: upstream
registers `#0000` as each stop's default, and emilia writes that as a
`var(…, transparent)` **fallback**. So `.Gradient.From.Indigo.500` on its own
still paints indigo → transparent rather than resolving to nothing.

Radial and conic gradients (`bg-radial`, `bg-conic`), the interpolation
suffixes (`bg-linear-to-r/oklch`) and colour-stop positions (`from-10%`) are
**not** tokens: a radial gradient is a different function with a position
argument, and specifying it from memory would be guessing.

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

### Space — spacing between children

`space-x-*` and `space-y-*` are **not declarations on the element**. They set a
margin on its children, so a `Space` token produces a rule with its own
selector:

```bp
val cls = emilia([.Pad.All.4, .Space.Y.4]);
await flush();
// .e_1f2{padding:calc(var(--spacing) * 4)}
// .e_1f2 > :not(:last-child){margin-block-end:calc(var(--spacing) * 4)}
```

Two rules of one class, because they do not describe the same elements. `X` is
`margin-inline-end`, `Y` is `margin-block-end` — the logical pair, for the same
reason `Pad.S`/`Pad.E` exist. The scale is the one `Pad` and `Margin` use,
`Neg` included, and `.Space.XReverse` / `.Space.YReverse` set upstream's
`--tw-space-x-reverse` / `--tw-space-y-reverse`.

The selector is a nesting template carrying exactly one `&`, so a modifier
wraps it rather than replacing it:

```bp
Token.Hover([.Space.Y.4])
// &:hover > :not(:last-child){margin-block-end:calc(var(--spacing) * 4)}
```

**Recorded, not hidden:** `space-x-*` is absent from the local Tailwind
reference entirely, so the child selector and the property are this front's
proposal rather than a transcription, and the `--tw-space-*-reverse` names are
upstream-internal and unverified. Both are pinned by a test, so changing them
is a visible change.

### Layout — `§ 5`

`Layout` is two things at once. The **eleven display values are the section's
own leaves**, so `.Layout.Flex` is a display value and not a flex container:

```bp
.Layout.Block        // display:block
.Layout.InlineBlock  // display:inline-block
.Layout.Inline       // display:inline
.Layout.Flex         // display:flex
.Layout.InlineFlex   // display:inline-flex
.Layout.Grid         // display:grid
.Layout.InlineGrid   // display:inline-grid
.Layout.Contents     // display:contents
.Layout.FlowRoot     // display:flow-root
.Layout.ListItem     // display:list-item
.Layout.Hidden       // display:none
```

`hidden` is the one row whose Tailwind name is not its CSS value: `.Layout.Hidden`
is `display:none`. The flex and grid CONTAINER properties — direction, wrap,
alignment, `gap` — are `Token.Flex`'s, not `Layout`'s.

Everything else in `§ 5` is a **sub-section** beside those leaves.

#### Position and inset

```bp
.Layout.Position.Absolute   // position:absolute
.Layout.Position.Sticky     // position:sticky

.Layout.Inset.All.0         // inset:0
.Layout.Inset.X.0           // left:0;right:0
.Layout.Inset.Y.0           // top:0;bottom:0
.Layout.Inset.T.4           // top:calc(var(--spacing) * 4)
.Layout.Inset.T.Neg.4       // top:calc(var(--spacing) * -4)
.Layout.Inset.T.Half.0      // top:calc(var(--spacing) * 0.5)
.Layout.Inset.T.Frac.Half   // top:50%
.Layout.Inset.T.Full        // top:100%
.Layout.Inset.T.Auto        // top:auto
.Layout.Inset.S.0           // inset-inline-start:0
.Layout.Inset.E.0           // inset-inline-end:0
```

`Inset` carries the **nine directions** `Pad` and `Margin` carry — `All`, `X`,
`Y`, `T`, `R`, `B`, `L` and the logical pair `S`/`E` — over the **same scale**,
answering the **same** `spacing(n)` / `spacingHalf(n)`. That is not a
coincidence to be maintained: `.Layout.Inset.T.4` and `.Pad.T.4` differ only in
the property name, and a test asserts the two length strings are equal.

`inset` is a real CSS shorthand, so `All` is one declaration. `left`/`right` and
`top`/`bottom` have none, so `X` and `Y` expand to two — which is how `§ 5.17`
prints `inset-x-0`.

#### Overflow, overscroll, visibility, z-index, isolation

```bp
.Layout.Overflow.Hidden        // overflow:hidden
.Layout.Overflow.Y.Auto        // overflow-y:auto
.Layout.Overscroll.Contain     // overscroll-behavior:contain
.Layout.Overscroll.X.None      // overscroll-behavior-x:none
.Layout.Visibility.Invisible   // visibility:hidden   ← not `invisible`
.Layout.Visibility.Collapse    // visibility:collapse
.Layout.Z.50                   // z-index:50
.Layout.Z.Auto                 // z-index:auto
.Layout.Isolation.Isolate      // isolation:isolate
```

Two of these are places a transcription slips, so each has its own assertion:
`invisible` is the **utility** name and `hidden` is the **CSS value**, and `Z`
is the one numeric family in `Layout` that is **not a length** — `z-50` is the
bare integer `50`, never a `calc` and never a `rem`.

#### Float, clear, replaced content

```bp
.Layout.Float.Start               // float:inline-start   ← not `start`
.Layout.Float.Left                // float:left
.Layout.Clear.Start               // clear:inline-start
.Layout.Clear.Both                // clear:both
.Layout.Object.Fit.Cover          // object-fit:cover
.Layout.Object.Pos.LeftBottom     // object-position:left bottom  ← one space
.Layout.Aspect.Square             // aspect-ratio:1 / 1
.Layout.Aspect.Video              // aspect-ratio:16 / 9
```

Three more traps, each pinned: the **utility** name is `float-start`, the CSS
**value** is `inline-start`; the hyphen in `object-left-bottom` belongs to the
utility name and the CSS value is two words separated by one space; and the
aspect ratios keep the spaces around the slash the way `§ 5.1` prints them.

#### Multi-column, fragmentation, box-sizing

```bp
.Layout.Columns.2                  // columns:2
.Layout.Columns.Md                 // columns:var(--container-md)
.Layout.BreakAfter.Page            // break-after:page
.Layout.BreakInside.AvoidColumn    // break-inside:avoid-column
.Layout.Box.Border                 // box-sizing:border-box
.Layout.BoxDecoration.Clone        // box-decoration-break:clone
```

A column **count** is a plain integer. A column **width** is the theme's
container ladder — `.Layout.Columns.Md` and `.Size.MaxW.Md` are the same width
by definition, so they are the same reference, and a project that overrides
`--container-md` moves both. No `Columns` leaf spells a `rem`.

`break-inside` carries a shorter leaf set than `break-after` / `break-before`:
`§ 5.5` has no `all`, no `page`, no `left` and no `right`.

The three are **flat sub-sections**, not `Break { After, Before, Inside }`. A
section head named like a top-level modifier variant silently shadows that
variant's payload, and `After`/`Before` are modifiers — see `AGENTS.md`
§ Maintainer rules.

**Not declared:** `columns-4` … `columns-12`, which resolve upstream through the
bare-integer rule rather than through a theme key; arbitrary `aspect-[4/3]` and
`z-[999]`, which belong to the escape-hatch front.

#### Nothing in `Layout` resolves a length

`Layout` declares 776 leaves, and a test walks **every one of them** asserting
the emitted declaration carries no `rem`, carries a `:`, and carries no Tailwind
class fragment. `Inset` goes through `spacing(n)` / `spacingHalf(n)`; the named
column widths go through `var(--container-*)`; `Z` is a bare integer. Halving
`--spacing` or changing `--container-md` moves the `:root` block and not one
byte of any `Layout` rule.

`examples/emilia-layout/` is the worked example: a media card that clips its
overflow, isolates a stacking context and crops a 16:9 image; and a sticky
header over a one-axis scroll panel with a badge on a negative inset.

### Flex — `§ 6`

`Flex` is the flex CONTAINER and the flex ITEM. `display:flex` is not here — it
is a display value and lives on `Layout` (`.Layout.Flex`); this section is
everything a box that already declares it can say next.

#### Direction and wrap

```bp
.Flex.Row          // flex-direction:row
.Flex.RowReverse   // flex-direction:row-reverse
.Flex.Col          // flex-direction:column
.Flex.ColReverse   // flex-direction:column-reverse
.Flex.Wrap         // flex-wrap:wrap
.Flex.WrapReverse  // flex-wrap:wrap-reverse
.Flex.NoWrap       // flex-wrap:nowrap
```

#### The shorthand, grow, shrink, basis and order

```bp
.Flex.Value.One       // flex:1 1 0%
.Flex.Value.Auto      // flex:1 1 auto
.Flex.Value.Initial   // flex:0 1 auto
.Flex.Value.None      // flex:none      ← the keyword, not `0 0 auto`

.Flex.Grow.1          // flex-grow:1
.Flex.Grow.0          // flex-grow:0
.Flex.Shrink.1        // flex-shrink:1
.Flex.Shrink.0        // flex-shrink:0

.Flex.Basis.0         // flex-basis:0
.Flex.Basis.4         // flex-basis:calc(var(--spacing) * 4)
.Flex.Basis.Half.1    // flex-basis:calc(var(--spacing) * 1.5)
.Flex.Basis.Px        // flex-basis:1px
.Flex.Basis.Auto      // flex-basis:auto
.Flex.Basis.Full      // flex-basis:100%
.Flex.Basis.Frac.Third // flex-basis:33.333333%

.Flex.Order.1         // order:1
.Flex.Order.12        // order:12
.Flex.Order.First     // order:-9999
.Flex.Order.Last      // order:9999
.Flex.Order.None      // order:0        ← `order:none` is not a value
```

The sub-section is `Value` and not `Flex`, because a section cannot carry a
sub-section of its own name.

`Basis` is the only family here that is a **length**, and it is front 35's
scale answering front 54's `spacing(n)` / `spacingHalf(n)` — `.Flex.Basis.4`
and `.Pad.All.4` carry the same length text because they call the same
function. Its fractions are front 35's percentages to the last decimal, and a
test compares `.Flex.Basis.Frac.Third` to `.Size.W.Frac.Third` rather than to a
literal.

`Grow`, `Shrink` and `Order` are **bare numbers**: `order-1` is `order:1`, a
count and not a length, so nothing in them reaches the theme. `order-first`
and `order-last` are the sentinels `-9999` and `9999`, and `order-none` is `0`.

#### Alignment — nine property groups, all under `Flex`

```bp
.Flex.Justify.Normal        // justify-content:normal
.Flex.Justify.Start         // justify-content:flex-start
.Flex.Justify.Between       // justify-content:space-between
.Flex.Justify.Evenly        // justify-content:space-evenly
.Flex.Justify.Stretch       // justify-content:stretch

.Flex.Items.Center          // align-items:center
.Flex.Items.Baseline        // align-items:baseline

.Flex.AlignSelf.Start       // align-self:flex-start
.Flex.Content.Between       // align-content:space-between

.Flex.JustifyItems.Start    // justify-items:start     ← not `flex-start`
.Flex.JustifySelf.Center    // justify-self:center

.Flex.PlaceContent.Between  // place-content:space-between
.Flex.PlaceItems.Center     // place-items:center
.Flex.PlaceSelf.Stretch     // place-self:stretch
```

**Alignment applies to grid as much as to flex, and it lives under `Flex`
anyway.** `align-items` and `justify-content` were spelled that way before this
front, and renaming a token that compiles today is what the milestone forbids.
A grid container writes `.Flex.Justify.Center` and gets
`justify-content:center`, which is the correct CSS for a grid; only the token's
spelling reads as though it were flex-only.

**Upstream is not internally consistent and emilia copies it rather than
smoothing it.** `justify-content`, `align-items`, `align-self` and
`align-content` take `flex-start` / `flex-end`; `justify-items`,
`justify-self` and the whole `place-*` family take `start` / `end`. One test is
dedicated to the asymmetry, asserting both spellings come out of the right
tokens.

`AlignSelf` and the flat `PlaceContent` / `PlaceItems` / `PlaceSelf`, rather
than `Self` and a nested `Place { … }`: **`Self` is a language keyword**, so a
section cannot be named it. The family is flattened the way `Break` is, and no
emitted byte differs.

### Grid — `§ 6.8`–`§ 6.14`

`display:grid` is `Layout`'s (`.Layout.Grid`) and used to be all emilia had:
there was no template, no span, no start, no end, no flow and no implicit
track, so nothing could be put inside the box it declares.

```bp
.Grid.Cols.12        // grid-template-columns:repeat(12, minmax(0, 1fr))
.Grid.Cols.None      // grid-template-columns:none
.Grid.Cols.Subgrid   // grid-template-columns:subgrid
.Grid.Rows.3         // grid-template-rows:repeat(3, minmax(0, 1fr))

.Grid.Col.Auto       // grid-column:auto
.Grid.Col.Span.2     // grid-column:span 2 / span 2
.Grid.Col.Span.Full  // grid-column:1 / -1
.Grid.Col.Start.13   // grid-column-start:13
.Grid.Col.End.Auto   // grid-column-end:auto
.Grid.Row.Span.2     // grid-row:span 2 / span 2
.Grid.Row.End.3      // grid-row-end:3

.Grid.Flow.Col       // grid-auto-flow:column   ← `col` abbreviates, CSS does not
.Grid.Flow.RowDense  // grid-auto-flow:row dense   ← one space, not a hyphen
.Grid.AutoCols.Min   // grid-auto-columns:min-content
.Grid.AutoRows.Fr    // grid-auto-rows:minmax(0, 1fr)
```

A template is a **function of the leaf**, not a lookup: `gridRepeat(n)` builds
`repeat(N, minmax(0, 1fr))` — one space after each comma — and it is the only
place that text is spelled. The `minmax(0, 1fr)` inside it is `gridFr()`, the
same string the implicit `fr` tracks read, so the two cannot drift.

`Cols`, `Rows` and `Span` run **1 … 12**; `Start` and `End` run **1 … 13**,
because a twelve-column grid has thirteen lines.

**Nothing in `Grid` is a length.** A column count, a span and a line number are
integers, so no leaf reaches `spacing(n)` and none spells a `rem`.

**Not declared:** every arbitrary-value form (`grid-cols-[200px_1fr]`,
`col-start-[7]`), which belongs to the escape-hatch front.

### Gap — `§ 6.15`

```bp
.Gap.All.0        // gap:0
.Gap.All.4        // gap:calc(var(--spacing) * 4)
.Gap.All.Half.1   // gap:calc(var(--spacing) * 1.5)
.Gap.All.Px       // gap:1px
.Gap.X.2          // column-gap:calc(var(--spacing) * 2)
.Gap.Y.6          // row-gap:calc(var(--spacing) * 6)
```

`Gap` is a **top-level section**, not a sub-section of `Flex`, because `gap`,
`column-gap` and `row-gap` separate the items of a grid exactly as they separate
the items of a flex row. It carries front 35's full scale — the thirty
multipliers `0 … 96`, the `Px` step and `Half { 0, 1, 2, 3 }` — on each of
`All`, `X` and `Y`.

`.Gap.All.4` and `.Pad.All.4` carry the same length because both answer
`spacing(4)`. Overriding `--spacing` moves both and rewrites neither rule.

The pre-front-37 `.Flex.Gap.{1,2,4,8}` paths **still compile and still emit
`gap:`** — the same declaration by a narrower name. They used to answer a
hand-written `rem` ladder (`gap-4` was `gap:1rem`, the last of the seven front
54 found); they answer `spacing(n)` now, and a test compares the two spellings
to **each other** so they cannot drift apart again.

#### Nothing in `Flex`, `Grid` or `Gap` resolves a length

The three sections declare **356 leaves** (126 + 125 + 105), and a test walks
every one of them asserting the emitted declaration carries no `rem`, carries a
`:`, and carries no Tailwind class fragment. Only `Flex.Basis` and `Gap` are
lengths and both go through `spacing(n)` / `spacingHalf(n)`; a grow factor, an
order, a column count, a span and a line number are bare integers. The same
test asserts `.Text.Size.Lg` DOES carry a `rem`, so the probe is known to
discriminate rather than to pass vacuously. It used to assert that of
`.Border.Rounded.Lg`, until front 40 rewrote the radius ladder to reference the
theme.

`examples/emilia-grid/` is the worked example: a toolbar whose heading takes the
remaining space and which stacks under `sm`, and a twelve-column dashboard that
reflows one → six → twelve across `md` and `lg`.

### Border — `§ 11.1`–`§ 11.4`

```bp
.Border.W.1                 // border-width:1px
.Border.W.0                 // border-width:0
.Border.W.8                 // border-width:8px
.Border.W.X.1               // border-left-width:1px;border-right-width:1px
.Border.W.Y.2               // border-top-width:2px;border-bottom-width:2px
.Border.W.T.4               // border-top-width:4px
.Border.W.S.1               // border-inline-start-width:1px
.Border.W.E.1               // border-inline-end-width:1px

.Border.Style.Solid         // border-style:solid
.Border.Style.Dashed        // border-style:dashed
.Border.Style.Hidden        // border-style:hidden
.Border.Style.None          // border-style:none

.Border.Color.Slate.200     // border-color:var(--color-slate-200)
.Border.Color.White         // border-color:var(--color-white)
.Border.Color.Transparent   // border-color:transparent
.Border.Color.Current       // border-color:currentColor
```

An **axis** is two declarations and a **side** is one, because CSS has no
`border-x-width`. The logical pair `S`/`E` is
`border-inline-start-width`/`-end-width`, which follows the writing direction
where `L`/`R` do not — the same distinction `Pad.S`/`Pad.E` draws.

`Border.Color` carries the whole of [the palette](#color--the-palette): 26
families × 11 shades, plus `White`/`Black`/`Transparent`/`Current`/`Inherit`.
Every cell goes through the same `paletteVar(family, shade)` that `Color`,
`Bg.Color`, `Outline.Color`, `Ring.Color` and `Divide.Color` use, so
`.Border.Color.Emerald.600` and `.Color.Emerald.600` reference **one** custom
property and cannot drift apart.

> **This changed what a compiling path means.** Before front 40 the dispatcher
> DISCARDED the shade — every cell emitted `border-color:red` or
> `border-color:gray`. Three shades of one family were therefore ONE class,
> since the class name is a hash of the body. If you pinned that text, it moved.

`.Border.Color.Hex(value)` is declared and, like every payload leaf nested in a
section, **cannot be constructed** — see
[`Color.Hex` is declared and unconstructible](#colorhexabc-is-declared-and-unconstructible).

#### Border radius — `§ 11.1`, `§ 21.5`

```bp
.Border.Rounded.None        // border-radius:0
.Border.Rounded.Sm          // border-radius:var(--radius-sm)
.Border.Rounded.Lg          // border-radius:var(--radius-lg)
.Border.Rounded.X2xl        // border-radius:var(--radius-2xl)
.Border.Rounded.Full        // border-radius:9999px

.Border.Rounded.T.Lg        // border-top-left-radius:var(--radius-lg);
                            // border-top-right-radius:var(--radius-lg)
.Border.Rounded.Tl.Lg       // border-top-left-radius:var(--radius-lg)
.Border.Rounded.B.None      // border-bottom-right-radius:0;
                            // border-bottom-left-radius:0
.Border.Rounded.Ss.Md       // border-start-start-radius:var(--radius-md)
```

Ten leaves — `None`, `Xs`, `Sm`, `Md`, `Lg`, `Xl`, `X2xl`, `X3xl`, `X4xl`,
`Full` — on the shorthand **and on each of fourteen directional sub-sections**:
the four sides `T`/`R`/`B`/`L`, the four physical corners `Tl`/`Tr`/`Br`/`Bl`,
and the six logical ones `S`/`E`/`Ss`/`Se`/`Es`/`Ee`. A **side** form emits two
corner properties and a **corner** form emits one, so `.Border.Rounded.Tl.Full`
is as much a path as `.Border.Rounded.Tl.Lg` is.

> **This changed too.** `Sm`, `Md` and `Lg` used to resolve a literal `rem`;
> they reference `var(--radius-*)` now, so overriding `--radius-lg` moves every
> rounded corner. `Full` and `None` do **not** change — upstream prints those two
> literally, and neither is a theme entry.

`examples/emilia-borders/` is the worked example: one width per side, the two
axes, the six styles, the palette with its shade, and a table-like card with a
rounded top, a square bottom and a dashed internal rule.

### Outline — `§ 11.5`–`§ 11.8`

```bp
.Outline.W.2                // outline-width:2px
.Outline.Style.Solid        // outline-style:solid
.Outline.Style.Dashed       // outline-style:dashed
.Outline.Color.Blue.500     // outline-color:var(--color-blue-500)
.Outline.Offset.2           // outline-offset:2px
.Outline.Offset.Neg.1       // outline-offset:-1px

.Outline.Style.None         // outline:2px solid transparent;outline-offset:2px
```

An outline is painted **outside** the border box and **takes no space**, which
is why a focus ring is an outline and not a border: focusing a control moves
nothing beside it.

```bp
val focus: Token[] = [.Outline.W.2, .Outline.Color.Indigo.500];
val tokens: Token[] = [.Border.W.1, Token.FocusVisible(focus)];
emilia(tokens);
// .e_x{border-width:1px}
// .e_x:focus-visible{outline-width:2px;outline-color:var(--color-indigo-500)}
```

> **`.Outline.Style.None` is the trap.** It is **not** `outline-style:none`.
> Upstream prints it as a *transparent* outline plus an offset, so a
> high-contrast mode still renders a ring where the design removed one. The
> token sits under `Style` because that is where upstream puts it; only the
> emission is special.

A negative offset is a leaf on a `Neg` sub-section and not a sign on a number,
because a numeric enum leaf is a run of digits.

### Ring — the box-shadow half of a focus ring

```bp
.Ring.W.2                   // --tw-ring-shadow:0 0 0 2px;
                            // box-shadow:var(--tw-ring-offset-shadow),
                            //            var(--tw-ring-shadow), var(--tw-shadow)
.Ring.Color.Indigo.500      // --tw-ring-color:var(--color-indigo-500)
.Ring.Offset.W.2            // --tw-ring-offset-width:2px
.Ring.Offset.Color.White    // --tw-ring-offset-color:var(--color-white)
.Ring.Inset                 // --tw-ring-inset:inset
```

A ring **composes with a shadow rather than overwriting it**: the `box-shadow`
declaration *lists* `var(--tw-shadow)` instead of writing a shadow of its own,
so a `Ring` token and an `Effect.Shadow` token in the same list both reach the
output. Token order is class identity, so swapping the two is a different class
— which is the sharpest case [contract 4](#stable-hashes--sites-collapse) has.

`.Ring.W.1` is upstream's bare `ring`. **The v4 default is 1px**, where v3's was
3px.

> **Verified and unverified.** The custom property names, the inset flag and the
> v4 default width were checked against upstream. The composed `box-shadow`
> list and the whole `ring-offset-*` family were **not** confirmed — v4's
> documentation no longer carries `ring-offset-*` at all. Both are declared
> because the surface asks for them, and both are recorded here rather than
> presented as transcribed.

### Divide — borders between children

```bp
.Divide.Y.1                 // & > :not(:last-child) {
                            //   border-top-width:0px;border-bottom-width:1px }
.Divide.X.2                 // & > :not(:last-child) {
                            //   border-inline-start-width:0px;
                            //   border-inline-end-width:2px }
.Divide.Color.Slate.200     // & > :not(:last-child) {
                            //   border-color:var(--color-slate-200) }
.Divide.Style.Dashed        // & > :not(:last-child) { border-style:dashed }
.Divide.XReverse            // & > :not(:last-child) { --tw-divide-x-reverse:1 }
```

`divide-*` declares on the element's **children**, not on the element — a
border on every child but the last — so the class body itself is **empty**:

```bp
val list: Token[] = [.Divide.Y.1, .Divide.Color.Gray.200];
emilia(list);
// .e_x > :not(:last-child){border-top-width:0px;border-bottom-width:1px;
//                          border-color:var(--color-gray-200)}
```

A width, a colour and a style all target the **same** selector, so they merge
into one child rule rather than three.

That selector is the same one [`Space`](#space--spacing-between-children) uses,
and it is spelled in exactly one place in the library — `siblingSelector()`.
`divide-*` and `space-y-*` separate the same children, and a test compares the
two families' output **byte for byte** so that the two can never be written
differently.

`examples/emilia-outline-ring/` is the worked example for all three sections.

#### Nothing in `Border`, `Outline`, `Ring` or `Divide` resolves a colour or a radius

The four sections declare **1704 leaves** (492 + 310 + 593 + 309), and a test
walks every one of them asserting the declaration carries no `rem`, no hex and
no colour function; carries a `:`; and carries no Tailwind class fragment. Each
walk has a control that must fail, and one further walk asserts that all 1440
colour cells carry a `var(--color-` reference and that a section's 288 are 288
**distinct** strings — because a literal probe alone does not catch a dispatcher
that discards its shade, which is exactly the defect this section used to have.

### Modifiers — the variant table

A modifier is the only way a token reaches a state, a breakpoint or a
pseudo-element. Each one carries a `Token[]` payload and is **not** a block
nested inside the class body — it is a `Variant` (an at-rule and a selector
template with exactly one `&`), and the tokens it carries become **sibling
rules** hoisted out of the class with their own selector and at-rule.

```bp
emilia([.Bg.Color.White, Token.Dark([.Bg.Color.Slate.900])])
// .e_x{background-color:var(--color-white)}
// @media (prefers-color-scheme: dark){.e_x{background-color:var(--color-slate-900)}}
```

Modifiers nest, and the OUTER one is outermost in the selector and in the
at-rule list alike — which is how a `dark:md:hover:` chain reads upstream:

```bp
Token.Dark([Token.Md([Token.Hover([.Text.Bold])])])
// @media (prefers-color-scheme: dark){@media (width >= 48rem){
//   @media (hover: hover){.e_x:hover{font-weight:bold}}}}
```

A breakpoint RANGE is nesting too, not a variant of its own — upstream's
`md:max-xl:` is `Token.Md([Token.MaxXl([…])])`. Every breakpoint reads the
theme's `--breakpoint-*` entry, both ways round, so overriding the ladder
moves the queries and the class hashes with it.

**Breakpoints — min-width, read from the theme's `--breakpoint-*` ladder**

| upstream | emilia | CSS |
| --- | --- | --- |
| `sm:` | `Token.Sm(inner)` | `@media (width >= 40rem){…}` |
| `md:` | `Token.Md(inner)` | `@media (width >= 48rem){…}` |
| `lg:` | `Token.Lg(inner)` | `@media (width >= 64rem){…}` |
| `xl:` | `Token.Xl(inner)` | `@media (width >= 80rem){…}` |
| `2xl:` | `Token.X2xl(inner)` | `@media (width >= 96rem){…}` |

**Breakpoints — max-width, the same ladder read the other way**

| upstream | emilia | CSS |
| --- | --- | --- |
| `max-sm:` | `Token.MaxSm(inner)` | `@media (width < 40rem){…}` |
| `max-md:` | `Token.MaxMd(inner)` | `@media (width < 48rem){…}` |
| `max-lg:` | `Token.MaxLg(inner)` | `@media (width < 64rem){…}` |
| `max-xl:` | `Token.MaxXl(inner)` | `@media (width < 80rem){…}` |
| `max-2xl:` | `Token.MaxX2xl(inner)` | `@media (width < 96rem){…}` |

**Dark mode and the other media features**

| upstream | emilia | CSS |
| --- | --- | --- |
| `dark:` | `Token.Dark(inner)` | `@media (prefers-color-scheme: dark){…}` |
| `print:` | `Token.Print(inner)` | `@media print{…}` |
| `portrait:` | `Token.Portrait(inner)` | `@media (orientation: portrait){…}` |
| `landscape:` | `Token.Landscape(inner)` | `@media (orientation: landscape){…}` |
| `motion-safe:` | `Token.MotionSafe(inner)` | `@media (prefers-reduced-motion: no-preference){…}` |
| `motion-reduce:` | `Token.MotionReduce(inner)` | `@media (prefers-reduced-motion: reduce){…}` |
| `contrast-more:` | `Token.ContrastMore(inner)` | `@media (prefers-contrast: more){…}` |
| `contrast-less:` | `Token.ContrastLess(inner)` | `@media (prefers-contrast: less){…}` |
| `forced-colors:` | `Token.ForcedColors(inner)` | `@media (forced-colors: active){…}` |

**Interaction state**

| upstream | emilia | CSS |
| --- | --- | --- |
| `hover:` | `Token.Hover(inner)` | `@media (hover: hover){&:hover{…}}` |
| `focus:` | `Token.Focus(inner)` | `&:focus{…}` |
| `focus-within:` | `Token.FocusWithin(inner)` | `&:focus-within{…}` |
| `focus-visible:` | `Token.FocusVisible(inner)` | `&:focus-visible{…}` |
| `active:` | `Token.Active(inner)` | `&:active{…}` |
| `visited:` | `Token.Visited(inner)` | `&:visited{…}` |
| `target:` | `Token.Target(inner)` | `&:target{…}` |
| `open:` | `Token.Open(inner)` | `&:is(:open, :popover-open){…}` |
| `inert:` | `Token.Inert(inner)` | `&:is([inert], [inert] *){…}` |

**Form state**

| upstream | emilia | CSS |
| --- | --- | --- |
| `disabled:` | `Token.Disabled(inner)` | `&:disabled{…}` |
| `enabled:` | `Token.Enabled(inner)` | `&:enabled{…}` |
| `checked:` | `Token.Checked(inner)` | `&:checked{…}` |
| `indeterminate:` | `Token.Indeterminate(inner)` | `&:indeterminate{…}` |
| `default:` | `Token.Default(inner)` | `&:default{…}` |
| `optional:` | `Token.Optional(inner)` | `&:optional{…}` |
| `required:` | `Token.Required(inner)` | `&:required{…}` |
| `valid:` | `Token.Valid(inner)` | `&:valid{…}` |
| `invalid:` | `Token.Invalid(inner)` | `&:invalid{…}` |
| `user-valid:` | `Token.UserValid(inner)` | `&:user-valid{…}` |
| `user-invalid:` | `Token.UserInvalid(inner)` | `&:user-invalid{…}` |
| `in-range:` | `Token.InRange(inner)` | `&:in-range{…}` |
| `out-of-range:` | `Token.OutOfRange(inner)` | `&:out-of-range{…}` |
| `placeholder-shown:` | `Token.PlaceholderShown(inner)` | `&:placeholder-shown{…}` |
| `autofill:` | `Token.Autofill(inner)` | `&:autofill{…}` |
| `read-only:` | `Token.ReadOnly(inner)` | `&:read-only{…}` |

**Structural position**

| upstream | emilia | CSS |
| --- | --- | --- |
| `first:` | `Token.First(inner)` | `&:first-child{…}` |
| `last:` | `Token.Last(inner)` | `&:last-child{…}` |
| `only:` | `Token.Only(inner)` | `&:only-child{…}` |
| `odd:` | `Token.Odd(inner)` | `&:nth-child(odd){…}` |
| `even:` | `Token.Even(inner)` | `&:nth-child(even){…}` |
| `first-of-type:` | `Token.FirstOfType(inner)` | `&:first-of-type{…}` |
| `last-of-type:` | `Token.LastOfType(inner)` | `&:last-of-type{…}` |
| `only-of-type:` | `Token.OnlyOfType(inner)` | `&:only-of-type{…}` |
| `empty:` | `Token.Empty(inner)` | `&:empty{…}` |
| `nth-N:` | `Token.Nth(index, inner)` | `&:nth-child(3){…}` |
| `nth-last-N:` | `Token.NthLast(index, inner)` | `&:nth-last-child(5){…}` |

**Pseudo-elements**

| upstream | emilia | CSS |
| --- | --- | --- |
| `before:` | `Token.Before(inner)` | `&::before{…}` |
| `after:` | `Token.After(inner)` | `&::after{…}` |
| `first-letter:` | `Token.FirstLetter(inner)` | `&::first-letter{…}` |
| `first-line:` | `Token.FirstLine(inner)` | `&::first-line{…}` |
| `placeholder:` | `Token.Placeholder(inner)` | `&::placeholder{…}` |
| `file:` | `Token.File(inner)` | `&::file-selector-button{…}` |
| `marker:` | `Token.Marker(inner)` | `& ::marker{…}` |
| `selection:` | `Token.Selection(inner)` | `& ::selection{…}` |
| `backdrop:` | `Token.Backdrop(inner)` | `&::backdrop{…}` |

**Parent state — the `.group` class is the consumer's, never emilia's**

| upstream | emilia | CSS |
| --- | --- | --- |
| `group-hover:` | `Token.GroupHover(inner)` | `&:is(:where(.group):hover *){…}` |
| `group-focus:` | `Token.GroupFocus(inner)` | `&:is(:where(.group):focus *){…}` |
| `group-active:` | `Token.GroupActive(inner)` | `&:is(:where(.group):active *){…}` |
| `group-visited:` | `Token.GroupVisited(inner)` | `&:is(:where(.group):visited *){…}` |
| `group-disabled:` | `Token.GroupDisabled(inner)` | `&:is(:where(.group):disabled *){…}` |
| `group-open:` | `Token.GroupOpen(inner)` | `&:is(:where(.group):open *){…}` |

**Sibling state — the `.peer` class is the consumer's, never emilia's**

| upstream | emilia | CSS |
| --- | --- | --- |
| `peer-hover:` | `Token.PeerHover(inner)` | `&:is(:where(.peer):hover ~ *){…}` |
| `peer-focus:` | `Token.PeerFocus(inner)` | `&:is(:where(.peer):focus ~ *){…}` |
| `peer-active:` | `Token.PeerActive(inner)` | `&:is(:where(.peer):active ~ *){…}` |
| `peer-checked:` | `Token.PeerChecked(inner)` | `&:is(:where(.peer):checked ~ *){…}` |
| `peer-invalid:` | `Token.PeerInvalid(inner)` | `&:is(:where(.peer):invalid ~ *){…}` |
| `peer-required:` | `Token.PeerRequired(inner)` | `&:is(:where(.peer):required ~ *){…}` |
| `peer-disabled:` | `Token.PeerDisabled(inner)` | `&:is(:where(.peer):disabled ~ *){…}` |
| `peer-placeholder-shown:` | `Token.PeerPlaceholderShown(inner)` | `&:is(:where(.peer):placeholder-shown ~ *){…}` |

**Writing direction and descent**

| upstream | emilia | CSS |
| --- | --- | --- |
| `rtl:` | `Token.Rtl(inner)` | `[dir="rtl"] &{…}` |
| `ltr:` | `Token.Ltr(inner)` | `[dir="ltr"] &{…}` |
| `*:` | `Token.Children(inner)` | `:is(& > *){…}` |
| `**:` | `Token.Descendants(inner)` | `:is(& *){…}` |

**The priority wrapper — not a variant, a flag on every rule it wraps**

| upstream | emilia | CSS |
| --- | --- | --- |
| `…!` | `Token.Important(inner)` | `!important` on every declaration it wraps |

`.group` and `.peer` are classes the **consumer's markup** carries: emilia
emits the selector that reads them and never the class itself.

`Important` is the one row that is not a variant — it adds no selector and
no at-rule, it sets the `!important` flag on every rule it wraps, which is
upstream's per-utility `!` suffix.

`examples/emilia-modifiers/` is the worked example: a responsive navigation
bar, a peer-driven form field, a self-striping table, and one panel rendered
under all three dark-mode strategies.

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
