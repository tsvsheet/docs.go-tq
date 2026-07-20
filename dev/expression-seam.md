# How go-tq consumes the go-tsvsheet expression seam

go-tq evaluates every embedded expression — `where` predicates, `derive` assignments, group aggregates — through [go-tsvsheet](https://github.com/tsvsheet/go-tsvsheet)'s standalone expression API: `CompileExpr` compiles the text, `Expr.Eval` evaluates it against a grid, and `FormatValue` renders the result as canonical cell text. This page describes how that seam is consumed, for contributors working in [internal/plan](https://github.com/tsvsheet/go-tq/tree/main/internal/plan) and [internal/exec](https://github.com/tsvsheet/go-tq/tree/main/internal/exec).

## The invariant: zero expression evaluation in go-tq

go-tq implements no expression evaluation of its own. Every operator, function, coercion, and error value comes from go-tsvsheet: the only production call sites into the engine are `Compile` in [internal/plan/expr.go](https://github.com/tsvsheet/go-tq/blob/main/internal/plan/expr.go) (`tsvsheet.CompileExpr`) and `evalText` in [internal/exec/exec.go](https://github.com/tsvsheet/go-tq/blob/main/internal/exec/exec.go) (`tsvsheet.FormatValue(expr.Eval(g, env.Compute))`). Because there is no second evaluator, divergence from tsvsheet expression semantics is structurally impossible — go-tq cannot compute `1+1` differently from a `.tsvt` formula, because go-tq never computes `1+1`.

## Plan-time rejection of cell references

tq addresses columns, never cells. The grammar admits raw A1 references (`B2`) and sheet-qualified references so they parse, but `planner.exprPlan` in [internal/plan/expr.go](https://github.com/tsvsheet/go-tq/blob/main/internal/plan/expr.go) rejects them at plan time with `ErrCellRef`, naming the offending reference and expression — before any row is processed. Nothing that reaches evaluation can address a cell of the author's table directly; the only cell references the engine ever sees are the synthetic ones the splice writes.

## Token-span splicing of column references

An expression reaches the plan layer as its original source text ([internal/ast](https://github.com/tsvsheet/go-tq/tree/main/internal/ast) preserves `Expr.Src`). `exprPlan` re-parses it for token spans (`ast.ParseExprInfo`) and resolves every `[name]`/`[N]` column reference against the schema; `splice` then rewrites the expression to pure tsvsheet form:

- each column-reference token span is replaced by its A1 rendering;
- every other byte of the author's expression passes through unchanged — there is no pretty-printing;
- bytes inside string literals are never touched;
- inter-token whitespace runes the tq grammar allows but a tsvsheet formula does not (TAB, CR, LF — a tq program may span lines) normalize to spaces.

There are two renderings, matching the two evaluation shapes:

- _Row stages_ (`where`, `derive`) use `rowRef`: `[stars]` resolved to column position 2 splices to `C1` — a cell reference into a one-row synthetic grid.
- _Group aggregates_ use `rangeRefAt`: for a group of `h` rows, `[stars]` splices to `C1:Ch` — the whole column across the group's grid.

Column letters are bijective base-26 (`A`–`Z`, `AA`, …), produced by `a1Col`.

## Synthetic-grid evaluation

Evaluation never sees the author's table directly; [internal/exec](https://github.com/tsvsheet/go-tq/blob/main/internal/exec/exec.go) builds a synthetic `tsvsheet.Grid` per evaluation:

- `rowGrid` lays one table row out as a single grid row, padded to the stage width, so its cells sit at `A1`, `B1`, … — the addresses row-stage splices produce.
- `gridOf` lays a whole group out, one grid row per table row, each padded to the stage width, so a spliced `C1:Ch` range covers exactly the group's column.

`evalText` is the single evaluation call: the compiled `Expr`, the synthetic grid, and the run's `ComputeOptions` (clock, limits, loader/fetcher gating) go in; the canonical cell text comes out, with array results reduced to their scalar-context value by `FormatValue`. `Eval` returns spreadsheet error values, never Go errors; strict mode detects them by matching the computed text against the engine's closed error-value set, which round-trips through `FormatValue` by construction.

## Compile once; the per-group cache

Each stage expression goes through `CompileExpr` exactly once per unique (expression, shape), at plan time — never per row:

- Row-stage expressions compile when the plan is built (`whereStage` and `assign` in [internal/plan/plan.go](https://github.com/tsvsheet/go-tq/blob/main/internal/plan/plan.go)).
- `where` compiles its predicate twice: the `(expr)=TRUE` form the filter evaluates on the common path (its `TRUE` proves the value is the boolean `TRUE`), and the raw form strict mode consults to tell an authentic author error (abort) from a wrapper artifact (drop).
- A group aggregate's spliced text depends on group height (the `C1:Ch` range), so `heightCompiler` returns a caching per-height compiler: a `map[exec.Height]tsvsheet.Expr` memoizes one `CompileExpr` per unique height seen, and the compiler is validated eagerly at height 1 so a seam failure surfaces at plan time rather than mid-run.

## The seam's error surface

`Compile` — the production `CompileFunc` — wraps a `tsvsheet.CompileExpr` failure as tq's `ErrSyntax` carrying the go-tsvsheet detail via `With`, so `errors.Is` reaches both engines' sentinels. The compiler is injected through `plan.Options.Compile` (`nil` selects the production one), which is how tests exercise the seam's failure paths with failing compilers.
