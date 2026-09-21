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
- **Not a runtime CSS engine.** No selector parsing, no nested
  selectors beyond the modifier wrappers (`Hover`, `Focus`, `Active`,
  `Md`, `Lg`, `Xl`), no preprocessor pipeline. The scope is a flat
  declaration list per class with the modifier wrappers nested ONE
  level deep (and themselves nestable, e.g. `Md(Hover(...))`).
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
command inside one of its members: emilia, emilia-card`. Every `modules/*/`
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
│       │                    files: root.bp · tokens.bp · emilia.bp ·
│       │                    no dependencies
│       └── src/
│           ├── root.bp    ← `pub mod tokens; pub default mod emilia;` (the
│           │                v0 build folded the `stylesheet` module into
│           │                `emilia.bp` — see the README's "Deferred")
│           ├── tokens.bp  ← the `Token` enum-shaped `type`
│           │                (`pub type Token { … }`, 1.0.3 surface): every
│           │                section + the modifier variants. Section
│           │                headers live in the docblock; NEVER a line
│           │                comment inside the enum body (parser gotcha)
│           └── emilia.bp  ← `emilia(tokens) -> string` + `flush()` + the
│                            `tokenToCss`/`tokensToCss` dispatchers + the
│                            `#\[@External\.<target>(…)]` host cells
│                            (`register`, `flushSheet`) + the 17 inline
│                            tests. It imports `Token` as
│                            `import { Token } from "tokens";` — naming the
│                            sibling module is **required**, see "Gotchas"
├── examples/
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

- **camelCase** all method/fn names (`tokenToCss`, `flushSheet`,
  `hashHex` — never `token_to_css` — memory:
  `feedback_camelcase_naming`).
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
  refuses) runs `src/emilia.bp`'s 17 in-file `test {}` blocks, 17/17 on
  commonJS and on erlang:
  - 7 leaf dispatchers (Text.Bold / Text.Size.Lg / Color.Black /
    Bg.White / Layout.Flex / Border.Rounded.Full / Effect.Shadow.Md);
  - 4 modifier composition tests (Hover / Md / multi-token / nested);
  - 6 public-surface smoke (empty-list class / single-token rule /
    mixed-token join / hash collapse / Hover wrap in registered class /
    two-flush independent blocks). The async `flush()` returns
    `@Future<string>`; tests `await flush()` via the implicit
    `test {…}` future context shipped in bot-lang's `test-runner-async`
    commit.
- `examples/emilia-card/` is the member `emilia-card` and carries 4 in-file
  tests on V1 enum-section paths (`.Pad.All.__4`, `.Color.Red.__500`, …). It
  **builds again**: jhonstart's `fix/context` front landed on its `feat`, so the
  `hooks.bp:109 use-without-context-effect` red that used to stop the build
  inside **jhonstart** (never inside emilia) is gone, and the example's line was
  deleted from `scripts/known-broken-examples.txt` — the list refuses to rot, so
  a listed example that builds fails the gate just as a red one does.

## Spec / phase status

| Phase | Status |
| --- | --- |
| F0 — lib stand-up | DONE-then-undone (V0 surface dropped during V1 WIP) |
| F1 — fill out `Token` | DONE (V1 nested-section landed `3f77623`) |
| F2 — `tokenToCss` exhaustive | DONE for V1 (re-pinned under v0.beta.22 ecosystem-and-snap-tail F2) |
| F3 — modifier composition | DONE for V1 (re-pinned alongside F2) |
| F4 — `flush()` per-render | DONE — async (`@Future<string>`); test bodies await via implicit future context (bot-lang `<test-runner-async>` commit) |
| F5 — example + docs sweep | DONE — `examples/emilia-card/` migrated to V1 enum-section paths (`.Pad.All.__4`, `.Color.Red.__500`, …) + `await flush()` |

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
