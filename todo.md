# front/54-theme — emilia front 54: the theme, the spacing scale, the custom-property block

Worktree: .tasks/emilia-theme, branch front/54-theme (from emilia feat d539978, a workspace).
Spec: specs/1.0.10-beta/05-emilia/54-emilia-theme/README.md (steps 1..N, the Mechanism section is
binding: flat entry list + `Ns` prefixes, `spacing(n)` = `calc(var(--spacing) * n)`, `@theme static`
always, `DarkMode` shipped here and consumed by front 34).

## Path translation — the spec predates the workspace migration
The spec's `Owns:` says `repository/emilia/src/theme.bp`. emilia is now a **workspace**: the core
lives in `modules/emilia/`, so every owned path is `modules/emilia/src/…`, the module list is
`modules/emilia/botopink.json` `files`, and `pub mod` lines go in `modules/emilia/src/root.bp`.
emilia's tests are **inline `test { … }` blocks** in the source (there is no `test/` directory);
the spec's `test/theme_test.bp` is therefore inline tests in `theme.bp`/`spacing.bp` unless
creating `modules/emilia/test/` is clearly better — decide, and record which in the report.

## Owns
- `modules/emilia/src/theme.bp`, `modules/emilia/src/spacing.bp` (new)
- the `pub mod theme;` / `pub mod spacing;` lines in `modules/emilia/src/root.bp`
- the two matching entries in `modules/emilia/botopink.json` `files`
- the tests for both, `AGENTS.md`, `CHANGELOG.md`, `docs.md` where the surface is documented

## Does not touch
- `modules/emilia/src/tokens.bp` (no new `Token` variant) and `src/emilia.bp` (no new `tokenToCss`
  arm) — fronts 33–47 rewire their dispatchers to `spacing()`/`theme` later, not here
- `examples/**` beyond what a green gate needs, other libraries, `specs/`, other worktrees

## Steps (from the README — tick as each lands)
- [ ] 0 — baseline: `botopink test` in `modules/emilia` per target, and the literal ladders the
      README names (`emilia.bp:190-220`, `:231-240`, `:242-260`, `:308-316`) re-read at HEAD
- [ ] 1 — `ThemeEntry`, `DarkMode`, `Theme`, `Ns` + `nsPrefix` (the only place a prefix is written)
- [ ] 2..N — the rest of the README's steps, in order; `extend` refuses an entry outside the known
      prefixes (refuse, never accept-and-ignore, no flag — decision 67)
- [ ] the `@theme` custom-property block is emitted, always (`@theme static` semantics)
- [ ] `spacing(n)` answers `calc(var(--spacing) * n)` — emilia never resolves a spacing value
- [ ] docs + AGENTS.md + CHANGELOG in the same commit; gate green; push front/54-theme
