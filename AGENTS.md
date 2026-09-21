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
   each variant to a `Sheet`, encodes the sheet, hashes the encoding
   into a stable class name, registers the `(name, payload)` pair on the
   per-render `Stylesheet`, returns the class name. Front 56 added
   **`emiliaWith(tokens, th)`**, the same under an explicit theme;
   `emilia(tokens)` is `emiliaWith(tokens, defaultTheme())`.
2. **`flush() -> string`** — drains the `Stylesheet` and renders the
   `<style>...</style>` **document** — the `@layer` statement, the theme
   layer, the base layer, the components layer, the utilities layer and
   the `@keyframes` blocks — AND clears the cell. Per-render contract —
   two consecutive calls emit two independent documents; the second has
   no `@layer utilities` body if no `register` happened in between.
   Front 56 added **`flushWith(o: Options)`**; `flush()` is
   `flushWith(defaultOptions())`.
3. **`Token` enum** — the typed authored surface (see `tokens.bp`).
   Sections: Text, Font, Color, Bg, Pad, Margin, Layout, Flex, Border,
   Effect + the 83 modifier variants of front 34, each carrying a nested
   `Token[]` (see below and `docs.md` § Modifiers). Front 33 widened **`Color`** to the
   26-family x 11-shade grid (17 chromatic Red..Rose + 9 neutral
   Slate..Taupe, each `50 100 … 900 950`) plus
   `White`/`Black`/`Transparent`/`Current`/`Inherit` and the `Hex(string)`
   escape, and added **`Bg.Color`** — the same grid on `background-color`,
   plus the five named colours. `Bg`'s legacy Red/Blue/Gray/White/Black/Hex
   leaves keep the `background` shorthand and their pre-33 output beside it.
   The top-level **`Alpha(percent: i32, inner: Token[])`** wrapper is front
   33's too — upstream's `/N` opacity suffix, rewriting each inner
   declaration's value into `color-mix(in oklab, <value> N%, transparent)`.

Front 54 adds the **theme** and the **spacing ladder** (`theme.bp`,
`spacing.bp`): `ThemeEntry`, `Theme`, `DarkMode`, `Ns`, `nsPrefix`,
`allNamespaces`, `defaultTheme`, `emptyTheme`, `extendTheme`, `clearNamespace`,
`namespace`, `themeValue`, `themeVar`, `themeCss`, `keyframeCss`, `darkAtRule`,
`darkSelector`, `withDarkMode`, `spacing`, `spacingHalf` — see `docs.md`
§ The theme. Rewiring `tokenToCss` to `spacing()` and to the theme is fronts
33–47's work. Front 35 took **six** of the seven drifted `rem` ladders front 54
found in `emilia.bp` — `padScaleX/Y/All` and `marginScaleX/Y/All`; the seventh,
`flexGapScale`, is `Flex.Gap`'s and belongs to front 37, which owns that
section.

Front 56 adds the **rule model, the codec and the renderer** (`output.bp`) —
see `docs.md` § The cascade and the output. It is the interface fronts 33, 34
and 35 consume, so it is stated here in one line each:

| Name | What 33/34/35 see |
| --- | --- |
| `Rule(layer, atRules, selector, declarations, important)` | one style rule; `atRules` outermost-first, `declarations` `;`-joined with no braces |
| `Block(header, body)` | a rule that is not a style rule — `@keyframes spin` + its brace-balanced body |
| `Sheet(rules, blocks)` | what a token list produces, and what a `…TokenToSheet` arm returns |
| `Variant(atRule, selector)` | what a modifier is — **front 34 writes one `Variant`-returning fn per variant name and nothing else about emission** |
| `selector` | a nesting template with **exactly one `&`**; no `&` means a literal selector; two or more is refused, with no argument that permits it |
| `emptySheet()` / `declSheet(decls)` | the one-line adapter for a section dispatcher; `declSheet("")` is `emptySheet()` |
| `staticSheet(layer, selector, decls)` | a rule whose selector is literal — 54's `:root`, 55's reset |
| `blockSheet(header, body)` | a `@keyframes` sheet — front 44's `animate-*` |
| `mergeSheet(a, b)` / `declarationsOf(s)` / `layerNames()` | concatenate rules then blocks; read every declaration back; the cascade order, written once |
| `nestVariant(s, v)` / `markImportant(s)` | wrap a sheet in a variant; set `important` on every rule — front 34's `Important(inner)` is one line on top |
| `encodeSheet(s)` / `decodeSheet(raw)` / `carriesSeparator(s)` | the codec that carries a `Sheet` through a string-keyed host cell, and the check its assumption rests on |
| `Options(theme, base, prefix, important, layers)` + `defaultOptions()` + the five `with…` | the build-level knobs; `withBase` is how front 55 turns preflight on |
| `renderRule(className, r, o)` / `renderDocument(raw, o)` | one rule; the whole `<style>` document |

**Fronts 33–47 never see `Rule`, `Sheet` or `Variant`.** A section dispatcher
keeps returning a declaration string and the shared `case` in `emilia.bp`
adapts it — `Text(_inner) -> declSheet(textTokenToCss(_inner))`. A front whose
tokens genuinely need a selector (`space-x-*`, `divide-*`) writes a
`…TokenToSheet(t, th) -> Sheet` instead and says so in its own README. Two
shapes, one line each in the shared `case`.

**The `Theme` reaches the shared dispatcher, not the sub-dispatchers.**
`tokenToSheet(t, th)` carries it and every modifier arm already uses it
(`Md(inner) -> nestVariant(tokensToSheet(inner, th), mdVariant(th))`). The ten
section sub-dispatchers still have their front-54 signature
(`fn textTokenToCss(t: Token.Text) -> string`): front 56 owns the shared
dispatcher and the public entry points, **not** the per-section ones, so a
front that needs the theme adds the parameter to its own function and to its
own one line of the shared `case` — still one line each, and no front-56 commit
touching a file two other fronts are editing.

Front 34 owns the **modifier table** — the variants under the
`// ── front 34 — modifiers ──` banner in `tokens.bp`, the block of the same name
in `emilia.bp` (one `Variant`-returning fn per name, plus its arms of the shared
`case`), and nothing about emission. There are **83** of them, 82 variants and
the one flag: five breakpoints and their five `max-` mirrors (all ten read the
theme's `--breakpoint-*`, so an override moves the query AND the class hash),
`Dark` and eight other media features, nine interaction states, sixteen form
states, nine structural positions plus the indexed `Nth`/`NthLast`, nine
pseudo-elements, six `Group*` and eight `Peer*`, `Rtl`/`Ltr`/`Children`/
`Descendants`, and `Important` (decision 81), which is `markImportant` rather
than a `Variant`. `Dark` consumes front 54's `darkAtRule`/`darkSelector` and
never learns which `DarkMode` strategy is in force: `Media` puts the whole
strategy in the at-rule, `Class` and `Attribute` in the selector, and a cell
proves each. A RANGE is nesting, not a name — `md:max-xl:` is
`Token.Md([Token.MaxXl([…])])`.

Front 35 owns the **spacing and sizing sections** — `Pad`, `Margin`, `Size` and
`Space` in `tokens.bp`, and `padTokenToCss`, `marginTokenToCss`,
`sizeTokenToCss` and `spaceTokenToSheet` in `emilia.bp`, fenced by the
`// ── front 35 — spacing and sizing ──` banner in both files. Its one rule:
**no leaf resolves a length**. Every value answers front 54's `spacing(n)` /
`spacingHalf(n)`, so `calc(var(--spacing) * N)` is spelled once in the library
and the whole scale consumes it. `Pad`/`Margin`/`Size` are ordinary
`…TokenToCss` dispatchers adapted by `declSheet`; `Space` is the one dispatcher
here that answers a `Sheet`, because `space-y-*` declares on the element's
CHILDREN and so needs a selector outside the class. That selector is
**`pub fn siblingSelector()`** — `& > :not(:last-child)` — and **front 40 must
call it rather than re-spell it**: `divide-*` separates the same children the
same way, and two fronts writing the same selector by hand are two fronts that
will eventually write it differently.

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
  ... }` + a host expression per `#[@External.<targert>(...)]` declaration. There is
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
  `#[@External.<targert>(...)]` — already shipped primitives.
- **Not a runtime CSS engine.** No selector parsing and no preprocessor
  pipeline. Since front 56 a modifier is **not** a nested block: it is a
  `Variant` — an at-rule and a selector template with one `&` — and a
  modified token becomes a **sibling rule** hoisted out of the class
  body. Variants nest (`Md(Hover(...))`), and the outer one is outermost
  in the selector and in the at-rule list alike.
- **commonJS and erlang.** The `Stylesheet` host cell is a JS `Map` on
  commonJS and an ordered `[{Name, Body}]` list in the process dictionary on
  erlang (same `register`/`flush` contract); `hashHex` folds the same djb2 on
  both, so class names agree for ASCII bodies. `botopink test --target erlang`
  runs the suite (17/17). beam/wasm are not ported.

## Tree

The repository is a **workspace** (decision 75 of 1.0.10-beta): the root
`botopink.json` declares members and is never a package — no `src`, `files`,
`entry` or `dependencies`; `botopink build`/`botopink test` there is the
located refusal `botopink.json is a workspace, not a package — run this
command inside one of its members: emilia, emilia-card, emilia-cascade,
emilia-modifiers, emilia-spacing, emilia-theme`. Every `modules/*/`
and `examples/*/` holding a `botopink.json` is a member, named by its own
manifest. The **core is the member `modules/emilia/`**; `from "emilia"`
resolves to it, never to the umbrella.

```text
emilia/
├── AGENTS.md          ← you are here
├── docs.md            ← the user-facing token reference
├── botopink.json      ← WORKSPACE: name emilia · version · targets
│                        [commonJS, erlang] (the default every member
│                        inherits and may only restrict) · workspaces
│                        ["modules/*", "examples/*"]. Nothing importable.
├── modules/
│   └── emilia/        ← THE CORE — what `from "emilia"` gives a consumer
│       ├── botopink.json  name emilia · src src/ · entry root.bp ·
│       │                    target commonJS · targets [commonJS, erlang] ·
│       │                    files: root.bp · tokens.bp · theme.bp ·
│       │                    spacing.bp · emilia.bp · no dependencies
│       └── src/
│           ├── root.bp    ← `pub mod tokens; pub mod theme;
│           │                pub mod spacing; pub default mod emilia;` (the
│           │                v0 build folded the `stylesheet` module into
│           │                `emilia.bp` — see the README's "Deferred")
│           ├── theme.bp   ← front 54: the theme as a FLAT `ThemeEntry[]`
│           │                with validated namespace prefixes (`Ns` +
│           │                `nsPrefix`, the only place a prefix is
│           │                written), `DarkMode`, `defaultTheme()`, and
│           │                the extendTheme / clearNamespace /
│           │                emptyTheme / namespace / themeValue /
│           │                themeVar operations
│           ├── spacing.bp ← front 54: `spacing(n)` = `calc(var(--spacing)
│           │                * n)` and `spacingHalf(n)`. emilia NEVER
│           │                resolves a spacing value. Front 35 deleted
│           │                six of the seven `rem` ladders that were in
│           │                `emilia.bp`; the seventh, `flexGapScale`, is
│           │                `Flex.Gap`'s and is front 37's. `spacingHalf`
│           │                cannot spell `-0.5` (an i32 has no negative
│           │                zero), which is why `Margin.*.Neg.Half` is
│           │                `{1,2,3}` — front 54 owes a signed half step
│           ├── output.bp  ← front 56: the rule model (`Rule`, `Block`,
│           │                `Sheet`, `Variant`), `nestVariant` /
│           │                `markImportant`, the `\t`/`\n`/`\r` codec that
│           │                carries a `Sheet` through the string-keyed host
│           │                cell, `Options` + the five `with…`, and
│           │                `renderRule` / `renderDocument`. It declares NO
│           │                external and imports only `theme` — the cells
│           │                cannot move here (see "Gotchas")
│           ├── tokens.bp  ← the `Token` enum-shaped `type`
│           │                (`pub type Token { … }`): every section + the
│           │                modifier variants. Section headers live in
│           │                the docblock. A line comment inside the enum
│           │                body USED to be a parser gotcha; front 35
│           │                measured it green (`botopink test` both
│           │                targets + every example) and its banner
│           │                fences `Pad`/`Margin`/`Size`/`Space` inside
│           │                the body. Prefer the docblock anyway
│           └── emilia.bp  ← `emilia(tokens) -> string` + `flush()` + the
│                            `tokenToCss`/`tokensToCss` dispatchers + the
│                            `#\[@External\.<target>(…)]` host cells
│                            (`register`, `flushSheet`) + the 17 inline
│                            tests. It imports `Token` as
│                            `import { Token } from "tokens";` — naming the
│                            sibling module is **required**, see "Gotchas"
├── examples/
│   ├── emilia-cascade/ ← member `emilia-cascade` (an application: entry main.bp,
│   │                    targets [commonJS, erlang], `emilia` via
│   │                    { "workspace": true } and NOTHING else — it is the
│   │                    front 56 showcase: one card whose hover is a sibling
│   │                    rule, whose breakpoint is a hoisted `@media` read from
│   │                    the theme, whose reset arrives through `Options` in its
│   │                    own `base` layer, and whose document is layered so a
│   │                    project's own CSS can beat a utility deliberately.
│   │                    10 in-file `test {}`, green on both targets)
│   ├── emilia-theme/  ← member `emilia-theme` (an application: entry main.bp,
│   │                    targets [commonJS, erlang], `emilia` via
│   │                    { "workspace": true } and NOTHING else — it is the
│   │                    front 54 showcase and the proof that the theme
│   │                    surface crosses a package boundary through
│   │                    `from "emilia"`. 6 in-file `test {}`; the flushed
│   │                    document does NOT yet carry the theme — wrapping
│   │                    `themeCss`/`keyframeCss` in cascade layers is
│   │                    front 56's `withTheme`/`flushWith`)
│   ├── emilia-spacing/ ← member `emilia-spacing` (an application: entry
│   │                    main.bp, targets [commonJS, erlang], `emilia` via
│   │                    { "workspace": true } — the front 35 showcase: the
│   │                    multiplier scale, nine padding and margin
│   │                    directions, `auto`, negatives, child spacing, the
│   │                    thirteen `Size` sub-sections, and a centred article
│   │                    shell with a full-bleed header. Its last test is
│   │                    the front's whole argument: halving `--spacing`
│   │                    moves the `:root` block and not one byte of any
│   │                    rule. 12 in-file `test {}`, green on both targets)
│   └── emilia-card/   ← member `emilia-card` (an application: entry main.bp,
│                        target commonJS, `emilia` via { "workspace": true },
│                        `jhonstart` still by { git, branch } until jhonstart
│                        is a workspace too): 4 in-file `test {}` composing
│                        three class names + a Hover/Md modifier
└── scripts/
    ├── git-hooks/     ← the pre-commit gate (§ Local gate): `botopink test`
    │                    per `modules/*/` member, then `botopink build` per
    │                    example
    └── known-broken-examples.txt ← the examples allowed to fail
```

There is **no `modules/emilia-test/` yet**: front `02-packaging` step 4 creates
it once `01-std` steps 2–3 give it `std/asserts` and `std/snapshots` to stand on.

`.d.bp` files are NOT in the module tree (memory:
`project_libs_module_migration_done`); emilia has none today.

`.github/workflows/test.yml` — CI: `zig build test-libs -- --lib emilia
--target <t>` for `{commonJS, erlang}` on ubuntu and macos, plus `commonJS`
on windows (`escript` ships cleanly only on linux + macos), against
botopink-lang `vars.BOTOPINK_LANG_REF` (default `feat`). Both rows are hard
cells — no `allow_fail`. Under the workspace, `--lib emilia` restricts the
runner to the **core member** (the umbrella has no row); without `--lib` the
runner discovers every member — one row each, the example as an application.
The examples stage reads each example's own manifest target, so it is pinned
to the commonJS row and runs once.

## Maintainer rules

- **A variant selector carries exactly ONE `&`, and a reference row that
  carries two is the reference's problem.** `§ 3.2` spells `open` as
  `&:open, &:popover-open`; front 56's `checkVariantSelector` refuses it, and it
  is right to — a two-`&` template duplicates the rule it wraps, which is a
  stylesheet that is silently wrong in a browser days later. The fix is a
  selector, not an escape hatch: the two states go inside one `:is()`
  (`&:is(:open, :popover-open)`), which matches the same elements through one
  `&`. Any future row spelled as a selector LIST takes the same treatment.
  Separately and confirmed by the maintainer: **upstream v4 also carries the
  legacy `[open]` attribute in that row** (`&:is([open], :popover-open,
  :open)`), which the local reference's table omits — the next front to touch
  the row with upstream in hand adds it, and the test pinning today's spelling
  is what makes that show up as a change.
- **`markImportant` is not a `Variant`, so `Important` is not a variant arm.**
  Its arm answers `markImportant(tokensToSheet(inner, th))` and it adds no
  selector and no at-rule; it also produces ONE RULE PER INNER TOKEN, so a test
  over two tokens reads two rules rather than one rule of two declarations.
- **camelCase** all method/fn names (`tokenToCss`, `flushSheet`,
  `hashHex` — never `token_to_css` — memory:
  `feedback_camelcase_naming`).
- **A payload leaf NESTED INSIDE A SECTION cannot be constructed.**
  `Token.Color.Hex("#abc")` reds `'Hex' is not declared in any behavior
  implemented for 'Token'`; `val h: Token = .Color.Hex("#abc");` reds `unbound
  variable 'Color'`. The variant type-checks in a `case` pattern, so the
  surface looks complete and is not — `Color.Hex`, `Bg.Hex` and
  `Border.Color.Hex` are all reachable by an arm and by nothing else. A
  payload-carrying token has to be a **top-level** variant with builtin-typed
  fields (`Token.Alpha(percent: i32, inner: Token[])`, `Token.Hover(inner:
  Token[])`), which is what contract 4a requires of every one in the milestone.
- **Enum payload destructuring uses the NAMED FIELD** —
  `ColorHex(value: string)` is matched as `ColorHex(value) -> …`, not
  `ColorHex(v) -> …`. The positional binding parses but lowers to
  `undefined` (codegen relies on the field name to project the payload
  from the runtime variant record).
- **No line comment inside an enum body** (kept from the 1.0.2 parser; the
  1.0.3 field list accepts comments, but the body stays comment-free).
  Section headers go in the module docblock.
- **`botopink format` is applied to every source, `tokens.bp` included.** It
  used to reorder the authored `Token` body — payload variants printed before
  the sections — which is why this line once excluded the file. botopink-lang
  `37d3dc7` records member positions and no longer reorders, and decision 34
  withdrew the exemption. It is **red today** and was red before the workspace
  migration: `botopink format --check` in `modules/emilia/` prints `Formatted
  src/emilia.bp` (`root.bp` and `tokens.bp` unchanged) and
  `examples/emilia-card/` prints `Formatted src/main.bp` — the formatter moved
  under the compiler since the last `style(src)` sweep. Reformatting is a
  source change and belongs to a `style(src)` commit, not to a packaging one.
- **FIXED — `.Color.Red.500` and five siblings used to be DECLARED AND
  UNREACHABLE.** The compiler resolved a leading-dot section path
  (`tryResolveEnumSectionPath`, compiler-core `comptime/infer.zig`) by iterating
  `env.typeDefs` — which holds the synthesised section enums beside the real
  ones — and returning the first enum whose section tree carried the path,
  never consulting the expected type. `Token` carries `Color.Red.500` and so
  does `__Token__Border` (`Token.Border.Color`, whose Red and Gray run
  100/500/700), so hash order decided, and it decided against us for
  `.Color.Red.{100,500,700}` and `.Color.Gray.{100,500,700}`: every spelling
  red, including the fully qualified `Token.Color.Red.500`, which the resolver
  did not accept at all. It failed loudly (`type mismatch: expected Token, got
  __Token__Border`) rather than emitting the wrong CSS, and it flipped whenever
  the typedef set grew. botopink-lang `f01c508a` now prefers the enum the
  expected type names, accepts the fully qualified spelling, and refuses an
  ambiguous path naming both candidates instead of picking one; emilia
  `1cd39b2` asserts all six cells, and the two palette tests went from eight
  shades to eleven. **The standing rule survives the fix**: never resolve a
  collision like this by adding or renaming a type to flip the hash order —
  that is invisible, and the next front re-breaks it.
- **A section of `Token` is a type written by its path** — `Token.Text`,
  `Token.Text.Size`, `Token.Border.Color` (botopink-lang decision 8 §5.3b).
  The flat spelling (`TokenText`) names nothing and reds with a hint; a
  sub-dispatcher takes its own section, never the whole `Token`, so its
  `case` stays exhaustive without a `_` and a new member reds it.
- **Array type spelling is postfix** — `Token[]`, NOT `[Token]`. The
  prefix bracket parses only as an array literal (`[Token.PadX4]`).
- **`from "emilia"` only**, never a relative module path and never the
  directory. The name resolves to the member `modules/emilia/`, which is what
  `emilia-card` reaches through `{ "emilia": { "workspace": true } }` and what
  an outside consumer reaches through the git form — a member is reached by
  its manifest `name`, never by its path (decision 75).
- **Cross-module `#[@External.<targert>(...)]` symbol imports don't lower at v0** —
  `import { register };` from a sibling module resolves the type but
  the runtime symbol is `undefined`. Until the codegen path closes
  that gap, keep all `#[@External.<targert>(...)]` host-cell expressions in the
  module that USES them (currently `emilia.bp`).
- **`extend` is a language keyword, so the theme's composer is `extendTheme`.**
  `Name extend Type { … }` is the type-extension block, which makes `fn
  extend(…)` a parse error (`unexpected \`extend\``) — the front 54 spec, front
  33's and front 56's all write `extend`, and none of them parses. Every front
  that composes a theme writes `extendTheme`.
- **A lambda whose whole body is an `if` expression returns `undefined` on
  commonJS.** `xs.map({ x -> if (c) { a } else { b } })` lowers to
  `(x) => { (() => { … })(); }` — the IIFE is emitted as a statement and its
  value is never returned. erlang is unaffected, so the suite is green on one
  target and silently wrong on the other. Bind the branch to a `val` and yield
  it (`val out = if (c) { a } else { b }; out;`). Reported to botopink-lang as a
  codegen defect (the commonJS lambda tail-expression path).
- **`String.contains` does not lower inside a lambda over an inferred
  element.** `xs.filter({ e -> e.name.contains("x") })` emits `.contains(…)`
  verbatim on commonJS (`e.name.contains is not a function`) because the
  `contains` → `includes` rename runs off a receiver type the inferencer has not
  resolved inside the lambda; a direct receiver (`s.contains(…)`) and a record
  field outside a lambda both work. Hoist to a typed `val`, or use `endsWith` /
  `indexOf(…) != -1`.
- **FIXED — the commonJS `String` prelude's `charCodeAt` patch used to be
  self-recursive.** The backend installs the whole `String` behavior prelude
  into any module that uses a member needing a patch — `slice` is one,
  `split`/`indexOf`/`startsWith` are not — and that prelude wrote
  `String.prototype.charCodeAt = function(index) { return
  ((this.valueOf().charCodeAt(index) ?? -1) | 0); }`, which called the patch it
  had just installed. One `s.slice(…)` anywhere in a module therefore made every
  `.charCodeAt(…)` in the PROGRAM blow the stack — `hashHex`'s host template
  does, so every non-empty class body did. **The ban on `String.slice` in this
  library is lifted**: botopink-lang's prelude now calls `codePointAt`, which
  also fixed a second defect the first was hiding (the `?? -1` was dead, since
  native `charCodeAt` answers `NaN`, so commonJS answered `0` out of range where
  erlang answered `-1`), and a test now walks the embedded prelude and fails if
  any template calls the method it patches. Code that split on a separator to
  avoid `slice` is correct as written and need not be unwound.
- **FIXED — a `case` arm over a uniquely-named variant used to lower to
  `instanceof`, which does not cross a package boundary.** The commonJS backend
  lowered an arm whose variant name was unique in the program to `_s instanceof
  __Token__Text__Size$X3xl` and an arm whose name repeated to `_s.tag === "Lg"`.
  A consumer package **re-emits its own copy** of the enum classes, so a value
  built in `examples/emilia-card/` was never `instanceof` the class `emilia`
  matched against: the `case` fell through every arm and answered `undefined`,
  which is why `.Text.Size.X3xl` and `.Text.Size.Base` were missing from that
  example's class bodies. It was invisible inside emilia's own suite, where
  there is one copy of the classes. botopink-lang now compares the `tag` string
  for a uniquely-named variant too; the example asserts the full class bodies
  and is green. Kept here as the reason a cross-package `case` is worth a
  runnable example.
- **FIXED — `Array.reverse()` used to mutate its receiver on commonJS and not on
  erlang.** `val rev = xs.reverse();` left `xs` reversed on commonJS (native
  `Array.prototype.reverse` is in-place and the codegen called it directly) and
  untouched on erlang (`lists:reverse/1` is pure), so code reading the receiver
  again answered differently per target — green on one, silently wrong on the
  other. botopink-lang now emits `toReversed()`, which answers a new array. The
  workaround in `output.bp`'s `wrapAtRules` (mapping the list twice) is no
  longer required and may be simplified by whichever front next touches it.
  Kept as the reason a per-target divergence is worth a cell rather than a note.
- **A sibling-module import always names its module** — `import { Token } from
  "tokens";`, never the bare `import { Token };`. Both type-check, but commonJS
  lowers the bare form to `require("../module")`: a path that resolves while
  emilia is compiled on its own and not when it is a dependency, which is how
  `examples/emilia-card` came to build and then die with `Cannot find module
  '../module'`. Reported to botopink-lang as a codegen defect
  (`src/codegen/commonJS.zig`, the require path of a dependency's sibling
  module); naming the module is the workaround and reads better anyway.

## Test surface

- `botopink test` inside `modules/emilia/` (never at the root — the umbrella
  refuses) runs every module's in-file `test {}` blocks, **283/283** on
  commonJS and on erlang: 6 (`spacing.bp`) + 37 (`theme.bp`) + 45
  (`output.bp`) + 195 (`emilia.bp`). `emilia.bp`'s 195 are front 56's 33
  (below) plus front 33's 81 plus front 35's 31 plus front 34's 50 — two per
  variant family (the `Variant` halves and the CSS the row renders), the three
  dark-mode strategies a cell each, the ranges, a three-deep chain, the indexed
  rows, `Important`, the empty inner list, six walks over the whole table, and
  three end-to-end documents. Front 35's 31 cover the scale
  and the nine directions of `Pad` and of `Margin`, `Auto` and `Neg` on each,
  the thirteen `Size` sub-sections (fractions, the per-axis viewport unit, the
  named container and breakpoint widths read back through `themeValue`), the
  `Space` rule shape and its selector, and three end-to-end documents. Three of
  the 31 are the front's REGRESSION: a walk over **1074** `Pad`/`Margin`/`Space`
  leaves asserting the output carries no `rem`, none of `padding-x`,
  `padding-y`, `margin-x`, `margin-y`, and none of `m-0.25`, `m-0.5`, `m-1`,
  `m-2`, `margin-auto`; a fourth walks all **566** `Size` leaves for the same
  `rem`. Front 33's 81: 26 one-per-family `Color` grid tests (all 286
  cells since `1cd39b2` closed the resolver defect) and 26 for
  the `Bg.Color` mirror (all 286), plus `paletteVar`, the shade-survives pin,
  the named colours on both properties, the pre-33 paths, the legacy `Bg`
  leaves, the longhand/shorthand split, the rule shape of a colour token, and
  seven over `paletteEntries()` — its 286 entries, the two anchor values in
  upstream's spelling, white/black not duplicated, composition through
  `extendTheme`, the absence of an `@theme` block in emilia's own output and
  the end-to-end two-token document, and ten over the `Alpha` wrapper — the
  two rows the reference shows, four percentages, two tokens under one
  wrapper, a non-colour token, a keyword colour, both nesting orders with a
  modifier, `alphaWrap` on plain strings, the codec, and the end-to-end
  `bg-red-500/50` document, and three pinning the palette as a
  STRING-TO-STRING mapping with the enum only at its edge — a family and a
  shade that came from no leaf, the property name living in the dispatcher,
  and a project override reaching every rule through the reference. Front
  56's 33:
  - 8 leaf dispatchers (Text.Bold / Text.Size.Lg / Color.Black /
    Bg.White / Layout.Flex / Border.Rounded.Full / Effect.Shadow.Md,
    plus the shape of a section rule) — `Color.Black` reads
    `color:var(--color-black)` since front 33;
  - 5 modifier tests, each naming the row of the variant reference its
    expected selector comes from, plus the theme-driven breakpoint;
  - 2 codec tests walking the dispatcher's own output for a separator;
  - 5 drain tests pinning the commonJS and Erlang host templates
    against each other;
  - 8 public-surface smoke (empty-list class / single-token rule /
    mixed-token fold / hash collapse / a modifier as a sibling rule /
    two-flush independence / the stock render's theme layer and
    keyframes);
  - 5 cascade tests (the conflict rule, list order, call order, an
    unrelated reorder, a variant following the rule it varies).
  The async `flush()` returns `@Future<string>`; tests `await flush()`
  via the implicit `test {…}` future context shipped in bot-lang's
  `test-runner-async` commit.
- `examples/emilia-card/` is the member `emilia-card` and carries 4 in-file
  tests on V1 enum-section paths (`.Pad.All.__4`, `.Color.Red.600`, …), the
  flush one rewritten by front 56 to the layered document and the hoisted
  modifier. It
  **builds again**: jhonstart's `fix/context` front landed on its `feat`, so the
  `hooks.bp:109 use-without-context-effect` red that used to stop the build
  inside **jhonstart** (never inside emilia) is gone, and the example's line was
  deleted from `scripts/known-broken-examples.txt` — the list refuses to rot, so
  a listed example that builds fails the gate just as a red one does.

- `examples/emilia-modifiers/` is the member `emilia-modifiers` and is front
  34's worked example: a navigation bar that is stacked and dark-surfaced on a
  phone and a row from `md:` up, whose links read the bar's hover through
  `.group`; a form field whose error message is shown by its SIBLING's invalid
  state and nothing else; a table that stripes itself with `Odd`/`Even`/`Nth`
  and pins one declaration with `Important`; and the same panel rendered under
  all three `DarkMode` strategies, which give three different classes because
  the strategy reaches the rule. 16 in-file tests, green on commonJS and on
  erlang; it builds and runs. Two of its assertions pin OTHER fronts' output
  and say so — `Margin.*.__0` is `margin-left:0` rather than a `calc`, and
  `Border.Color.Red.__500` is still `border-color:red` because `Border.Color`
  is pre-front-33 and front 40 owns the rewrite; what this example pins there
  is the selector, which is front 34's.

- `examples/emilia-spacing/` is the member `emilia-spacing` and is front 35's
  worked example: the multiplier scale through `.Pad.All.*`, the nine padding
  and margin directions, `m-auto` and `mx-auto` on one ladder, a centred
  article shell (`max-w-3xl` + `mx-auto` + `px-6`) as one class and one rule,
  a full-bleed header whose negative `-mx-6` cancels it, fractions, the
  per-axis viewport unit, `size-12`, the six logical forms, and a comment
  thread spaced by `Space.Y` with a reply pulled up by `-mt-2`. Its last two
  tests are the front's argument: no token in the example emits a resolved
  length or a class fragment, and halving `--spacing` gives the SAME class with
  the SAME declarations — only the `:root` block moves. 12 in-file tests, green
  on commonJS and on erlang.

- `examples/emilia-cascade/` is the member `emilia-cascade` and is front 56's
  worked example: the reset supplied through `withBase`, the hover hoisted into
  `@media (hover: hover){.e_x:hover{…}}`, the breakpoint hoisted into
  `@media (width >= 48rem)` with its width read from `--breakpoint-md`, the four
  cascade layers in order, `withPrefix`, `withLayers(o, false)`,
  `withImportant(o, true)` and the two-flush contract. 10 in-file tests, green
  on commonJS and on erlang; it builds and runs.

- `examples/emilia-theme/` is the member `emilia-theme` and carries 10 in-file tests over
  the front 54 surface, imported across a package boundary (`from "emilia"`, the
  `{ "workspace": true }` form). It is green on commonJS and on erlang, builds, and runs
  (`botopink run` prints the resolved brand value, the `var(…)` reference form,
  `padding:calc(var(--spacing) * 4)`, the class name, the dark at-rule + selector, and
  the `<style>` document). Since front 56 the flushed document **does** carry the
  theme: `main` renders it with `flushWith(withTheme(defaultOptions(), th))`, and four
  of the ten tests cover the theme layer, the layer order, `withLayers(o, false)` and
  `withPrefix`.

## Spec / phase status

| Phase | Status |
| --- | --- |
| F0 — lib stand-up | DONE-then-undone (V0 surface dropped during V1 WIP) |
| F1 — fill out `Token` | DONE (V1 nested-section landed `3f77623`) |
| F2 — `tokenToCss` exhaustive | DONE for V1 (re-pinned under v0.beta.22 ecosystem-and-snap-tail F2) |
| F3 — modifier composition | DONE for V1 (re-pinned alongside F2) |
| F4 — `flush()` per-render | DONE — async (`@Future<string>`); test bodies await via implicit future context (bot-lang `<test-runner-async>` commit) |
| F5 — example + docs sweep | DONE — `examples/emilia-card/` migrated to V1 enum-section paths (`.Pad.All.__4`, `.Color.Red.__500`, …) + `await flush()` |
| 1.0.10-beta front 56 — cascade and output | DONE — `output.bp` + the host-cell and public-entry half of `emilia.bp` + `examples/emilia-cascade/`; steps 1–8. `flushSheet` is gone, `drainRules` takes its place, and document assembly happens once in botopink. Fronts 33–47 adapt with `declSheet(…)`, front 34 writes the variant table, fronts 35/40 write `…TokenToSheet`, front 44 writes `blockSheet`, front 55 writes `withBase`, front 59 writes the components layer |
| 1.0.10-beta front 34 — modifiers | DONE — the variant table in `tokens.bp` + `emilia.bp` under the front's banner; 83 modifiers; steps 1-7. One `Variant`-returning fn per name and no wrapping logic: front 56's `nestVariant` applies them |
| 1.0.10-beta front 33 — colour palette | DONE — steps 1–5. `Token.Color` and `Token.Bg.Color` are the 26 x 11 grid + the five named colours; `colorTokenToCss`/`bgColorTokenToCss` emit `var(--color-<family>-<shade>)` through `paletteVar` over front 54's `themeVar`/`nsPrefix`, so no arm discards its shade and no literal ladder is left; `paletteEntries()` carries the 286 OKLCH values from upstream 4.3.2 for a consumer to compose; `Alpha(percent, inner)` is upstream's `/N` suffix. **All 286 cells are reachable since `1cd39b2`**: `.Color.Red.{100,500,700}` and `.Color.Gray.{100,500,700}` were declared and unreachable while the compiler's leading-dot resolver guessed between `Token` and `__Token__Border`, and botopink-lang `f01c508a` closed it; see § Maintainer rules. Fronts 39/40/41/47 consume `paletteVar(family, shade)` and `paletteEntries()` |
| 1.0.10-beta front 35 — spacing and sizing | DONE — steps 1–5: `padTokenToCss`/`marginTokenToCss` emit real CSS properties and every leaf answers front 54's `spacing(n)` (the six `rem` ladders deleted, `padding-x:`/`padding-y:`/`margin-y:` and `m-0.25`/`m-1`/`margin-auto` gone, each pinned); `Pad` and `Margin` carry nine directions over the 35-leaf scale, `Auto` on every margin direction and a `Neg` sub-section on each — 936 leaves, walked by one test. **`Neg.Half` is `{1,2,3}`**: `spacingHalf(-0)` is `spacingHalf(0)`, so `-0.5` is unreachable until front 54 grows a signed half step. `Token.Size` carries thirteen sub-sections over `§ 8.1`–`§ 8.7`, 566 leaves, and spells no `rem`: the named container widths are `var(--container-*)` through a `containerVar` over front 54's `nsPrefix`/`themeVar`, `MaxW.Screen.*` is `var(--breakpoint-*)`. `Token.Space` is `space-x-*`/`space-y-*` — the one dispatcher here answering a `Sheet`, under `siblingSelector()`; its child selector and the `--tw-space-*-reverse` names are a PROPOSAL, the local reference carrying no `space-*` row at all. `examples/emilia-spacing/` is the showcase (12 tests). 1640 leaves across the four sections; 202 → 233 inline tests in `modules/emilia`, green on commonJS and erlang |
| 1.0.10-beta front 54 — theme | DONE — `theme.bp` + `spacing.bp` + `examples/emilia-theme/`; steps 1–7. Front 33 hands over `paletteEntries() -> ThemeEntry[]`; fronts 33–47 rewire the dispatchers; front 56 wraps `themeCss`/`keyframeCss`; front 34 consumes `DarkMode` |

Spec lives in
[`tasks/v0.beta.20/specs/ecosystem.md`](../../tasks/v0.beta.20/specs/ecosystem.md);
the V1 re-author follow-up rides on
[`tasks/v0.beta.22/specs/05-ecosystem-and-snap-tail.md`](../../tasks/v0.beta.22/specs/05-ecosystem-and-snap-tail.md)
F2's deferred half.

## Local gate

`scripts/git-hooks/pre-commit` is the tracked pre-commit gate. It is
self-contained: it sources `scripts/git-hooks/lib/runner-standalone.sh`
from this repository and reaches nothing outside it, so a standalone
clone, a checkout inside the botopink meta workspace and a worktree run
the same gate. Install it once per clone:

```sh
git config core.hooksPath scripts/git-hooks
```

`core.hooksPath` is per clone and applies to every worktree of it. The
gate checks staged files for conflict markers, then — because the root
`botopink.json` is a workspace — runs `botopink test` **inside every
`modules/*/` that holds a `botopink.json`**, each on its own manifest
target (`erl` and `node` on `PATH`); a red member fails the gate and names
the re-run command. (A root manifest without `"workspaces"` keeps the old
single `botopink test` over `src/` + `test/`.) So a source file that does not
parse — e.g. one carrying markdown escapes like `#\[@External\.node(` —
still fails the commit. The compiler binary is located via (in order)
`$BOTOPINK_BIN`, the nearest ancestor
`repository/botopink-lang/zig-out/bin/botopink`, then `$PATH`. If none
resolve, the gate prints a yellow warning and exits 0 — CI runs the full
suite and catches any regression there. Never commit with `--no-verify`;
fix the red instead.

After `botopink test`, the gate builds every `examples/*/` that has a
`botopink.json` (`runExamplesGate`, each with its own manifest target,
into a throwaway `--out`); CI runs the same function once per workflow.
`scripts/known-broken-examples.txt` lists the examples allowed to fail —
`examples/<name>  <reason>` per line — and cannot rot: a listed example
that builds, or a listed path that no longer exists, fails the gate too.
When a fix makes an example build, delete its line in the same commit. The list may be absent,
empty or hold only `#` comments — each means no example is allowed to fail.
The list is empty today: `examples/emilia-card` was listed while jhonstart `feat`
did not compile against botopink-lang `feat`, and its line came out once
jhonstart's `fix/context` front landed (see § Test surface). It builds **and runs**
again (`botopink run` prints the tree, the three `e_<hash>` class names and the
`<style>` block); it depends on jhonstart, so CI checks jhonstart out
beside emilia before the examples gate. Its builder calls pass `attrs`
explicitly (`h1([…], [])`) — parameter defaults are not applied by the compiler
yet (botopink-lang 1.0.4-beta 06 N1) — and its `main` is
`#[@future] fn main() -> @Future<void>` so `flush()` can be awaited.
