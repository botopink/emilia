# emilia · CHANGELOG

## Unreleased — v0.beta.22

- **The eleven display values, and the front 36 banner in both files**
  (1.0.10-beta front `36-emilia-layout`, step 1). `Layout` carried six of
  `§ 5.8`'s eleven display values; `InlineFlex`, `InlineGrid`, `Contents`,
  `FlowRoot` and `ListItem` are added as BARE SIBLINGS of the six, inside
  `Layout` itself and not under a sub-section, which is what keeps
  `.Layout.Flex` spelled the way every consumer spells it today. The six that
  predate this front emit byte-identical CSS, pinned by an assertion that lists
  them first. `// ── front 36 — layout ──` now fences the section in `tokens.bp`
  and `layoutTokenToCss` with its sub-dispatchers in `emilia.bp`; the front's
  tests live beside the dispatchers, not at the end of the file, for the reason
  front 35 records — two fronts appending to the same last line cannot be
  merged.

- **The dispatcher contract, and 776 leaves walked for a resolved length**
  (1.0.10-beta front `36-emilia-layout`, step 6). `layoutTokenToCss(t, th)` and
  every sub-dispatcher under the front 36 banner now carry `th: Theme`, each is
  `val out = case …; return out;` with arrow arms only, and each is exhaustive
  with no `_` — so a leaf added to `Layout` reds its own dispatcher rather than
  falling through. The shared `case`'s `Layout` arm is the one line this front
  changed there; no new top-level arm was added.
  The front's REGRESSION walks **all 776 `Layout` leaves** and asserts no
  declaration carries a `rem`, that every one carries a `:`, and that none
  carries a Tailwind class fragment (`inset-x-`, `top-`, `z-50`,
  `overflow-auto`, `float-start`, `box-border`). The same test asserts
  `.Border.Rounded.Lg` DOES carry a `rem`, so the probe is known to
  discriminate; a deliberate `0.25rem` planted in one inset arm was confirmed
  to fail it.

- **Multi-column, fragmentation and box-sizing — and the column widths are the
  theme's** (1.0.10-beta front `36-emilia-layout`, step 5). `Layout.Columns`,
  `Layout.Break { After, Before, Inside }`, `Layout.Box` and
  `Layout.BoxDecoration` close `§ 5.2`–`§ 5.6`.
  **The front's spec printed the fourteen named column widths as `rem`
  literals — `columns-md` → `columns:28rem` — and that is a defect, not a
  transcription.** Front 54's theme already carries `--container-md: 28rem` and
  front 35's `.Size.MaxW.Md` already reads it, so `columns:28rem` would be a
  SECOND spelling of one width: a project overriding `--container-md` would see
  its `max-width` move and its `columns` stay put. `.Layout.Columns.Md` emits
  `columns:var(--container-md)` through front 35's `containerVar`, and a test
  asserts the thirteen named widths carry no `rem` and that the `columns` value
  and the `max-width` value are the same string.
  The three counts are not lengths — `columns-2` is `columns:2` — and
  `break-inside` keeps its own shorter leaf set, `§ 5.5` having no `all`, no
  `page`, no `left` and no `right`.
  **Reference gap, recorded rather than hidden:** `columns-4` … `columns-12`
  resolve upstream through the bare-integer rule rather than through a theme
  key, so they are left undeclared until that is confirmed.

- **Float, clear, and an image that can be cropped** (1.0.10-beta front
  `36-emilia-layout`, step 4). `Layout.Float` and `Layout.Clear` answer
  `§ 5.9`/`§ 5.10`, `Layout.Object` the `Fit`/`Pos` split of `§ 5.12`/`§ 5.13`,
  and `Layout.Aspect` the three ratios of `§ 5.1`.
  Three name-versus-value traps, each with its own assertion rather than a
  shared one: **`float-start` is `float:inline-start`** (and `clear-start` is
  `clear:inline-start`) — the utility carries the logical name, CSS carries the
  logical VALUE, and `float:start` is not a value at all; the four two-word
  object positions are **one space, not a hyphen** — `object-left-bottom` is
  `object-position:left bottom`, and a test asserts `left-bottom` does not
  survive into the declaration; and `aspect-square` is **`1 / 1` with spaces
  around the slash**, the way `§ 5.1` prints it, with `16/9` asserted absent.

- **A scroll container, a stacking order and a box that keeps its space**
  (1.0.10-beta front `36-emilia-layout`, step 3). `§ 5.14`, `§ 5.15`, `§ 5.18`,
  `§ 5.19` and `§ 5.11` had no token: emilia could not clip, could not scroll,
  could not stack and could not hide an element while keeping its space.
  `Layout.Overflow` answers five values on the shorthand and five on each of
  `X`/`Y` — three properties, not one property with an axis flag — and
  `Layout.Overscroll` three on each of the same three. `Layout.Visibility`,
  `Layout.Z` and `Layout.Isolation` complete the step.
  **`invisible` is the UTILITY name and `hidden` is the CSS value**, and a
  transcription that carried the name through would emit
  `visibility:invisible`, which no browser honours; the test asserts the value
  AND that the word `invisible` does not survive into the declaration.
  `Z` is the one numeric family in this front that is not a length: `z-50` is
  `z-index:50`, a bare integer that never reaches `spacing` and that a test
  pins as carrying no `calc`, no `rem` and no `px`.

- **`position` and the whole inset family** (1.0.10-beta front
  `36-emilia-layout`, step 2). Nothing in emilia could be positioned, and no
  positioned box could be placed: `§ 5.16` and `§ 5.17` had no token at all.
  `Layout.Position` answers the five values and `Layout.Inset` the **nine
  directions front 35's `Pad` and `Margin` already carry** — `All`, `X`, `Y`,
  `T`, `R`, `B`, `L` and the logical pair `S`/`E` — over the **same scale**,
  answering the **same** `spacing(n)` / `spacingHalf(n)`, plus `Auto`, `Full`,
  `Frac { Half, Third, TwoThirds }` and a `Neg` sub-section per direction for
  `-top-4`. `.Layout.Inset.T.4` and `.Pad.T.4` therefore carry identical length
  text BY CONSTRUCTION rather than by two transcriptions that agree today; a
  test asserts the two values are equal rather than asserting each separately.
  `inset` is a real CSS shorthand so `All` is one declaration, while `X` and `Y`
  expand to `left:…;right:…` and `top:…;bottom:…` — `§ 5.17`'s own printing —
  through front 35's `axisDecl`, reused rather than re-spelled.
  `layoutTokenToCss` takes `th: Theme` from this step on, which is this front's
  one line of the shared `case` and nothing else of it.

- **`examples/emilia-card` stops pinning a compiler defect.** The commonJS
  `case`-over-a-uniquely-named-variant defect recorded further down this file
  is **fixed upstream**: such an arm now compares the `tag` string instead of
  testing `instanceof`, so a value built in a consumer package is matched by
  the library again. `.Text.Size.X3xl` and `.Text.Size.Base` reach the CSS, and
  the example's two class-body assertions — written to pin the pre-fix shape so
  the fix would show up as a change there — are rewritten to the real output:
  `.e_…{font-size:1.875rem;font-weight:bold;color:var(--color-red-600)}` and
  `.e_…{font-size:1rem;color:var(--color-gray-600)}`. The comment declaring
  them pinned is gone. Nothing in the library changed; `botopink test` in
  `examples/emilia-card` is 4/4 on its one declared target, commonJS. The
  example's 600 shades stay 600: front 33 moved them off 500 to dodge the
  leading-dot section resolver, and **that defect is still live** —
  `.Color.Red.500` and `.Color.Gray.500` still red with `type mismatch:
  expected Token, got __Token__Border`. See the BLOCKED note in `emilia.bp`.

- **`examples/emilia-spacing/`, and the front 35 block is fenced in both files**
  (1.0.10-beta front `35-emilia-spacing-sizing`, step 5). The
  `// ── front 35 — spacing and sizing ──` banner now fences the four sections
  in `tokens.bp` as well as the four dispatchers in `emilia.bp`. A line comment
  inside the `Token` enum body was recorded as a parser gotcha and is not one
  any more: measured green on both targets and on every example, and the note
  in `AGENTS.md` is corrected rather than worked around.
  The new workspace member `examples/emilia-spacing/` is the front's worked
  example — the multiplier scale, the nine directions, `auto`, negatives,
  a centred article shell as one class and one rule, a full-bleed header whose
  `-mx-6` cancels it, fractions, the per-axis viewport unit, `size-12`, the six
  logical forms, and a comment thread spaced by `Space.Y` with a reply pulled up
  by `-mt-2`. Its last two tests are the front's whole argument: no token in it
  emits a resolved length or a class fragment, and **halving `--spacing` gives
  the same class with the same declarations — only the `:root` block moves**.
  12 in-file tests, green on commonJS and erlang.
  `examples/emilia-cascade/` had four assertions pinning `padding:1rem` and
  `padding:2rem`; they now pin `calc(var(--spacing) * 4)` and
  `calc(var(--spacing) * 8)`, which is the correction this front owns.

- **`Space` — the first emilia token that declares on something other than the
  element** (1.0.10-beta front `35-emilia-spacing-sizing`, step 4).
  `space-x-*` / `space-y-*` set a margin on an element's CHILDREN, so
  `spaceTokenToSheet(t, th) -> Sheet` is the second of the two shapes contract
  4a defines and the one arm of `tokenToSheet` that does not go through
  `declSheet`. Its rule carries the nesting template
  **`& > :not(:last-child)`**, so `emilia([.Pad.All.4, .Space.Y.4])` renders as
  two rules of one class — `.e_x{padding:…}` and
  `.e_x > :not(:last-child){margin-block-end:…}` — and a modifier wraps the
  child selector rather than replacing it (`&:hover > :not(:last-child)`).
  The selector is `pub fn siblingSelector()`, written once: **front 40's
  `divide-*` must call it** rather than re-spell it. `X` is
  `margin-inline-end` and `Y` is `margin-block-end`, the logical pair, over the
  same scale and the same `Neg` as `Margin`; `XReverse`/`YReverse` set
  `--tw-space-x-reverse` / `--tw-space-y-reverse`.
  **Reference gap, recorded rather than hidden:** `space-*` is absent from the
  local `TAILWIND_CSS_DOCS.md` entirely, so both the child selector and the
  property are this front's proposal and not a transcription, and the two
  `--tw-space-*-reverse` names are upstream-internal and unverified. A test
  pins each, so changing one is a visible change.

- **`Size` — a component can be given a width** (1.0.10-beta front
  `35-emilia-spacing-sizing`, step 3). There was no sizing section at all: no
  width, height, min or max token of any kind. `Token.Size` carries thirteen
  sub-sections over `§ 8.1`–`§ 8.7` — `W`, `H`, `Both` (upstream's `size-*`, two
  declarations from one leaf), `MinW`, `MaxW`, `MinH`, `MaxH` and the six
  logical `Inline`/`Block` forms — 566 leaves of four kinds. A NUMBER is the
  spacing ladder again (`w-64` is `width:calc(var(--spacing) * 64)`); a `Frac`
  is a percentage carried by name, because `1/2` is neither an identifier nor a
  run of digits; a KEYWORD is an intrinsic size or a viewport unit, and **the
  viewport unit differs by axis** — `w-screen` is `100vw` where `h-screen` is
  `100vh`, asserted directly.
  **The named container widths are the theme's**: `.Size.MaxW.Md` is
  `max-width:var(--container-md)`, not `max-width:28rem`, built through a
  `containerVar` over front 54's `nsPrefix(Ns.Container)` and `themeVar` — the
  same shape as front 33's `paletteVar`. `MaxW.Screen.*` reads `--breakpoint-*`
  the same way. Front 54's theme already carried all thirteen container sizes
  and five breakpoints with exactly the lengths `§ 8.3` prints, so the front
  spells no `rem` of its own: a walk over all 566 leaves asserts the output
  carries none, and a second test reads every name back through `themeValue` so
  the reference AND the length are both pinned. `MaxW` carries all thirteen
  container names — `X3xs` and `X2xs` as well as the `Xs`..`X7xl` the spec
  lists — because the theme carries them and upstream has the utilities.

- **Nine directions for padding and margin, over a 35-leaf scale** (1.0.10-beta
  front `35-emilia-spacing-sizing`, step 2). `Pad` and `Margin` had three
  directions (`X`, `Y`, `All`) over five values and no `p-0` at all; they now
  carry `All`, `X`, `Y`, `T`, `R`, `B`, `L` and the logical pair `S`/`E`
  (`padding-inline-start` / `padding-inline-end`) over the thirty multipliers of
  upstream's default theme plus `Px` and `Half { 0, 1, 2, 3 }` — `0.5` cannot be
  an enum leaf, a numeric leaf being a run of digits, so `Half.1` reads "one and
  a half". `Auto` moved from `Margin.X` alone to every margin direction, and
  each of the nine gained a **`Neg` sub-section** — `-mt-4` is
  `.Margin.T.Neg.4`, a five-segment path, answering `spacing(-4)`, and it is the
  one Tailwind family with no alternative spelling. 936 leaves, every one of
  them a multiplier of `--spacing` and none of them a length: a test walks all
  936 through the public dispatcher and asserts the output carries no `rem`, no
  non-property and no class fragment.
  **`Neg.Half` is `{ 1, 2, 3 }`, not `{ 0, 1, 2, 3 }`** — `spacingHalf(-0)` is
  `spacingHalf(0)`, an `i32` having no negative zero, so `-0.5` is the one rung
  of front 54's ladder that cannot be spelled and this front declines to write a
  second `calc(var(--spacing) * …)` to reach it. Front 54 owes a signed half
  step; until then `-mt-0.5` has no token rather than a wrong one.

- **Spacing stopped emitting text that is not CSS** (1.0.10-beta front
  `35-emilia-spacing-sizing`, step 1). Three of emilia's ten sections emitted
  strings no browser can read, and all three were here. `padTokenToCss`
  answered `padding-x:` and `padding-y:` — **there is no `padding-x` property**,
  so `px-4` and `py-4` were silently discarded by every browser; `Pad.X` now
  answers the two real properties, `;`-joined inside the one token
  (`padding-left:… ;padding-right:…`), and `Margin.Y` likewise instead of
  `margin-y:`. `marginScaleX` was worse: it answered `m-0.25`, `m-0.5`, `m-1`,
  `m-2` and `margin-auto` — **Tailwind class fragments**, in a position where
  only a declaration is legal. `Margin.X.Auto` is `margin-left:auto;margin-right:auto`
  now, `Auto` having become a value on the ladder rather than a property.
  The six hand-written `rem` ladders (`padScaleX/Y/All`,
  `marginScaleX/Y/All`) are **deleted, not widened**: every leaf answers front
  54's `spacing(n)`, so a value is `calc(var(--spacing) * N)` and a project's
  `--spacing: 4px` override finally reaches the utilities that consume it.
  This changes the CSS four existing tokens emit and the front owns the break —
  no path that compiled stops compiling, and what those paths emitted was never
  valid CSS. Eight tests pin it, including one walk over every `Pad`/`Margin`
  leaf asserting that `padding-x`, `padding-y`, `margin-x`, `margin-y`,
  `m-0.25`, `m-0.5`, `m-1`, `m-2`, `margin-auto` and the substring `rem` appear
  nowhere in the output.

- **The colour palette resolves through the theme** (1.0.10-beta front
  `33-emilia-color-palette`, step 1). `Token.Color` is the 26-family x
  11-shade grid of `§ 3.6` / `§ 21.1` — the seventeen chromatic families
  `Red..Rose` and the nine neutral `Slate..Taupe`, each `50 100 … 900 950` —
  plus `White`, `Black`, `Transparent`, `Current`, `Inherit` and the retained
  `Hex(string)` escape. **The emitted CSS changed and this front owns the
  break**: `colorTokenToCss` answered `"color:red"` for every shade of Red, so
  `.Color.Red.100` and `.Color.Red.900` rendered byte-identically and the shade
  level existed in the type and did nothing. It now answers
  `color:var(--color-red-500)`, byte-equal with upstream's own `.text-red-500`
  rule, built by the new `pub fn paletteVar(family, shade)` over front 54's
  `themeVar` and front 54's `nsPrefix(Ns.Color)` — **no literal ladder, and the
  `--color-` prefix still has exactly one author in the library**.
  `redPaletteHex`, a correct nine-shade ladder of v3 hex values that nothing
  called, is deleted. `.Color.White` / `.Color.Black` follow the theme too
  (`var(--color-white)` / `var(--color-black)`), which rewrites five front-56
  assertions and the two examples that pinned the hex.

- **The palette is a string-to-string mapping, with the enum at its edge**
  (front `33-emilia-color-palette`, step 5). `paletteVar(family, shade)` and
  `alphaWrap(percent, css)` are `pub` and take plain strings; the property name
  (`color:`, `background-color:`) lives in the dispatcher rather than in the
  value; no function in the front takes a `Token.Color` where a `string` would
  do. So front 57's escape hatch can hand a colour this enum does not carry to
  the same formatter and get a well-formed declaration back, without a second
  palette table and without this front changing shape. `Color.Hex(value)` is
  retained unchanged as the proof that a string payload splices — and recorded
  as **unconstructible**, with the exact compiler errors measured against
  `zig-out/bin/botopink`: `Token.Color.Hex("#abc")` reds `'Hex' is not declared
  in any behavior implemented for 'Token'`, and `val h: Token =
  .Color.Hex("#abc");` reds `unbound variable 'Color'`. A payload-carrying
  token has to be a top-level variant with builtin-typed fields.

- **`Alpha(percent, inner)` — upstream's `/N` opacity suffix** (front
  `33-emilia-color-palette`, step 4). `bg-red-500/50` is the most used colour
  form in real markup and v0 had no token for it. `Token.Alpha(percent: 50,
  inner: [.Bg.Color.Red.500])` emits `background-color:color-mix(in oklab,
  var(--color-red-500) 50%, transparent)`. A **top-level** variant with a
  builtin-typed field, because opacity cannot be a leaf under the family (the
  shade already is one) and a payload leaf nested in a section has no
  constructible spelling. **It rewrites a `Sheet`, not a string** — the spec's
  `alphaWrap(percent, tokensToCss(inner))` predates front 56, and folding the
  inner tokens back to one string would flatten away the selector and at-rule
  of any modifier inside, so `Hover([Alpha(…)])` and `Alpha([Hover(…)])` both
  work and both are pinned. A non-colour token is rewritten too and the result
  is meaningless CSS, deliberately: a silently dropped declaration hides the
  mistake, one a browser discards does not. `alphaWrap(percent, css)` is `pub`
  and takes plain strings, for front 57. No `String.slice` anywhere — `split` +
  `at` + the ARRAY `slice` do the same job without installing the prelude whose
  `charCodeAt` patch is self-recursive.

- **`paletteEntries()` — the 286 numeric values, as data** (front
  `33-emilia-color-palette`, step 3). The other half of the mechanism: the
  utility emits `var(--color-red-500)` and this declares it. A plain
  `ThemeEntry[]`, not a rendered block, so a consumer composes it —
  `extendTheme(defaultTheme(), paletteEntries())` — and front 54's `themeCss`
  renders it into the `theme` layer. Keeping it a list is what lets a project
  that already imports upstream's own theme leave it out and still use every
  colour token; `emilia(...)` emits no `@theme` block of its own at any size of
  token list. Transcribed from upstream `tailwindcss` **4.3.2**'s `theme.css`;
  `--color-white` / `--color-black` stay front 54's and are not duplicated.
  **The two anchor values are in upstream's spelling, not the reference's**:
  `§ 3.6` prints `oklch(0.637 0.237 25.331)` for red-500 and upstream writes
  `oklch(63.7% 0.237 25.331)` — the same lightness, two spellings — and
  byte-parity with the emitted block is what is worth having.

- **`Bg.Color` — the palette on `background-color`** (front
  `33-emilia-color-palette`, step 2). The same 26 x 11 grid and the same five
  named colours under a sub-section of `Bg`, and **all 286 cells resolve**:
  `Bg` is a head segment no other enum carries, so the four-segment path
  `.Bg.Color.Red.500` has no hash-order tie to lose and is the way to reach the
  six shades `.Color.Red` / `.Color.Gray` cannot. The property is
  `background-color`, the longhand upstream sets (`§ 10.3`) — the `background`
  shorthand the v0 stub emitted resets every other background property of the
  element as a side effect. `bgTokenToCss` is front 39's and gains exactly two
  things from this front: the `Color` arm and the `th: Theme` parameter contract
  4a asks of every sub-dispatcher. Its four legacy leaves (`.Bg.White`,
  `.Bg.Black`, `.Bg.Red.500`, `.Bg.Gray.500`) keep the shorthand and the output
  they had, so nothing that compiled changed meaning.

- **Six colour cells are declared and unreachable** (front
  `33-emilia-color-palette`). `.Color.Red.{100,500,700}` and
  `.Color.Gray.{100,500,700}` do not compile in any spelling. The compiler's
  leading-dot section resolver scans every registered enum — the synthesised
  section enums included — and returns the first whose tree carries the path,
  never consulting the expected type; `Token.Border.Color` (front 40's stub)
  carries the same six leaves, so hash order decides and today it decides
  against `Token`. It reds at the call site rather than emitting the wrong CSS.
  `examples/emilia-card/` moved from `.Color.Red.__500` / `.Color.Gray.__500`
  to the 600 shade for that reason, with the cause written above the tokens.
  Reported to botopink-lang; `.Bg.Color.<Family>.<shade>` reaches every shade
  unaffected, because `Bg` is a head segment no other enum carries.

- **`examples/emilia-cascade/`** (1.0.10-beta front
  `56-emilia-cascade-and-output`, Examples). A new workspace member, the
  front's worked example: one card whose styles reach outside its own class.
  The hover is a **sibling rule** (`@media (hover: hover){.e_x:hover{…}}`) and
  not a block nested in the class body; the breakpoint is a hoisted
  `@media (width >= 48rem)` whose width is read from the theme's
  `--breakpoint-md`, so overriding the variable moves the breakpoint; the reset
  arrives through `Options` and lands in its own `base` layer behind a
  **literal** selector the class name and the prefix never reach; and the
  document is layered, which is what lets a project's own CSS beat a utility
  deliberately rather than by accident of order. Ten in-file tests, green on
  commonJS and on erlang, also covering `withPrefix`, `withLayers(o, false)`,
  `withImportant(o, true)` and the two-flush contract.

- **The host cell is a dumb string store** (1.0.10-beta front
  `56-emilia-cascade-and-output`, step 5). `flushSheet()` is **gone** and
  `drainRules()` takes its place: it hands back the registered entries as
  `name + "\t" + payload` records joined by `"\n"`, in insertion order, and
  clears the cell — the same per-render contract. `flushSheet` assembled the
  `<style>` document **inside** the two `#[@External]` templates, once in
  JavaScript and once in Erlang, which is where the byte-level divergence risk
  lived and where nothing written in botopink could reach it. Keeping it as
  the spec's step 5 asks would have left exactly the duplicate assembly the
  front's own definition of done forbids, and nothing outside `emilia.bp`
  could name it. Five tests now pin the two templates against each other on
  both targets.

- **`emiliaWith` / `flushWith`, and a document with layers in it** (front
  `56-emilia-cascade-and-output`, step 7). `emiliaWith(tokens, th)` and
  `flushWith(o)` are the new entry points; `emilia(tokens)` is
  `emiliaWith(tokens, defaultTheme())` and `flush()` is
  `flushWith(defaultOptions())`, both with the signatures they had, so no
  consumer changes. What changed is **the emitted CSS**, and this front owns
  the break: `[.Bg.Black, Hover([.Bg.White])]` went from
  `<style>.e_x{background:#000000;:hover{background:#ffffff}}</style>` to a
  layered document whose modifier is a **sibling rule**, hoisted out with the
  selector and at-rule the variant reference gives it. The six modifier tests
  in `emilia.bp` and the flush test in `examples/emilia-card/` are rewritten to
  the hoisted shape and each names the row it comes from.

- **The six V1 modifiers, against the variant reference** (front
  `56-emilia-cascade-and-output`, step 7). `hoverVariant()` is
  `@media (hover: hover) { &:hover }` and not a bare `:hover`;
  `focusVariant()`/`activeVariant()` are selector-only; `mdVariant(th)`,
  `lgVariant(th)` and `xlVariant(th)` are `@media (width >= 48rem | 64rem |
  80rem)` and not `@media(min-width:768px)`. A breakpoint **reads its width
  from the theme**, so a project that overrides `--breakpoint-md` gets its own
  media query — and so the theme is genuinely an input to the class hash,
  which is contract 4's clause 1 amended. Front 34 moves the six into its own
  variant table and adds the rest.

- **The conflict rule, written down and tested** (front
  `56-emilia-cascade-and-output`, step 8). The last rule in the stylesheet
  wins: inside one `emilia()` call that is the token list order, across calls
  it is registration order, and before this front nothing said either —
  `flushSheet` iterated a `Map` on commonJS and a `lists:keystore` list on
  erlang and no test compared them. `[.Layout.Grid, .Layout.Flex]` renders
  `display:grid;display:flex`, reversing the list reverses the winner and is a
  different class, two calls render in call order, and reordering an unrelated
  token moves no other rule.

- **The drain stream layers two separators on one character, and that is
  resolved rather than avoided** (front `56-emilia-cascade-and-output`, step
  5). The cell joins its entries with `"\n"` and each payload joins its own
  records with `"\n"`; the front's spec writes both without noticing. It stays
  unambiguous because a payload record is **tagged**: a line whose first field
  is `R` or `B` continues the entry above it, and any other line opens a new
  one. A registered name is `e_<hex>`, so it is never `R` or `B`.

- **`String.prototype.charCodeAt`'s commonJS prelude patch is self-recursive**
  (found by front `56-emilia-cascade-and-output`). The backend installs the
  whole `String` behavior prelude into any module using a member that needs a
  patch — `slice` is one — and that prelude writes
  `String.prototype.charCodeAt = function(index) { return ((this.valueOf().charCodeAt(index) ?? -1) | 0); }`,
  which calls the patch it has just installed. One `s.slice(…)` anywhere in
  `output.bp` therefore made `hashHex`'s host template blow the stack for every
  non-empty class body, on commonJS only. `output.bp` splits on the separator
  instead of slicing. Reported to botopink-lang.

- **A `case` arm over a uniquely-named variant lowers to `instanceof`, and
  `instanceof` does not cross a package boundary** (found by front
  `56-emilia-cascade-and-output`, pre-existing). The commonJS backend lowers an
  arm whose variant name is unique in the program to
  `if (_s instanceof __Token__Text__Size$X3xl)` and an arm whose name repeats
  to `if (_s.tag === "Lg")`. A consumer package re-emits its own copy of the
  enum classes, so a value built in `examples/emilia-card/` is never
  `instanceof` the class `emilia` matches against: the whole `case` falls
  through and answers `undefined`. `.Text.Size.X3xl` and `.Text.Size.Base`
  have gone missing from that example's two class bodies since before this
  front — the pre-56 document read `.e_x{;font-weight:bold;color:red}`, and the
  empty leading declaration is the same token going nowhere — while
  `.Text.Size.Lg` survives only because `Lg` repeats elsewhere in `Token`. The
  example's assertions now pin what is actually emitted, so the fix shows up
  as a change there. Reported to botopink-lang.

- **The rule model** (1.0.10-beta front `56-emilia-cascade-and-output`, steps 1–3). New
  module `modules/emilia/src/output.bp`, declared `pub mod output;` in `root.bp` and
  listed in the member manifest's `files`. It declares **no external** and touches no
  host cell. `Rule(layer, atRules, selector, declarations, important)` replaces the
  declaration string a token used to lower to; `Block(header, body)` carries a rule that
  is not a style rule (today only `@keyframes`); `Sheet(rules, blocks)` is what a token
  list produces; `Variant(atRule, selector)` is what a modifier is. A `selector` is a
  **nesting template** carrying exactly one `&`, so every row of the variant reference
  is one template — `&:hover`, `[dir="rtl"] &`, `:is(& > *)`,
  `&:is(:where(.group):hover *)` — and a selector with **no** `&` is a literal selector,
  which is how the theme writes `:root` and front 55 writes its reset. Constructors:
  `emptySheet()`, `declSheet(decls)` (empty string → empty sheet), `staticSheet(layer,
  selector, decls)`, `blockSheet(header, body)`, `mergeSheet(a, b)`, `declarationsOf(s)`,
  `layerNames()`. `nestVariant(s, v)` wraps a sheet in a variant and `markImportant(s)`
  sets the flag on every rule; both return new values, records being immutable.
  `nestVariant` **refuses** a variant selector that does not carry exactly one `&`,
  naming the selector: with none the variant replaces the rule it was meant to wrap,
  with two it duplicates it. There is no argument that relaxes it.

- **CSS nesting runs inner-`&`-first, and the front's own spec had it backwards**
  (front `56-emilia-cascade-and-output`, step 2). `&:hover { &::before { … } }`
  flattens to `&:hover::before`: the **inner** rule's `&` is what the **outer**
  variant's selector replaces. The spec's Mechanism says the opposite ("the new
  selector is `v.selector` with its single `&` replaced by the rule's current
  selector"), which renders every nested pair backwards and contradicts the spec's
  own acceptance list. `nestRule` implements the acceptance list, and the two nesting
  tests pin both orders so it cannot drift back.

- **The codec** (front `56-emilia-cascade-and-output`, step 4). A host cell stores one
  string per class, so `encodeSheet(s)`/`decodeSheet(raw)` carry a `Sheet` through it:
  records joined by `"\n"` and tagged `R`/`B`, fields by `"\t"`, the `atRules` list by
  `"\r"`. `encodeSheet(emptySheet())` is `""` and `decodeSheet("")` is `emptySheet()`.
  That no rendered declaration carries one of the three is the assumption the codec
  rests on, so it is **checked** rather than assumed: `carriesSeparator(s)` is a value,
  and a test walks the dispatcher output and the theme's own strings through it.

- **Options and the render** (front `56-emilia-cascade-and-output`, step 6).
  `Options(theme, base, prefix, important, layers)` with `defaultOptions()` and the five
  `withTheme`/`withBase`/`withPrefix`/`withImportant`/`withLayers` updaters.
  `renderRule(className, r, o)` substitutes `"." + prefix + className` for the rule's
  `&`, wraps the declarations in the rule's at-rules **outermost-first**, and appends
  `!important` **per declaration** when either the rule or the options say so; a literal
  selector renders literally and the prefix never reaches it. `renderDocument(raw, o)`
  emits `@layer theme, base, components, utilities;` first when `layers == true` and **no
  `@layer` token at all** when it is false, then each non-empty layer in that order — the
  theme as a `:root` rule, `o.base` (front 55's reset, **opt-in** here rather than
  opt-out), the components layer, then the registered classes in registration order with,
  inside one class, the unconditioned rules before the conditioned ones — and finally the
  `@keyframes` blocks, outside every layer and deduplicated by header. Measured: 42/42
  (`output.bp`) + 37/37 (`theme.bp`) + 6/6 (`spacing.bp`) + 17/17 (`emilia.bp`) on
  commonJS and on erlang.

- **`Array.reverse()` mutates its receiver on commonJS and does not on erlang** (found
  by front `56-emilia-cascade-and-output`). `val rev = xs.reverse();` leaves `xs`
  reversed on commonJS and untouched on erlang, so any fold over a reversed list is a
  silent target divergence. `wrapAtRules` therefore maps the list twice — the opening
  braces in order, the closing braces after the body — instead of folding a reversed
  copy. Reported to botopink-lang as a codegen defect alongside the two front 54 found.

- **The theme** (1.0.10-beta front `54-emilia-theme`, step 1). New module
  `modules/emilia/src/theme.bp`, declared `pub mod theme;` in `root.bp` and listed in
  the member manifest's `files`. A theme is a **flat `ThemeEntry[]`**, not nineteen
  record fields: `ThemeEntry(name, value)`, `Theme(entries, keyframes, darkMode)`,
  `DarkMode { Media, Class(name), Attribute(name, value) }`, and the `Ns` enum naming
  the nineteen namespaces. `nsPrefix(ns)` is the **only** place a custom-property
  prefix string is written — `--color-`, `--font-`, …, `@keyframes ` — and
  `Ns.Spacing` maps to `--spacing` with no trailing dash, because it is a single
  variable rather than a family. `allNamespaces()` lists the nineteen in declaration
  order. Measured: `botopink test` in `modules/emilia/` is 5/5 (`theme.bp`) + 17/17
  (`emilia.bp`) on commonJS and on erlang.

- **`defaultTheme()`** (front `54-emilia-theme`, step 2). The stock theme, in declaration
  order: `--spacing`, the five `--breakpoint-*` (rem, not px), the eight `--radius-*`,
  `--color-black`/`--color-white`, the thirteen `--text-*` sizes each paired with its
  `--text-*--line-height` (26 entries), the seven `--shadow-*` (`sm`–`xl` two-shadow), the
  thirteen `--container-*`, the four `--animate-*`, and the four `keyframes` bodies (`spin`,
  `ping`, `pulse`, `bounce`) in their own list, because a keyframes value is a rule body and
  cannot live inside `:root`. Entry ORDER is a contract — emilia's class names are content
  hashes, so a reordered theme is a different document. The palette is **front 33's**: only
  black and white ship here, and front 33 hands over `paletteEntries() -> ThemeEntry[]` for
  `extend(defaultTheme(), paletteEntries())`. Measured: 14/14 + 17/17 on both targets.

- **extend · override · clear · empty · read** (front `54-emilia-theme`, step 3).
  `extendTheme(th, entries)` adds entries and **overrides in place** — a name already
  present keeps its position, because the entry order is a content-hash contract;
  `clearNamespace(th, ns)` is the `--color-*: initial` reset, `emptyTheme()` the
  `--*: initial` one, `namespace(th, ns)` reads one namespace back, `themeValue(th,
  name)` resolves a name to its literal value (`""` when absent) and `themeVar(name)`
  gives the `var(--name)` reference form every utility emits. `extendTheme` **refuses**
  an entry whose name matches no known prefix, naming it; there is no permissive mode
  and no argument that relaxes it. Two spellings the language forced: the composer is
  `extendTheme`, because `extend` is a keyword (`Name extend Type { … }`), and a lambda
  body that is a bare `if` expression is bound to a `val` first, because the bare form
  lowers to `undefined` on commonJS. Measured: 24/24 + 17/17 on both targets.

- **`spacing(n)`** (front `54-emilia-theme`, step 4). New module
  `modules/emilia/src/spacing.bp`. `spacing(4)` answers `calc(var(--spacing) * 4)`,
  `spacing(0)` answers `0`, `spacing(-4)` answers `calc(var(--spacing) * -4)`, and
  `spacingHalf(n)` covers the closed set of fractional steps. **emilia never resolves a
  spacing value**: emitting the `rem` literal would be byte-different from upstream for
  every spacing utility and would make a consumer's `--spacing: 4px` override silently
  ineffective. `i32` rather than `f64`, so the emitted string never depends on a
  backend's float formatting. Measured: 6/6 + 24/24 + 17/17 on both targets.

- **The static custom-property block and the dark-mode strategy** (front
  `54-emilia-theme`, steps 5 and 6). `themeCss(th)` renders the theme as the BODY of a
  `:root` rule — `name:value` joined with `;`, in the theme's own order, which is
  `defaultTheme()`'s declaration order followed by each `extendTheme`'s. The block is
  **always the whole theme** (`@theme static` semantics): tree-shaking needs a
  whole-program pass over every `emilia()` call site and emilia hashes per call site, so
  the tree-shaken form is out of scope for this milestone — not a bug to file later.
  `keyframeCss(th)` hands back the four `@keyframes` bodies, which cannot live inside
  `:root`; each carries the bare animation name and a brace-balanced body, so front 56
  writes `nsPrefix(Ns.Keyframes) + name + value` and never spells the at-rule.
  `darkAtRule(th)` / `darkSelector(th)` turn the strategy into the two pieces front 34
  needs: `@media (prefers-color-scheme: dark)` + `&` for `Media`, `""` +
  `&:where(.dark, .dark *)` for `Class`, `""` +
  `&:where([data-theme=dark], [data-theme=dark] *)` for `Attribute`. `withDarkMode(th,
  mode)` swaps the strategy and changes nothing else. Verified: `themeCss(defaultTheme())`
  and the four keyframes blocks are **byte-identical to the 70 expected lines** of the
  milestone's own `05-emilia/test-snap.md`. Measured: 6/6 + 36/36 + 17/17 on both targets.

- **A theme is a module** (front `54-emilia-theme`, step 7). Upstream shares a theme
  between projects by importing a CSS file; in botopink a theme is a function in a module,
  so sharing it is an ordinary package dependency. New workspace member
  `examples/emilia-theme/` (application, `emilia` via `{ "workspace": true }` and nothing
  else) defines a brand theme in one function, clears the stock colour namespace, composes
  a second package's entries over the default, reads values back through `themeValue`, and
  styles a card with `spacing(4)`. It is the proof that the front 54 surface crosses a
  package boundary through `from "emilia"`. 6/6 on commonJS and on erlang; it builds and
  runs. The flushed document does not yet carry the theme — that is front 56's
  `withTheme`/`flushWith`. `theme.bp` carries the same composition assertion inline, so
  the front's own suite covers it without the example. Measured: 6/6 (spacing.bp) + 37/37
  (theme.bp) + 17/17 (emilia.bp) in `modules/emilia/`, 6/6 in `examples/emilia-theme/`,
  both targets.

- **The repository is a workspace** (`02-packaging` step 2; decisions 75 and 76 of
  1.0.10-beta). `botopink.json` at the root is `{ name, version, description, targets
  [commonJS, erlang], workspaces ["modules/*", "examples/*"] }` — no `src`, `files`,
  `target` or `dependencies`; `botopink build/test` there is the located refusal naming
  the members (`emilia, emilia-card`). The core moved with `git mv` to
  `modules/emilia/` (the three `src/*.bp`; emilia has no `test/` — every test is inline)
  and its manifest carries `name emilia`, `entry root.bp`, `target commonJS`,
  `targets [commonJS, erlang]` and `files: [root.bp, tokens.bp, emilia.bp]` — a library
  member without `files` is `✗ ships nothing`. `examples/emilia-card` is the member
  `emilia-card`, depending on `emilia` via `{ "workspace": true }` instead of the git
  form; `jhonstart` keeps `{ git, branch }` until jhonstart is a workspace too and a
  `path` to `…/modules/jhonstart` exists. The pre-commit runner is workspace-aware:
  `botopink test` in every `modules/*/` member, then the examples gate. Measured: the
  core 17/17 on commonJS and on erlang at its new path; `botopink-lib-test` prints one
  row per member and no umbrella row. `examples/emilia-card` joins
  `scripts/known-broken-examples.txt` — jhonstart `feat` (`13d1672`) does not compile
  against botopink-lang `feat` (`hooks.bp:109 use-without-context-effect`), which is
  jhonstart's `fix/context` front, not emilia's; the example's own 4 tests still pass.

- **Section types by path** (botopink-lang front 06 N28/C8): the 27
  sub-dispatcher annotations name their section by path — `TokenText` is
  `Token.Text`, `TokenTextSize` is `Token.Text.Size`, … The flat names were
  never declared; the checker accepted them until pattern bindings became
  typed. Tests 17/17 on commonJS and erlang; `emilia-card` builds and its 4
  tests pass (it still fails at run time on the known commonJS sibling
  `require("../module")`, as before).

- **1.0.3 surface** (botopink-lang front 12): `pub enum Token` is
  `pub type Token { … }` (sections and payload variants unchanged). Tests 17/17 on
  commonJS and erlang; `emilia-card` builds and its 4 tests pass (it still fails at
  run time on the known commonJS sibling `require("../module")`, as before).
  `botopink format` is not applied: it reorders the variants and sections.

- The erlang target runs: `register`, `flushSheet` and `hashHex` carry an
  `@External.Erlang` form (process-dictionary sheet, the same djb2 hash), so
  `botopink test --target erlang` passes 17/17 instead of stopping at
  `MissingExternalTarget`; `botopink.json` lists `erlang` in `targets`.

- The examples gate no longer aborts silently on a `scripts/known-broken-examples.txt`
  holding only comments or blank lines: the runner reads the list with `awk`, whose
  "no entry" is not a failure under `set -euo pipefail`.

- `examples/emilia-card` builds again: its jhonstart builder calls pass `attrs`
  explicitly (parameter defaults are not applied yet), it leaves
  `scripts/known-broken-examples.txt`, and CI checks jhonstart out beside
  emilia so the examples gate can resolve it.

- **MIT license.** `LICENSE` (`Copyright (c) 2026 Eric Fillipe and botopink
  contributors`) backs the README's License section, which now points at it.

- The gate builds the examples: after `botopink test`, the pre-commit hook
  and CI run `botopink build` in every `examples/*/` with a `botopink.json`;
  `scripts/known-broken-examples.txt` lists the ones allowed to fail, and a
  listed example that builds fails the gate.
- **A gate.** `scripts/git-hooks/pre-commit` (conflict markers, then
  `botopink test`; install with `git config core.hooksPath
  scripts/git-hooks`) and `.github/workflows/test.yml` (`test-libs --lib
  emilia --target commonJS`), matching the sibling libraries. Before this,
  a commit ran nothing locally or in CI.
- **ecosystem-and-snap-tail F2** — `tokenToCss` exhaustive dispatch +
  modifier composition re-pinned for the V1 nested-section enum (the
  V0 surface dropped during the `3f77623` WIP migration). Every top-
  level section (`Text/Font/Color/Bg/Pad/Margin/Layout/Flex/Border/
  Effect`) routes to its typed sub-dispatcher; every modifier
  (`Hover/Focus/Active/Md/Lg/Xl`) recurses through the new
  `tokensToCss` helper to compose the inner CSS inside the
  pseudo/media wrapper. 11 in-file smoke tests pin the V1 leaf set +
  modifier composition. The full `emilia(tokens)/flush()` public
  surface remains deferred (the `register`/`flushSheet`/`hashHex`
  declares + `pub fn emilia/flush` need re-authoring under a follow-
  up commit pair, blocked on the V0→V1 example migration).

  GOTCHA pinned: in `case ARM(field) -> ...`, the bind name maps to
  the variant's **literal field name**. Section-auto-synth variants
  carry `_inner` (use `_inner`); user-declared payloads use the
  declared name (`Hover(inner: Token[])` → `Hover(inner)`, not
  `Hover(_inner)`). Mixing the two issues `undefined.map` at runtime.

## Unreleased — v0.beta.20

- **F0** lib stand-up (v0 surface): `Token` enum with 3 sections
  (Text/Color/Bg/Pad — V0 flat variants pending the `enum-sections`
  language extension) + 6 modifier variants (`Hover`, `Focus`,
  `Active`, `Md`, `Lg`, `Xl`), `emilia(tokens: Token[]) -> string`
  entry point, `flush() -> string` per-render serializer, the
  `Stylesheet` host cell (commonJS only at v1 — folded into
  `emilia.bp` because cross-module `#[@External.<targert>(...)]` symbol imports don't
  lower at v0). Two-module package (`tokens` + `emilia`); `botopink
  test` green over 9 in-file blocks.
- **F2** `tokenToCss` exhaustive `case` covering every v0 variant.
- **F3** modifier composition — `Hover`/`Focus`/`Active` produce
  `:pseudo{…}` blocks; `Md`/`Lg`/`Xl` produce
  `@media(min-width:Xpx){…}` blocks; modifiers nest
  (`Md(Hover(...))`).
- **F4** `flush()` per-render semantics — register collects, flush
  serialises + clears, two consecutive flushes emit two independent
  blocks.
- **F5** runnable [`examples/emilia-card/`](examples/emilia-card/) — a
  small jhonstart page composed with three emilia class names + a
  modifier in the body style; depends on `jhonstart` + `emilia`; 4
  in-file `test {}` green.
- Docs sweep: README, AGENTS, docs, this CHANGELOG.

### Deferred (v0.beta.21+)

- F0 second half — two generic jhonstart hooks
  (annotation-on-builder, `[name]={expr}` html attribute). Block on
  the jhonstart `Element` attribute slot + a call-site decorator
  mechanism in the compiler.
- F1 full `Token` shape — `enum-sections` language extension lands the
  nested section paths (`.Color.Red.500`, `.Pad.X.4`).
- `stylesheet.bp` split — cross-module external import parity.
- erlang/beam `Stylesheet` port.

## v0.0.1 — Seed

- Repository created under `botopink/emilia`. Tracked from
  `botopink/projects` (workspace root) as a git submodule on the
  `feat` branch. Intent + design surface lived in
  `tasks/v0.beta.19/specs/emilia.md`, later updated to
  `tasks/v0.beta.20/specs/ecosystem.md` (v20 ecosystem-expansion
  keystone).
