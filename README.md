# swc_ecma_parser (Sentry fork)

A minimal fork of [`swc_ecma_parser`](https://github.com/swc-project/swc) 27.0.7
that bounds parser recursion, so that hostile JavaScript can neither abort the
process it is parsed in nor exhaust its memory.

Consumed by `symbolicator` through `symbolic-sourcemapcache` →
`js-source-scopes`, which parses attacker-supplied minified sources to recover
scope names.

## Why

`swc_ecma_parser` parses expressions — and nested statements — with mutual
recursion whose depth is driven entirely by the input. The crate's only
`stacker::maybe_grow` call site covers the consequent of an `if` statement, so
everything else is bounded only by the thread stack, and a few kilobytes of
input exhausts it. In Rust a stack overflow is a `SIGSEGV`/abort rather than an
unwind, so it cannot be caught with `catch_unwind` and takes the whole process
down.

On a 2 MiB stack (a Tokio worker's default) the stock crate aborts on, among
others:

| construct | aborts at depth | input size |
|---|---|---|
| `new(()=>` … `)` | 297 | ~2.4 KB |
| `({a:` … `})` | 357 | ~1.4 KB |
| `(` … `)` | 625 | ~1.3 KB |
| `{` … `}` (nested blocks) | < 5,000 | ~10 KB |
| `!` … / `--` … / `typeof` … | ~4,500 | ~4.5 KB |
| `1?` … `:0` / `1?0:` … | ~8,760 | ~35 KB |

### Why `maybe_grow` alone is not the fix

`maybe_grow` does not enlarge the thread stack: it allocates a fresh segment on
the **heap** and moves the stack onto it. Depth then becomes bounded by memory
rather than by the stack, which converts the abort into unbounded memory
growth. Measured with `maybe_grow` alone on these paths:

| depth | input | peak RSS |
|---|---|---|
| 10,000 | 20 KB | 36 MB |
| 100,000 | 195 KB | 334 MB |
| 400,000 | 781 KB | **1,329 MB**, in 0.22 s |

That is ~3.3 KB of RSS per nesting level, an amplification of roughly **1700x**
from input bytes — a cheaper and more reliable denial of service than the crash
it replaces. Growing the stack is only safe once something else bounds the
depth.

## What changed

A recursion counter on the parser, checked at the four entry points of the
recursion. Past `MAX_RECURSION_DEPTH` the parser reports
`SyntaxError::TooMuchRecursion`, an ordinary syntax error that callers can
recover from. `maybe_grow` runs alongside it so the permitted depth is
comfortably reachable on a small stack.

| guard | covers |
|---|---|
| `parse_assignment_expr` | brackets, ternaries, binary and comma chains, arrow bodies |
| `parse_unary_expr` | prefix chains (`!`, `~`, `-`, `typeof`, `void`, `delete`, `await`, `++`, `--`) |
| `parse_member_expr_or_new_expr_inner` | `new` chains |
| `parse_stmt_internal` | nested blocks and function bodies |

74 lines across four files. `git show f950926` is the whole patch;
`0001-*.patch` is the same thing in a form that can be reapplied to another
base.

`MAX_RECURSION_DEPTH` is **2048**. The deepest of the JavaScript fixtures in
`getsentry/symbolic` is `jquery.js` at 54 (`jquery.min.js` 52, a Metro bundle
38), so this leaves roughly 38x headroom.

## Verification

- **Behaviour-preserving.** ASTs are identical to the stock crate's for all 27
  JavaScript fixtures in `getsentry/symbolic` (compared as `{:?}` of the parsed
  `Program`).
- **Bounded.** A 781 KB nested payload peaks at 6 MB of RSS and a graceful
  parse error — against 1,329 MB with `maybe_grow` alone, and an abort on
  stock. Every construct in the table above now fails gracefully instead of
  aborting.
- **No performance cost.** `jquery.min.js` parses in 9.2 ms vs 9.7 ms stock
  over 40 iterations; `preact.module.js` unchanged at 1.1 ms.
- **Sequence expressions are unaffected.** swc parses `a,a,a,…` iteratively
  (depth 4 regardless of length), so a 2 MB comma chain costs 91 MB on both
  stock and this build and is not rejected.

Very deep input may surface as either an `Err` or as `Ok` carrying recovered
errors in `take_errors()`, depending on where swc's existing error recovery
catches it. Both are bounded and neither aborts.

## Known remaining limit

Dropping a deeply nested AST is also recursive, and overflows on its own at a
depth of roughly 6,000 (arrow functions) to 22,000 (parentheses). The 2048
limit keeps ASTs well clear of that, but raising `MAX_RECURSION_DEPTH`
substantially means re-checking it.

Note also that `js-source-scopes` walks the resulting AST with `swc_ecma_visit`,
which recurses without a guard of its own. The depth limit keeps that walk
shallow in practice, but it is not fixed by this fork.

## Why a standalone fork rather than a fork of the monorepo

This repository holds the contents of the *published* crate, whose manifest
depends on `swc_common`, `swc_ecma_ast` and friends by version. Those therefore
continue to resolve from crates.io, and exactly one copy of each ends up in the
build — the same copies `js-source-scopes` was compiled against.

Forking `swc-project/swc` itself would instead bring the whole `swc_*` family
in as path dependencies from the fork. Cargo treats those as distinct from the
crates.io copies even at identical versions, so every one of them would need
its own `[patch.crates-io]` entry, and any that was missed would surface as a
type mismatch at the `js-source-scopes` boundary.

The tradeoff is that this fork cannot be rebased onto upstream or turned
directly into a pull request. Carry the change upstream from a monorepo
checkout using `0001-*.patch`.

## Using it

In the **consuming binary's** root manifest — `symbolicator`'s, not
`symbolic`'s, since `[patch]` applies only to the workspace being built and
does not reach downstream consumers of a published library:

```toml
[patch.crates-io]
swc_ecma_parser = { git = "https://github.com/getsentry/swc_ecma_parser-sentry", tag = "v27.0.7-sentry.1" }
```

The version stays `27.0.7`, so it satisfies `js-source-scopes`'s `^27`
requirement and no other manifest needs to change. Confirm it took effect with:

```
cargo tree -i swc_ecma_parser
```

which should name the git source rather than `registry+https://...`.

## Upgrading

`js-source-scopes` pins the `swc_ecma_parser` major, so this fork has to track
it. When that dependency moves to a new major, re-import the published crate at
the new version as a first commit and reapply `0001-*.patch` on top. The
guarded functions have been stable across recent versions; note that `47.0.0`
was inspected and carries the same single `maybe_grow` call site as 27.0.7, but
the patch has not been built or tested against it.

Retire the fork once upstream bounds these paths itself.
