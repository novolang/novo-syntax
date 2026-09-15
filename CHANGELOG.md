# Changelog

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1

- The interface, published before anything is implemented: nine
  modules, every `pub` body a `todo()`, the signatures and the effect
  rows checked by the compiler as claims.
- Cut from `orbit/novo-front`, which stays in orbit as the differential
  harness against the compiler's own front end.

### Design notes

- **What stayed in `orbit/novo-front`.** That package is the
  differential harness against the compiler's own front end and is not
  replaced by this one. It keeps `main.nv` and its `--tokens` /
  `--ast` / `--types` / `--effects` modes; the wire-format rendering in
  `token.nv` and `ast.nv`, which is fixed byte for byte by the
  compiler's own dump so the two can be diffed; `check.nv` and
  `effects.nv` in full, with the three generated tables they read
  (`stdlib_table.nv`, `stdlib_effects.nv`, `prelude_table.nv`, 4,700
  lines dumped out of one compiler build, right for a harness and
  wrong for a package); and `lexer_tests.nv` and `SUBSET.md`.
- **The tree is typed, and novo-front's is not.** novo-front's
  `ast.nv` is a homogeneous `Node` — a line of text and the nodes
  indented under it — because what that program produces is a dump.
  Nothing there type-checks the shapes. `nsyast` is derived from
  `compiler/lib/ast.ml` instead, and one homogeneous node becomes six
  category pairs. `impl_target_key` is the one function that crosses
  unchanged, over `NsyType` instead of over a `Node`, with the same
  rule: one function computes the key so a definition site and a use
  site cannot disagree, and `[T; N]` keys as `[T]` because the length
  erases.
- **The seven position fields that became one rule.** The reference
  tree positions expressions, statements and declarations and leaves
  types, patterns and members bare, then carries `sf_pos`, `ev_pos`,
  `fs_pos`, `rg_addr_pos`, `mr_base_pos`, `mr_size_pos`, `ms_pad_pos`,
  `vt_addr_pos` and `bsp_member_lines`, each added after a formatter
  bug. A span on every node states that fact once.
- **Four divergences from the reference, each deliberate.** `pub` is a
  field rather than a synthetic annotation named `"pub"`, because a
  formatter that printed the annotation list verbatim wrote `@pub` on
  a line of its own and the formatted file stopped parsing. `a += b`
  is not expanded at parse time. Resolution is not type checking. No
  error codes are minted.
- **The consumers this interface was shaped for.** `novols` takes
  `nsyparse.parse`, `nsyquery.path_at`, `place_at`, `outline`,
  `nsycheck.resolve` and `rename_spans`, replacing a hand-built symbol
  index over the compiler's AST. `novo fmt` takes
  `nsyprint.print_tree`, `is_canonical` and `first_difference`,
  replacing its own comment-recovery scanner, its own parenthesisation
  table and two raw-literal accessors. `orbit/novo-treesitter` takes
  `nsylex.lex` plus `nsytoken.kind_name`, replacing a corpus
  comparison that shells out to `novo parse --tokens`.
- **Something has to own the style, and nothing does.** `nsyprint`
  publishes the printer's contract and one canonical `NsyStyle`, but
  what canonical is — four spaces, where a wrapped signature breaks,
  how many blank lines survive between declarations — lives in
  `compiler/bin/novo.ml` and is written down nowhere a package can be
  held to. Until `docs/tooling.md` has a `novo fmt` specification,
  `nsyprint.is_canonical` is checkable against the current `novo fmt`
  and against nothing else.
