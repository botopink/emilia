# emilia

> Styled-components + tailwind-style CSS-in-bp library for botopink/jhonstart —
> pure botopink, driven by `@Expr` and `@ExprCustom`. Reached via `from "emilia"`.

`emilia` is the CSS sibling of `jhonstart`'s `html` and `erika`'s `erika "…"`:
the same `template fn (comptime t: @Expr<string>) -> @ExprCustom<T>` mechanism
applied to a tiny CSS sub-language. It collects a styled `Element` decoration
(a `className` + a stylesheet entry) at compile time and renders the collected
stylesheet alongside jhonstart's `renderToString` walk.

## Forms

**`styled.<tag>` (styled-components):**

```bp
import {styled} from "emilia";

val Card = styled.div """
    background: white;
    padding: 16px;
    border-radius: 8px;
""";

val title = Card([text("hello")]);   // a jhonstart Element with a stable className
```

**`css"""…"""` (anonymous, reusable across tags):**

```bp
import {css} from "emilia";

val pad = css """
    padding: 16px;
    border-radius: 8px;
""";

val card = div([attr(pad), text("hi")]);
```

**`tw"…"` (tailwind-style utility shorthand, expanded at comptime):**

```bp
import {tw} from "emilia";

val card = div([attr(tw "p-4 rounded-md bg-white"), text("hi")]);
```

The full grammar, the comptime expansion model, and the jhonstart integration
contract live in [docs.md](docs.md) (added with the implementation). The intent
spec is tracked in the parent workspace under
`tasks/v0.beta.19/specs/emilia.md`.

## Status

Seed. The library is intent-only right now — the spec drives the work; no
runtime is shipped from this commit. See [AGENTS.md](AGENTS.md) for the design
rules and the "zero compiler surface" constraint.

## License

Same as the parent botopink workspace.
