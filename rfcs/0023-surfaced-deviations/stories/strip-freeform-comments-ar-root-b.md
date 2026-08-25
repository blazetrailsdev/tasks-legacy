---
title: "strip-freeform-comments-ar-root-b"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Follow-on slice of `strip-freeform-comments-ar-root`, which swept
`packages/activerecord/src/a*.ts` (219 violations, ~470 LOC of deletions) and
landed `"packages/activerecord/src/a*.ts"` in the `no-freeform-comments` block's
`files` list in `eslint.config.mjs:739-750`.

`blazetrails/no-freeform-comments` is registered per glob, so the root
directory is being swept one alphabetical slice per PR and stays green in
between. The remaining root files measure **1510 violations** across ~197
files, so this is several more PRs — file the next slice as you go rather than
widening this story.

Measured violation counts by first letter (run
`pnpm eslint "packages/activerecord/src/*.ts" -f json` with the glob temporarily
added to the block):

```text
b 172   c 180   d  87   e  52   f  84   h  10   i  66   j   9
l   9   m 208   n  40   o   1   p  60   q  33   r 138   s 143
t 202   u   2   v  12   w   2
```

At ~2.1 deleted lines per violation, a 700-LOC PR is roughly 300 violations, so
`b*.ts` (172, dominated by `base.ts` at 121) is a sensible next slice on its
own; `c*.ts` (180) the one after.

The bar (from the arel/activemodel pass, slice 1, and the `a*` slice): a comment
that restates the line or branch it sits on goes, whatever its subject —
including one narrating a TypeScript deviation. What survives, survives as JSDoc
carrying a tag or a Rails citation. Rails' OWN comments are deleted too (the Ruby
is vendored and cited). A comment recording deferred work or a known-divergent
shape becomes a story, not a better comment.

Note from the `a*` slice: the autofix can empty a block — e.g.
`associations.ts:838` was `if (ownWrappedNames.has(name)) { /* comment */ } else {`
and had to be re-expressed in code as `if (!ownWrappedNames.has(name)) {`, not by
restoring the comment.

## Acceptance criteria

- [ ] `packages/activerecord/src/b*.ts` added to the `no-freeform-comments`
      block's `files` in `eslint.config.mjs` (extend the alphabetical slice
      comment already there).
- [ ] `pnpm eslint --fix` applied over that glob and the deletions reviewed
      rather than taken on trust.
- [ ] `pnpm eslint` clean over the glob, and a second `--fix` run is a no-op.
- [ ] `pnpm prettier --write` over the glob (deletions leave double blank lines).
- [ ] `pnpm typecheck` clean; the test files touched run green.
- [ ] Deletions that empty a block (`catch {}`, an `if`/`else` arm) are expressed
      in code, not by restoring the comment.
- [ ] Any deferred work or known deviation found in a deleted comment is filed
      as its own story with the trails/Rails `file:line`.
- [ ] The next slice (`c*.ts`) filed as its own story.
