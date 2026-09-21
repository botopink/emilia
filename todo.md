# front/54-theme — emilia front 54: the theme, the spacing scale, the custom-property block

Worktree: .tasks/emilia-theme, branch front/54-theme (from emilia feat d539978, a workspace).
Spec: specs/1.0.10-beta/05-emilia/54-emilia-theme/README.md.

## STATUS: DONE — steps 0..7 all landed green, both targets.

Baseline (step 0, at d539978): `botopink test` in `modules/emilia` = **17 passed, 0 failed**
on `--target commonJS` and on `--target erlang` (one module, `emilia.bp`). The literal
ladders the README names were re-read at HEAD and its claims check out: `padScaleX`
(`emilia.bp:190`), `padScaleY` (`:201`), `padScaleAll` (`:211`), `marginScaleX` (`:230`),
`marginScaleY` (`:241`), `marginScaleAll` (`:251`), `flexGapScale` (`:311`) — seven copies of
the same ladder, `marginScaleX` emitting `m-0.25`/`m-0.5`/`m-1`/`m-2` (class names, not CSS
values) under `margin-x:` (`:227`), which is not a CSS property.

After: `modules/emilia` = **6/6 (spacing.bp) + 37/37 (theme.bp) + 17/17 (emilia.bp)**;
`examples/emilia-theme` = **6/6**; `examples/emilia-card` builds; both targets throughout.

## Path translation applied
emilia is a workspace, so every owned path is `modules/emilia/src/…`, the module list is
`modules/emilia/botopink.json` `files`, and `pub mod` lines are in `modules/emilia/src/root.bp`.

**Tests are INLINE `test { … }` blocks**, not a new `modules/emilia/test/`. Reason: emilia's
entire suite is inline; the member manifest has an explicit `files` list under `src: "src/"`
and no `test` key, so a `test/` directory would need a manifest change beyond this front's
Owns list, and the pre-commit gate + CI both run `botopink test` per `modules/*/` member,
which already picks up inline blocks.

## Steps
- [x] 0 — baseline, measured (above)
- [x] 1 — `ThemeEntry`, `DarkMode`, `Theme`, `Ns` + `nsPrefix` + `allNamespaces`
- [x] 2 — `defaultTheme()`, the default tables, in declaration order
- [x] 3 — `extendTheme` / `clearNamespace` / `emptyTheme` / `namespace` / `themeValue` /
      `themeVar`, with the refusal
- [x] 4 — `spacing(n)` / `spacingHalf(n)` in `spacing.bp`
- [x] 5 — `themeCss(th)` / `keyframeCss(th)`
- [x] 6 — `darkAtRule(th)` / `darkSelector(th)` / `withDarkMode(th, mode)`
- [x] 7 — a theme is a module: `examples/emilia-theme/` + the composition test
- [x] the `@theme` block is emitted always (`@theme static` semantics)
- [x] `spacing(n)` answers `calc(var(--spacing) * n)`; emilia never resolves a spacing value
- [x] docs + AGENTS.md + CHANGELOG in the same commit; gate green

## The README did not survive contact with the language in three places
1. **`extend` is a reserved keyword** (`Name extend Type { … }`, the type-extension block),
   so `pub fn extend(th, entries)` is a parse error. Shipped as **`extendTheme`**. The spec,
   front 33's README, front 56's and `05-emilia/test-snap.md` all write `extend`; none
   parses. Fronts 33 and 56 must use `extendTheme`.
2. **The refusal cannot "fail the build".** `extendTheme` takes a runtime `ThemeEntry[]` —
   front 33 hands over `paletteEntries()`, a call, not a literal — so no comptime check can
   see it. Shipped as a hard `@panic` naming the offending name and listing the nineteen
   prefixes: it refuses, it never accepts-and-ignores, and there is no flag. Verified out of
   tree on both targets. The acceptance's "a test asserts the ABSENCE of the entry" is the
   accept-and-ignore behaviour decision 67 forbids, so the case is a note in the test
   section instead (the rakun convention the front's own Test plan names).
3. **`examples/theme-example.bp` as written imports front 56** (`defaultOptions`,
   `withTheme`, `flushWith`). Shipped as `examples/emilia-theme/`, a real workspace member
   asserting everything expressible today; the flushed-document assertion is 56's.

## Two botopink-lang defects found and worked around (both in AGENTS.md § Maintainer rules)
- A lambda whose whole body is an `if` expression returns `undefined` on commonJS:
  `xs.map({ x -> if (c) { a } else { b } })` lowers to `(x) => { (() => { … })(); }` — the
  IIFE emitted as a statement, its value never returned. erlang is unaffected, so the bare
  form is green on one target and silently wrong on the other.
- `String.contains` does not lower inside a lambda over an inferred element:
  `xs.filter({ e -> e.name.contains("x") })` emits `.contains(…)` verbatim on commonJS.
  A direct receiver and a record field outside a lambda both work.

## Left for fronts 33–47 / 56
- `tokens.bp` gained no `Token` variant and `emilia.bp` gained no `tokenToCss` arm: rewiring
  the dispatchers to `spacing()` and to the theme is 33–47's, and deleting the seven `rem`
  ladders is front 35's step.
- Front 33 hands over `paletteEntries() -> ThemeEntry[]`; the composed theme is
  `extendTheme(defaultTheme(), paletteEntries())`. Written identically here and in 33.
- Front 56 wraps `themeCss(th)` and `keyframeCss(th)` into the cascade layers.
- Front 34 consumes `DarkMode` through `darkAtRule` / `darkSelector` and owns the `Dark`
  token.
