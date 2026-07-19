---
title: Home
---

**The [tq](https://github.com/tsvsheet/tq) engine as an importable Go library.** tq is the query language of the tsvsheet ecosystem — a `|`-separated pipeline of relational verbs (`select`, `where`, `derive`, `sort`, `group`, …) over a TSV or [tsvt](https://github.com/tsvsheet/tsvsheet) table, with every embedded expression written in the tsvsheet formula language. `go-tq` parses a query, plans it against your table's header, and executes it — TSV in, TSV out — delegating every expression evaluation to the [go-tsvsheet](https://github.com/tsvsheet/go-tsvsheet) engine so a predicate in tq and a formula in a sheet can never disagree.

The library is the one implementation of tq semantics, behind the [`tq`](https://github.com/tsvsheet/tq.go) CLI. Import it to query tabular data from your own Go programs.

## Install

```sh
go get github.com/tsvsheet/go-tq
```

API reference: **[pkg.go.dev/github.com/tsvsheet/go-tq](https://pkg.go.dev/github.com/tsvsheet/go-tq)**.

## Quickstart

```go
package main

import (
	"fmt"
	"os"
	"strings"

	tq "github.com/tsvsheet/go-tq"
)

func main() {
	program, err := tq.Parse(`where [stars] > 1000 | derive ratio = round([stars] / [forks], 2) | select name, ratio | sort -ratio`)
	if err != nil {
		panic(err) // matchable with errors.Is(err, tq.ErrSyntax)
	}

	input := "name\tstars\tforks\nglaze\t2400\t120\nfoo\t80\t10\ngloo\t5100\t300\n"
	table, err := tq.ReadTable(strings.NewReader(input), tq.Options{})
	if err != nil {
		panic(err)
	}

	out, err := program.Run(table, tq.Options{})
	if err != nil {
		panic(err) // e.g. errors.Is(err, tq.ErrUnknownColumn)
	}
	_ = tq.WriteTable(os.Stdout, out)
	fmt.Println()
}
```

`Parse` compiles the query once; the resulting `Program` is an immutable value, safe to reuse concurrently across tables. `Run` plans against the table's header first — an unknown column, a raw A1 cell reference, or a name-requiring verb in headerless mode fails before any row is touched — then executes the stages.

## The model

- **The first row is the header** by default; `Options.IsHeaderless` switches to positional-only references (`[N]`).
- **Columns, never cells.** Expressions reference columns as `[name]` or `[N]`; in stage positions bare identifiers work (`select name, stars`). Raw A1 references are rejected at plan time.
- **Expressions are tsvsheet expressions** — same operators, functions, coercions, and error values, evaluated by the go-tsvsheet engine. The one syntactic difference: `|` always separates stages, so the formula pipe sugar is unavailable inside a tq expression.
- **Values, not formulas.** When the input contains `=formula` cells, the sheet is computed first and the query sees computed values (`Options.IsRaw` skips this). A computed error value like `#DIV/0!` is data: it projects, sorts, and groups by its text — and under `Options.IsStrict` the first error value aborts the run instead.

## The verbs

`select` · `drop` · `where` · `derive` · `rename` · `sort` (stable, typed total order, `-` descending) · `distinct` · `limit` · `offset` · `group … { name = aggregate, … }` — aggregates are ordinary tsvsheet functions (`sum`, `avg`, `counta`, …) over the group's column ranges. The language is specified normatively in the [tq](https://github.com/tsvsheet/tq) repo's SPECIFICATION.

Every verb also exists as a Go constructor (`tq.Select`, `tq.Where`, `tq.GroupBy`, …) building the same `Program` values programmatically — parsed and built programs behave identically by contract.

## Scope

go-tq is not an analytical database. For heavy analytics over large value-only TSVs — joins, window functions, columnar scans — embed a SQL engine like [DuckDB](https://duckdb.org) instead; tq does not compete there. go-tq's value is being the tsvsheet ecosystem's query engine as an ordinary Go dependency: it computes `.tsvt` formulas before querying, evaluates predicates with the very engine the sheet uses, preserves raw cell text with no type inference, and embeds anywhere a Go library fits — no external database engine to carry.
