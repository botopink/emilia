# emilia · CHANGELOG

## Unreleased — v0.beta.22

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
