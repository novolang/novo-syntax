# novo-syntax

novo-lang's own front end, as a library: the lexer, the parser, the syntax
tree, a visitor, a printer, name resolution, the effects every function
performs, and the queries an editor asks. The grammar is the one in
novo-lang's language specification, `SPEC.md`, and the authority for every
answer here is the compiler in the
[novo-lang repository](https://github.com/klhjensen/novo). Nothing in this
package opens a file. Source arrives as a string the caller already holds, and
every answer is a value.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **lexer** turns source text into a list of **tokens**. A token here carries no
text of its own. It carries a **span**, a half-open byte range `{ start, stop }`
into the source the caller holds. An **`NsySource`** is that text paired with a
line table, built once, so a span can be turned into a line and a column.

novo-lang's layout is **indentation-sensitive**, which means the lexer emits
tokens the author did not type: a NEWLINE at the end of a logical line, an
INDENT when a block opens and a DEDENT when it closes. It also suppresses
newlines inside brackets, allows a block lambda to escape that suppression, and
continues a line that begins with an operator. Those rules are what the
language's layout is.

**Trivia** are comments and runs of blank lines. In this package they are
tokens, in source order, with spans, rather than being skipped. A lexical
mistake is a token too, so a bad escape halfway down a file does not cost the
tokens below it.

A **parser** turns tokens into a **tree**. Each syntactic category is a pair: a
`kind` enumeration that says what the node is, wrapped in a struct that carries
its span. `NsyExpr` is `{ kind: NsyExprKind, span }`. Every node has a span,
literals included.

The parse always answers a tree. A production that fails becomes an error node
at the span that failed, the parser resynchronises at the next statement or
declaration boundary, and the faults come back beside the tree.

**Resolution** binds every name occurrence to the declaration it names, and
indexes every declaration. It is not type checking: nothing here infers a type,
selects an implementation or monomorphises.

An **effect row** is the set of effects a function performs, directly and
through everything it calls. The specification charges a function the union of
both. A **declared clause** is what the author wrote in the signature, and the
two are different things: the clause is a claim and the row is the measurement.

## Install

```
novo pkg add novo-syntax
```

## Example

```novo
use nsytoken
use nsylex
use nsyparse
use nsyprint
use nsyquery

fn main() [io]
    // Pair the source text with its line table. Every span indexes into it.
    let src = nsytoken.source("fn main() [io]\n    println(\"hi\")\n")

    // Lex the whole file. Comments and blank lines are tokens in this list.
    let toks = nsylex.lex(src)

    // Parse. This always answers a tree, even for a file that does not compile.
    let parsed = nsyparse.parse_tokens(src, toks)

    // is_clean is the success test: no faults were recorded.
    if nsyparse.is_clean(parsed)
        // Print the tree back as canonical source.
        println(nsyprint.to_source(src, toks, parsed.tree, nsyprint.canonical()))

    // One entry per top-level declaration, for an editor's outline.
    for item in nsyquery.outline(src, parsed.tree)
        println(item.name)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `nsytoken` | The span, the source with its line table, the keyword and punctuation tables, the trivia kinds, the lexical faults, and the token itself. |
| `nsylex` | The lexer as a value. Feed it bytes and drain the tokens they completed, or lex a whole file in one call. |
| `nsyast` | The syntax tree: six category pairs, their kind enumerations, and the small functions that read a declaration's name, span and annotations. |
| `nsyparse` | Tokens to a tree, the faults found on the way, and the recovery policy as a public predicate. |
| `nsyvisit` | Walking the tree and rewriting it, as two traits with an effect parameter. |
| `nsyprint` | The tree back to source text, into a buffer the caller owns, under a style value. |
| `nsycheck` | What every name refers to, over a signature table the caller supplies. |
| `nsyeffect` | The effect row of every function in a file, and the comparison of a row against a declared clause or a budget. |
| `nsyquery` | The questions an editor asks: the path at an offset, what kind of place an offset is, the outline, the fold ranges, the references and the rename spans. |

## How to choose an entry point

**A tool that reads a whole file calls `nsylex.lex` and `nsyparse.parse`.**
`parse` lexes and parses in one call. `parse_tokens` is the same over a token
list the caller already has, which is what a tool that also wants the tokens
uses.

**A tool that re-reads a file on every keystroke uses the lexer as a value.**
`nsylex.new` starts one, `feed` takes the next run of bytes and answers a new
lexer and the tokens that run completed, and `finish` closes the file.
`nsylex.needs_more` says whether the input so far ended mid-construct, which is
the question a read-eval loop asks. `nsylex.relex_from` and `nsylex.splice` are
how an edit is applied to a token list without re-lexing the whole file.

**A pass over the tree implements `NsyVisitor`.** `nsyvisit.walk_tree` costs
whatever the visitor costs, so a visitor that only counts pays nothing and one
that writes diagnostics pays for the writing. `NsyRewriter` is the same shape
for a pass that produces a new tree.

**A formatter calls `nsyprint`.** Every entry point appends into a `str.builder`
handle the caller owns. `to_source` is the convenience for a caller that wants a
string. `is_canonical` and `first_difference` are the two questions a
check-only mode asks.

**An editor calls `nsyquery`.** `path_at` answers the whole chain of nodes from
the file down to the innermost one, in one traversal. `place_at` answers from
the tokens rather than the tree, which is what completion needs, because the
offset completion fires at is usually in a file that does not parse.

## The rules a user needs

1. **A token carries a span, not text.** `nsytoken.slice` turns a span into a
   string against the source it came from. A span from one `NsySource` means
   nothing against another.
2. **A span is half-open.** `start` is the first byte and `stop` is one past the
   last. An empty span has `start == stop`, which is what an insertion point is
   and what a zero-width layout token gets.
3. **Comments and blank lines are in the token stream.** Use
   `nsytoken.without_trivia` for a list without them. A formatter reads them
   from the stream, and there is nothing to re-scan.
4. **A lexical mistake is a token, not a stop.** `NsyTkFault` sits at the span
   that went wrong and lexing continues. `nsytoken.faults` collects them.
5. **The lexer emits tokens nobody typed.** NEWLINE, INDENT and DEDENT are what
   the language's layout is, and any assertion about the stream has to account
   for them. Language specification, section 2.
6. **`nsyparse.parse` never answers "no tree".** A failure becomes
   `NsyExError`, `NsyStError` or `NsyDcError` at the failing span.
   `nsyparse.is_clean` is the success test, and `parsed.faults` is the list.
7. **Recovery is at statement and declaration boundaries.**
   `nsyparse.is_recovery_point` is that policy as a predicate, so the rule that
   decides how much of a file one mistake costs can be read against the grammar.
8. **Every node carries a span, literals included.** A literal's spelling stays
   in the source and is reached through the node's span. That is what lets a
   printer put back `0x00000000` rather than `0x0`.
9. **A compound assignment keeps the operator the author wrote.** `a += b` is
   not expanded into `a = a + b` in this tree. A printer over an expanded tree
   would rewrite the file.
10. **`pub` is a field on a declaration, not an annotation.**
    `nsyast.decl_is_pub` reads it.
11. **The printer parenthesises by comparing precedence levels.** The tree
    records no parentheses, so `nsyast.binop_level` is the cascade as a number
    and the printer compares them. The criterion the printer is held to is that
    `parse(print(t))` equals `t`, node for node.
12. **A literal is printed from its span when the spelling still decodes to the
    value in the tree, and from the value otherwise.** A node a pass
    synthesised has no spelling to read back.
13. **The signature table is an argument, not something this package ships.**
    `nsycheck.table_of_tree` builds the half that comes from parsed novo-lang
    source. A caller with a compiler in the same process fills the rest.
    `nsycheck.empty_table()` resolves everything local and reports the rest as
    unknown rather than guessing.
14. **`nsyeffect.rows_of` answers a whole file's rows at once.** An effect row
    is a fixed point over the call graph, because recursion exists, so computing
    one row means computing the component it sits in. Language specification,
    section 5.5.
15. **`nsyquery.place_at` reads tokens and every other query reads the tree.**
    The offset a completion fires at is in a file that does not parse, and an
    error node cannot say whether a type or an expression was being written.
16. **The walkers have no wildcard branch.** Adding a node kind to `nsyast` is a
    compile error in `nsyvisit` and nowhere else, so no pass can quietly answer
    for a smaller program than it was given.

## What is not included

- **Type checking.** Inference, trait selection, monomorphisation and the
  error-type bound belong to the compiler. A package that did half of them would
  be a second authority saying something slightly different from `novo build`.
- **Error codes.** This package mints no `E1002`-style identifiers. The compiler
  owns that namespace.
- **A file reader.** A caller that has a path reads it. There is deliberately no
  `lex_from<S: Read[e]>` entry point either: a span is a byte range into source
  the caller holds, so a caller that streamed bytes past this package and did
  not keep them could not resolve a single token it got back.
- **A standard-library signature table.** See rule 13. A table baked in here
  would pin the package to one compiler release and answer "no such function"
  for everything published since.
- **Dependencies.** The lexer is a hand-written character machine, not a pattern
  set, so no regular-expression engine is needed.
- **Running on a microcontroller.** A parser allocates a node per production, a
  list per body and a string per identifier, and `str.builder`, which the
  printer writes through, is not available on a device with no heap allocator.
- **The wire formats the compiler's own dump commands emit.** Those stay in the
  differential harness this package was cut from.

## Related packages

- `std.str` in the standard library holds the `str.builder` handle every
  `nsyprint` entry point appends into.
- [sarif-nv](https://novo-lang.org/packages/sarif-nv) turns diagnostics into the
  interchange format a code-scanning service reads. A pass built on `nsyvisit`
  that finds something writes its findings through it.
- [table-nv](https://novo-lang.org/packages/table-nv) draws a table for a
  terminal, which is what a tool printing an outline or a row report wants.

## Tests

```bash
novo test tests                            # every suite
novo test tests/nsytoken_tests.nv          # spans, the keyword and punctuation tables, trivia
novo test tests/nsylex_tests.nv            # the layout contract, and the lexer as a value
novo test tests/nsyparse_tests.nv          # a tree from every file, and the faults beside it
novo test tests/nsyprint_tests.nv          # round-trip identity, idempotence, byte stability
novo test tests/nsyquery_tests.nv          # resolution, effect rows, the visitor, the queries
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented`
panic, because every body is a `todo()`. The tests are the specification the
implementation will have to satisfy.

The vectors are the reference front end's own. The reserved-word table is the
compiler lexer's keyword table word for word, and the punctuation table is its
operator rules in longest-match order. The layout assertions are the ones the
differential harness already makes against the compiler, restated over this
package's stream with the trivia filtered out. A spelling that drifts from the
compiler's is a failing assertion here rather than a corpus divergence somebody
has to bisect.

The printer suite asserts three properties, in order of how much they claim.
`parse(print(t))` equals `t`, so the printer does not change the program.
`print(parse(print(s)))` equals `print(s)`, so printing is idempotent. And
`print(parse(s))` equals `s` for canonical `s`, which is byte stability, and is
the strongest of the three.

## Implementation status

| Item | Implemented |
| --- | --- |
| `nsytoken`'s four types and eight enumerations | declared |
| `nsytoken`'s 27 functions, from `source` to `str_value` | no |
| `nsylex.NsyLexer` | declared |
| `nsylex`'s 14 functions, from `new` to `splice` | no |
| `nsyast`'s 54 records and 17 enumerations | declared |
| `nsyast`'s 14 functions, from `effect_of` to `decl_kind_name` | no |
| `nsyparse`'s four types and `NsyParseKind` | declared |
| `nsyparse`'s 11 functions, from `parse` to `first_fault` | no |
| `nsyvisit.NsyNameRole`, `.NsyVisitor` and `.NsyRewriter` | declared |
| `nsyvisit`'s 14 walk, rewrite and children functions | no |
| `nsyprint.NsyStyle` | declared |
| `nsyprint`'s 15 functions, from `canonical` to `write_tree` | no |
| `nsycheck`'s six records and two enumerations | declared |
| `nsycheck`'s 12 functions, from `empty_table` to `fault_message` | no |
| `nsyeffect`'s three records and `NsyEffKind` | declared |
| `nsyeffect`'s 11 functions, from `rows_of` to `fault_message` | no |
| `nsyquery`'s three records and two enumerations | declared |
| `nsyquery`'s 19 functions, from `node_span` to `doc_examples` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
