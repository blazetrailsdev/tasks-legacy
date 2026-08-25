---
rfc: "0103-parity-api-scoring-correctness"
title: "parity:api scoring correctness residue"
status: closed
created: 2026-08-12
updated: 2026-08-25
owner: "@deanmarano"
packages: []
clusters: []
priority: 2
---

# parity:api scoring correctness residue

## Problem

Two correctness bugs in the compare tooling outlived RFC 0072 (which burned down
the parity buckets themselves) and RFC 0092 (which consolidated the tools and is
closed). Both make `parity:api` score something other than what it reports:

1. `extra-surface-scores-overridden-ruby-files` — when a Ruby file is overridden,
   extra-surface scores its TS target against an empty allowed set, so every name
   in that file reads as novel.
2. `writer-resolves-to-set-name-when-reader-claims-bare` — a Ruby writer resolves
   to `set<Name>` when its reader already claims the bare name, contradicting the
   `conventions.ts` rule that a Ruby `foo=` maps to the same camelCase name as its
   reader (RFC 0081's premise).

Both distort the numbers other RFCs burn down against, which is why they are
scoring correctness rather than tooling polish.

## Scope

`scripts/api-compare` / `scripts/parity` scoring only. Output sharding is RFC
0097; the JSDoc tag family is RFC 0080 (closed).

## Done means

Both stories land, and the totals move only by the amount the fix explains —
each PR states the before/after `parity:api` delta it causes.
