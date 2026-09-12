# novo-syntax

**Status: NOT IMPLEMENTED — interface only.**

novo-lang's own front end, as a library: the indentation-sensitive
lexer, the recursive-descent parser, the typed AST with a span on every
node, a visitor, a canonical printer, the resolution and effect-row
tables, and the queries an editor asks. Every `pub` body is a
`todo()`; the signatures and the effect rows are the design, and the
compiler checks them as claims.

```novo
use nsytoken
use nsylex
use nsyparse
use nsyprint

fn main() [io]
    let src = nsytoken.source("fn main()\n    pass\n")
    let toks = nsylex.lex(src)
    let parsed = nsyparse.parse_tokens(src, toks)
    // `parse` always answers a tree; `is_clean` is the success test.
    if nsyparse.is_clean(parsed)
        println(nsyprint.to_source(src, toks, parsed.tree, nsyprint.canonical()))
```

## Build, run and test

```
novo pkg add novo-syntax          # add it to a project
novo pkg build                    # type-check and effect-check it
novo doc .                        # write APIDOC.md and compile the examples
novo test --llvm tests/nsytoken_tests.nv    # one API suite
```

There is no `novo run`: this is a library with no `main`. Every suite
under `tests/` is **red** today, and deliberately — see § Tests.

## The layer, and why

`core`. No effects at all, and for a front end that is the whole design
rather than a label.

A lexer's natural API opens a path (`Lexer.from_file`), and a language
server's natural API watches a directory. Neither is here. Source
arrives as a `Str` the caller already holds, a span is a byte range into
that same `Str`, and every answer is a value. What that buys is not
only the budget: the same parser runs inside `novols`, inside `novo
fmt`, inside a wasm playground, and inside a test that holds one line of
source in a string literal — four deployments that a `parse_file` API
serves in one.

`@tier(embedded)` is **not** claimed, and the reason is stated rather
than implied: a parser allocates constantly — a node per production, a
list per body, a `Str` per identifier — and `str.builder`, which
`nsyprint` writes through, is not available at that tier. A device that
wanted novo-lang source on it would want a different package, not this
one with a claim it could not keep.

## Why no stream flows through

`docs/publishing.md` § How a `core` package takes a stream from its host
names four shapes, and this package uses two of them and deliberately
refuses the third.

**Feed and drain** is `nsylex`: `feed(lexer, chunk)` answers a new lexer
and the tokens that chunk completed, `finish` closes the file. A caller
may split the source anywhere, including inside a string literal.

**The bound effect parameter** appears exactly twice, and both are
about giving something to the caller rather than taking something from
it: `nsyvisit.walk_tree<V: NsyVisitor[e]>` is charged what the
visitor costs, and `nsyprint.write_tree<W: Write[e]>` is charged what
the sink costs.

What is **not** here is `lex_from<S: Read[e]>(src: S)`. It would type,
it would pass the audit, and it would be a trap: a span is a byte range
into source the caller holds, so a caller that streamed the bytes past
this package and did not keep them could not resolve a single token it
got back. The honest API is the one that makes the caller hold the
text, so that is the only one offered.

## The load-bearing interfaces

Three, and they are in the three modules everything else is downstream
of.

**`NsySource`, and the span as a byte range.** A token carries no text
— `NsySpan { start, stop }` and nothing else — because the source is
where the bytes already are. A span alone is half an answer (`start:
4021` is what a lexer can afford to carry, `line 118, column 6` is what
a person reads), so `NsySource` pairs the caller's text with its line
table, built once, and every entry point takes one. That is what makes
lexing a 100 KB module one list of small records rather than one `Str`
per identifier, and it is what lets the formatter read back a spelling
the tree does not carry.

**Trivia are tokens, and so are faults.** Comments and blank-line runs
are in the stream, in source order, with spans — not skipped. The
reference front end drops them in the lexer, and the consequence is
visible in the compiler today: `novo fmt` re-scans the raw source with
its own comment scanner and re-interleaves what it finds around a
printed tree, and a version of that scanner which knew `//` but not `#`
or `/* … */` deleted every comment it did not know, with no diagnostic
and exit 0. A formatter over this stream cannot have that bug, because
there is nothing to re-scan. Lexical faults are tokens for the same
family of reason: a lexer that stopped at the first bad escape would
hand a language server one diagnostic and no tokens for the rest of the
file, which is the state a file is in for most of the time anyone is
looking at it.

**A span on every node, and `nsyparse` never answering "no tree".**
`NsyExpr` is `{ kind: NsyExprKind, span }` — the span is a fact about
the pair, not a field sixty variants each have to remember. The
reference positions expressions, statements and declarations and leaves
types, patterns and members bare, and then carries seven more position
fields that were each added after a formatter bug (see the table
below). Making the span total states that once. And `parse` answers
`NsyParseResult`, not `Result`: a failure becomes an `NsyExError`,
`NsyStError` or `NsyDcError` node at the span that failed, the parser
resynchronises at a statement boundary, and the faults come back beside
the tree. `nsyparse.is_recovery_point` is that policy as a public
predicate, so the rule that decides how much of a file one typo costs
is a function a reader can check against the grammar rather than a loop
condition buried where it can only be inferred from the damage.

## The reference implementation

Two, and they are the same front end written twice.

- `compiler/lib/lexer.mll`, `compiler/lib/parser.mly` and
  `compiler/lib/ast.ml` are the authority: `SPEC.md` is the grammar and
  those three are what the compiler actually runs.
- `orbit/novo-front` is the same front end written in novo-lang, with a
  differential harness that diffs its output against the compiler's own
  (`novo parse --tokens` / `--ast` / `--types` / `--effects`) over every
  `.nv` file in the repository. **This package is cut out of it.**

## The modules

| Module | What it is | `pub fn` | `pub struct` | `pub enum` | `pub trait` |
| --- | --- | ---: | ---: | ---: | ---: |
| `nsytoken` | spans, the line table, the reserved and punctuation tables, trivia, the token | 27 | 4 | 8 | 0 |
| `nsylex` | the indentation-sensitive lexer as a value: feed, drain, finish | 14 | 1 | 0 | 0 |
| `nsyast` | the typed tree, with a span on every node | 14 | 54 | 17 | 0 |
| `nsyparse` | tokens to a tree, and the faults beside it | 11 | 4 | 1 | 0 |
| `nsyvisit` | the visitor and the rewriter, as traits | 14 | 0 | 1 | 2 |
| `nsyprint` | the tree back to canonical source, into a caller's buffer | 15 | 1 | 0 | 0 |
| `nsycheck` | what every name refers to, over a table the caller supplies | 12 | 6 | 2 | 0 |
| `nsyeffect` | the effect row every function carries | 11 | 3 | 1 | 0 |
| `nsyquery` | the path at an offset, the place at an offset, the outline | 19 | 3 | 2 | 0 |
| **total** | | **137** | **76** | **32** | **2** |

## The consumers

| Consumer | What it takes | What it replaces today |
| --- | --- | --- |
| `novols` (`compiler/lsp/`) | `nsyparse.parse`, `nsyquery.path_at`, `place_at`, `outline`, `nsycheck.resolve`, `rename_spans` | a hand-built symbol index over `Novo_lib.Ast.decl list`, and `Lexer.reset` / `Parser.program` driven directly |
| `novo fmt` (in `compiler/bin/novo.ml`) | `nsyprint.print_tree`, `is_canonical`, `first_difference` | its own `fmt_comment` recovery scanner, its own parenthesisation table, and `Lexer.int_raw_at` / `string_raw_at` for literal spellings |
| `orbit/novo-treesitter` | `nsylex.lex` plus `nsytoken.kind_name` | a corpus comparison that shells out to `novo parse --tokens` |
| `orbit/novo-front` | nothing — it keeps its own copy | — |

## The split: what moved, what stayed

`orbit/novo-front` **stays where it is and keeps doing its job.** It is
the differential harness — `tests/orbit/test_novo_front.sh` runs it
against the compiler in five shards, over `examples/`,
`compiler/stdlib/`, `orbit/` and its own boundary corpus, in token,
token-position, AST, AST-position, type and effect modes. Nothing in
this package replaces that, and nothing in it is a dependency of it.
The harness keeps:

- `main.nv` — the CLI the shards drive, and its `--tokens` / `--ast` /
  `--types` / `--effects` modes;
- the wire-format rendering in `token.nv` and `ast.nv` — `tok_name`,
  `render_str`, `tok_escape`, `pos_prefix`, `render_tree`,
  `render_tree_pos`, `Render for Tok` — which is fixed byte for byte by
  `compiler/bin/novo.ml`'s `tok_dump_line` and `Ast_dump`, and exists
  only so the two can be diffed;
- `check.nv` and `effects.nv` in full, and the three generated tables
  they read — `stdlib_table.nv`, `stdlib_effects.nv`,
  `prelude_table.nv` — which are dumped out of ONE build of the
  compiler and are right for a harness and wrong for a package (see
  `nsycheck`'s header);
- `lexer_tests.nv` and `SUBSET.md`.

### `ast.nv` (16 items)

novo-front's `ast.nv` is **not a typed AST**. It is a homogeneous
`Node` — a line of text and the nodes indented under it — because what
that program produces is a DUMP, and a dump is a homogeneous tree. Its
own header says so, and names the cost: nothing there type-checks the
shapes, and the differential harness is the only thing that catches a
`parse_type` that returned an expression node.

A library cannot make that trade. So `nsyast` is a typed tree derived
from `compiler/lib/ast.ml`, and this is the one place the package's
shape is not novo-front's.

| novo-front (`ast.nv`) | novo-syntax | Note |
| --- | --- | --- |
| `struct Node` | `NsyExpr`, `NsyStmt`, `NsyDecl`, `NsyPat`, `NsyType`, `NsyLit` and their `…Kind` enums | one homogeneous node becomes six category pairs |
| `nd`, `leaf`, `ndp`, `leafp` | — | constructors for the dump node; a typed tree has literal constructors |
| `with_pos` | — | re-stamping a node's position is a dump-shape repair; a typed node's span comes from its own production |
| `pos_prefix`, `render_into`, `render_tree`, `render_tree_pos` | **stays in novo-front** | the `--ast` wire format |
| `b01`, `join_names`, `opt_name` | — | the three scalar encodings a TEXTUAL header needs (`1`/`0`, `a,b,c`, `-`); a typed field needs none |
| `head_word_of`, `kid0` | — | reading a field back out of a header string; a typed node has fields |
| `impl_target_key` | `nsyast.impl_target_key` | **the one function that crosses unchanged.** Over `NsyType` instead of over a `Node`, with the same rule and the same reason: one function computes the key, so a definition site and a use site cannot disagree, and `[T; N]` keys as `[T]` because the length erases |

### `token.nv` (12 items)

Not in the brief's list, but the split runs through it, so it is
accounted for here.

| novo-front (`token.nv`) | novo-syntax | Note |
| --- | --- | --- |
| `enum Tok` | `NsyTokKind` | `TPunct(name: Str)` / `TKeyword(name: Str)` become `NsyTkPunct(NsyPunct)` / `NsyTkKeyword(NsyKeyword)` — closed enums, because a consumer asking "is this `fn`?" should not be doing a string comparison |
| `struct Pos`, `struct PTok`, `PTok.raw` | `NsySpan`, `NsyToken`, `NsySource` | a position and a spelling both become the span; `raw` disappears, because the source IS the spelling |
| `enum StrSeg` | `NsyStrSeg` | `SegLit(text)` becomes `NsySegLit(span)` — no decoded text — and `SegHole(body, bline, bcol)` becomes `NsySegHole(span, tokens)`: the hole's token run, already lexed at file offsets |
| `tok_name` | `nsytoken.kind_name` | same names, so a corpus check needs no translation table |
| `pos_prefix`, `tok_escape`, `escape_byte`, `hex_digit`, `render_str`, `trait Render`, `impl Render for Tok` | **stays in novo-front** | the `--tokens` wire format |

### `lexer.nv` (69 items)

| novo-front (`lexer.nv`) | novo-syntax | Note |
| --- | --- | --- |
| `lex`, `lex_seeded` | `nsylex.lex`, `nsylex.seeded` + `feed`/`finish` | the whole-file call stays; the seeded one becomes a constructor, because the state between two feeds is what a language server and a read-eval loop ask questions of |
| `struct LexOut` (`toks`, `positions`, `raws`, `err`, `err_line`, `err_col`) | `[NsyToken]` | four parallel arrays and a single first-error become one list: the position is the token's span, the spelling is its span, and an error is `NsyTkFault` IN the stream, so there can be more than one |
| `keyword_name`, `word_token` | `nsytoken.keyword_of`, `nsytoken.keyword_text` | `Str -> Str` becomes `Str -> ?NsyKeyword`; the after-a-dot rule moves into `nsylex`, where the state that decides it lives |
| `op_token`, `op_at`, `struct OpHit`, `no_op` | `NsyPunct` + `nsytoken.punct_text` | the longest-match table becomes an enum and its spellings |
| `cont_op_at`, `cont_op_ignores_line_opener` | `nsytoken.punct_continues_line` | the continuation table, as a predicate |
| `is_block_opener` | `nsytoken.keyword_opens_block` | same set, same caveat: the kind alone is not the whole test |
| `struct IndentStep`, `process_indent`, `top_of`, `drop_deeper` | `NsyLexer.indent`, `NsyLexer.brackets`, `bracket_indent`, `match_wait` | the indentation machinery becomes fields on the lexer value, named for what they do |
| `struct StrHit`, `scan_string`, `scan_multiline`, `flush_run`, `ml_strip_width`, `ml_dedent` | `nsylex.feed` + `NsyTkStr(NsyStrForm, [NsyStrSeg])` | the three string forms become a form tag on one token kind |
| `struct HoleHit`, `scan_hole`, `struct NestHit`, `copy_nested_str` | `NsySegHole(span, tokens)` | a hole's body was a raw `Str` plus a seed position; it is now the token run, lexed by `nsylex.seeded` |
| `struct NumHit`, `scan_number`, `scan_decimal`, `scan_radix`, `float_len`, `hex_len`, `bin_len`, `dec_len`, `exp_end`, `int63_max`, `acc_limit` | `NsyTkInt(value, NsyRadix, digits)`, `NsyTkIntMin`, `NsyTkFloat` | the scanners are private to the implementation; the radix and the digit span are what survive into the surface, because the printer needs them |
| `struct UniHit`, `struct UniVal`, `scan_uni_brace`, `unicode_escape_value`, `append_utf8`, `escape_known`, `unescape`, `escape_desc` | `nsytoken.unescape_into`, `nsytoken.escape_into` | decoding is on demand and into a caller's buffer, because most consumers never look at a literal's value |
| `struct CharHit`, `scan_char`, `char_lit_len` | `NsyStrChar` | a character literal is a string FORM; the parser turns it into an integer and `nsyprint` puts the quotes back from the span |
| `struct SkipHit`, `skip_to_eol`, `skip_block_comment`, `comment_line_end` | `NsyTkTrivia(NsyTrivia)` | comments were skipped; they are now tokens |
| `unknown_escape_msg`, `incomplete_u_msg`, `bad_hex_u_msg`, `above_max_u_msg`, `surrogate_u_msg`, `char_u_not_ascii_msg`, `backslash_in_hole_msg`, `malformed_char_msg`, `unexpected_char_msg` | `NsyFaultKind` + `nsytoken.fault_message` | nine message builders become one enum and one renderer, so a consumer can match on the fault rather than on its English |
| `is_digit`, `is_letter`, `is_hex_digit`, `is_ident_char`, `at`, `hex_value`, `ident_end`, `word_continues` | — | character classification, private to the implementation |

### `parser.nv` (119 items)

Every `parse_*` function is one production of
`compiler/lib/parser.mly`'s cascade, and they stay one production each
— they are simply private, because a caller parses a FILE, an
EXPRESSION or a TYPE and never a `parse_bxor`.

| novo-front (`parser.nv`) | novo-syntax | Note |
| --- | --- | --- |
| `parse_program` | `nsyparse.parse`, `nsyparse.parse_tokens` | the two entry points a caller has: with the lex, and over a stream already lexed |
| `parse_expr` (and the cascade under it: `parse_pipe`, `parse_nullcoal`, `parse_or`, `parse_and`, `parse_not`, `parse_cmp`, `parse_range`, `parse_bor`, `parse_bxor`, `parse_band`, `parse_shift`, `parse_add`, `parse_mul`, `parse_unary`, `parse_cast`, `parse_postfix`, `parse_call`, `parse_primary`, `parse_inline_if`, `parse_inline_if_tail`, `parse_paren_lambda`, `parse_lambda_body`, `parse_struct_literal`, `parse_list_comp`, `parse_rhs`) | `nsyparse.parse_expr` public, the cascade private; the LEVELS are public as `nsyast.binop_level` | the printer is the second reader of the cascade and cannot guess it, so the numbers are surface and the functions are not |
| `parse_type`, `expect_gt`, `parse_effect_names` | `nsyparse.parse_type`; `NsyEffects` | `>>` re-opening is private; an effect clause becomes a type carrying `written`, which is the distinction `fn go() []` makes and `fn go()` does not |
| `parse_pattern`, `parse_or_pattern`, `parse_pattern_base`, `parse_list_pattern`, `parse_ident_pattern`, `parse_enum_pattern_args`, `parse_struct_pattern`, `finish_cons_pattern` | `NsyPat` / `NsyPatKind` | private productions; the tree is the surface |
| `parse_stmt`, `parse_let`, `parse_binding`, `parse_for`, `parse_block`, `body_node`, `parse_if_expr`, `parse_elif_arm`, `parse_block_if_tail`, `parse_block_match`, `parse_block_loop`, `parse_match_arm` | `NsyStmt` / `NsyStmtKind`, `NsyIf`, `NsyMatch`, `NsyMatchArm` | the same; `body_node`, a synthetic grouping node the DUMP needed, has no counterpart in a typed tree |
| `parse_decl`, `parse_fn`, `parse_const`, `parse_alias`, `parse_use`, `parse_struct`, `parse_struct_field`, `parse_enum`, `parse_enum_variant`, `parse_trait`, `parse_trait_member`, `parse_impl`, `finish_impl`, `parse_signature`, `struct SigParts`, `parse_params`, `parse_param`, `parse_tparams`, `take_bound_name`, `take_bound_with_effect` | `NsyDecl` / `NsyDeclKind` and the 19 declaration records | `SigParts` becomes `NsyFnSig`; `take_bound_with_effect`'s answer becomes `NsyTypeParam.binds_effect`, one record per parameter instead of a third parallel list |
| `parse_annotation`, `parse_ann_arg`, `parse_member_annot_arg`, `past_member_annots`, `is_struct_field` | `NsyAnnot`, `NsyAnnArg`; `nsyast.find_annot` | lookahead is private; the annotation is surface |
| `parse_hole`, `try_parse_expr`, `struct TryRes`, `string_expr`, `flat_string`, `find_top_colon` | `nsyparse.parse_hole`, `NsyExInterp([NsyInterpPart])` | `TryRes` (an expression plus a "did it work") is `NsyExprResult` (an expression plus a fault LIST); `find_top_colon`'s job — splitting `${e:spec}` — becomes `NsyIpExpr(value, format)` |
| `struct P`, `new_parser`, `new_parser_pos`, `pos_at`, `raw_at`, `here`, `stuck`, `cur`, `nxt`, `nxt2`, `tok_at`, `class_at`, `advance`, `accept`, `expect`, `group_end` | — | the parser's cursor, private. `stuck` — a flag that stops a failed parse from looping — becomes real recovery: `nsyparse.is_recovery_point` and `next_recovery_point` |
| `fail`, `unsupported` | `NsyParseFault`, `NsyParseKind`, `nsyparse.fault_message` | one accumulated message becomes a list of positioned faults, each with where the parser resumed |
| `tok_ident`, `tok_int`, `tok_segs`, `take_ident`, `take_binder`, `take_int`, `take_int_raw`, `float_of_tok`, `big_of_tok`, `is_upper_first` | — | payload extraction, private; the tree carries the values |
| `lit_int`, `lit_hex`, `lit_big`, `lit_float`, `lit_str`, `int_min_lit` | `NsyLit` / `NsyLitKind` | six header-string builders become one node with a span, which is what makes the SPELLING recoverable |
| `cmp_op_name`, `assign_op_name`, `compound_binop_name`, `expanded_assign` | `NsyBinOp`, `NsyAssignOp`, `nsyast.binop_text`, `assign_op_text` | `expanded_assign` — desugaring `a += b` into `a = a + b` at parse time — does **not** cross: `NsyStAssign` keeps the operator the author wrote, because a formatter that expanded it would rewrite the file |

### The seven positions that became one rule

`compiler/lib/ast.ml` carries these beyond its expression, statement
and declaration positions, each added after a formatter bug. In
`nsyast` they are the span every node already has.

| Reference field | What it was for |
| --- | --- |
| `sf_pos` | a struct member — "the one place the formatter could not thread a comment, because there was no line to key one to" |
| `ev_pos` | an enum variant, the same |
| `fs_pos` | a trait signature — a member with no line is one no comment can be keyed to, and the formatter hoisted the comment out of the trait body |
| `rg_addr_pos` | a register's address literal — `novo fmt` rendered it from its value, so `0x00000000` came back `0x0` and a register map stopped lining up against its datasheet |
| `mr_base_pos`, `mr_size_pos` | a memory region's base and size, for the same reason plus `512K` over `524288` |
| `ms_pad_pos` | a section's `pad_to` literal |
| `vt_addr_pos` | a vector table's address literal |
| `bsp_member_lines` | an association list of member name to line number, because a `bsp` declaration's members are flattened into scalars — without it a comment inside one was re-emitted at column 0 below the whole declaration |

## Where a row wanted to widen

Recorded here because the interface milestone asks for it, and because
each of these is a decision somebody should be able to argue with.

- **The tree is typed, and novo-front's is not.** The plan row says
  "the same shape novo-front's `ast.nv` has". It has no shape to copy:
  it is a dump transducer over one homogeneous `Node`. The shape comes
  from `compiler/lib/ast.ml` instead, and the divergences from THAT are
  the table above.
- **`pub` is a field, not a synthetic annotation.** The reference
  encodes it as an annotation named `"pub"` so that package visibility
  needed no tree change; the formatter that printed the annotation list
  verbatim wrote `@pub` on a line of its own and the formatted file
  stopped parsing. One boolean, and no split at each print site.
- **`a += b` is not expanded at parse time.** novo-front desugars it;
  this tree keeps the operator, because a formatter over an expanded
  tree rewrites the file.
- **Resolution, not type checking.** `nsycheck` binds names and
  indexes declarations. It does not infer types, select impls or
  monomorphise. A package that did half of that would be a second
  authority saying something slightly different from `novo build`,
  which is worse for a reader than saying nothing.
- **No error codes.** This package assigns no `E1002`-style
  identifiers. The compiler owns that namespace, and a package that
  minted its own would collide the first time both appeared in one
  diagnostic list.

## A row this package found missing

**`novo-fmt` is not on the grid, and something has to own the style.**
`nsyprint` publishes the printer's contract and one canonical
`NsyStyle`, but the decision about what canonical IS — four spaces,
where a wrapped signature breaks, how many blank lines survive between
declarations — lives in `compiler/bin/novo.ml` today and is not written
down anywhere a package can be held to. Either `docs/tooling.md` §
`novo fmt` becomes that specification, or the style belongs in a row of
its own. Until one of those, `nsyprint.is_canonical` is checkable
against the current `novo fmt` and against nothing else.

## Dependencies

None, and the reason is the layer. The two things a front end usually
reaches for are a file reader and a regular-expression engine; this
package has no file to read, and its lexer is the reference's own
hand-written character machine rather than a pattern set. The stdlib
signature table `nsycheck` and `nsyeffect` resolve against is an
ARGUMENT the caller supplies, not a generated module in here, so the
package does not depend on a compiler release either.

## Tests

`tests/` holds five API suites written against the signatures. They are
**red** today — every function they call is a `todo()` — and that is the
interface-package contract: the assertions say exactly what the
implementation has to answer, and the vectors are the reference's own,
so a spelling that drifts from the compiler's is a red assertion rather
than a corpus divergence somebody has to bisect.

## Licence

Apache-2.0.
