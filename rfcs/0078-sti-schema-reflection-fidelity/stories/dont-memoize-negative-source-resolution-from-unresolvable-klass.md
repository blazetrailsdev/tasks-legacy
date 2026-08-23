---
title: "dont-memoize-negative-source-resolution-from-unresolvable-klass"
status: done
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6723
claim: "2026-08-23T16:57:27Z"
assignee: "dont-memoize-negative-source-resolution-from-unresolvable-klass"
blocked-by: null
closed-reason: null
---

## Context

Established while closing `reflection-registry-poison-actual-mechanism`
(PR for the RFC 0078 bundle). The cross-file reflection-memo poison is a
NEGATIVE memo formed under an incomplete registry, not a shadowed name:

`ThroughReflection#sourceReflectionName`
(`packages/activerecord/src/reflection.ts:1883-1920`) resolves candidate source
names through `throughRef.klass`. When the target model is not registered yet,
`klass` raises `NameError: Missing model class Tagging …` — and the catch at the
bottom of that method swallows it and memoizes
`_sourceReflectionNameCache = null`. `sourceReflection` then reads the `null`,
memoizes its own `null`, and `checkValidityBang`
(`reflection.ts:1949`) raises `HasManyThroughSourceAssociationNotFoundError`.

Instrumented repro (`POISON_PROBE`, generation gate disabled):

```text
NEG noSrcName Post tags       regsize 71
NEG noSrcName Member organization regsize 71
```

Today this heals only because `modelRegistry.generation` bumps on the NEXT
registration (fresh registrations bump too), which expires the memo. A negative
memo formed AFTER the last registration in a worker therefore never heals.

## Acceptance criteria

- A resolution that failed because the class could not be resolved (the
  `NameError` arm) is NOT memoized — either skip the memo write on that arm, or
  validate on read.
- The distinction is kept: a genuine "no such source association" answer on a
  fully-registered registry may still memoize.
- Regression test that fails on the baseline: form the negative memo with the
  target model absent, register nothing further, and assert the association
  resolves once the model IS registered.
- `nested-through-associations.test.ts` + `associations.test.ts` run in one
  worker stay green.
