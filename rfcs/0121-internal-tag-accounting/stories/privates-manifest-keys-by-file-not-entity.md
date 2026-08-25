---
title: "rails-private-methods.json is keyed per-file, so one entity's private name over-tags unrelated same-named members"
status: claimed
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: "2026-08-25T17:14:37Z"
assignee: "converge-test-fixtures-class-attribute-stores"
blocked-by: null
closed-reason: null
---

## Context

`eslint/rails-private-methods.json` is keyed by **(TS file, bare name)**, with no
notion of which entity in that file owns the name. The projection in
`scripts/build-rails-privates-manifest.ts` folds every entity whose Ruby source
file maps to one TS file into a single name set, so a private name on one class
gates every same-named member in the file, including members of unrelated types.

Concrete instance surfaced by `rails-privates-manifest-missing-gem-packages`
(PR #7042), which enrolled `rack`:

`vendor/rack/lib/rack/lock.rb` declares a `private` `unlock` on `Rack::Lock`
(under the `private` section following its public `call`). trails'
`packages/activerecord/../rack/src/lock.ts` also declares a local `Mutex`
protocol:

```ts
interface Mutex {
  lock(): void;
  unlock(): void;
}

class DefaultMutex implements Mutex { ... }
```

`Mutex#unlock` is a mutex-protocol member, not `Rack::Lock`'s private helper —
Ruby's `Mutex#unlock` (stdlib) is public. But because both live in `lock.ts`,
the rule demanded `@internal` on the interface member and on
`DefaultMutex#unlock`, and PR #7042 tagged them to get the lane green. Those two
tags are **wrong on the merits** and should come off once the manifest can tell
the entities apart.

This is a general precision problem, not one file: any TS file that hosts both a
Rails-mirroring class and a helper type sharing a method name will over-tag the
same way. The blast radius grows as more packages enroll.

## Converged shape

Give the manifest entity granularity, so a name gates only the members of the
entity that contributes it.

The extractor already carries `fqn` per `RubyEntity`
(`scripts/build-rails-privates-manifest.ts`'s `RubyEntity` interface), and
`api-compare` already resolves a TS class name to a Ruby FQN, so the data exists
on both sides. Options:

1. Key the manifest as `file -> className -> names`, with a `null`/`""` bucket
   for top-level functions, and have `eslint/rails-private-jsdoc.mjs` resolve the
   enclosing class of the reported node before looking a name up.
2. If per-class resolution turns out to be more than this RFC wants, at minimum
   subtract names that resolve to a public member of a DIFFERENT entity in the
   same file — the same "mixed" subtraction the builder already applies per Ruby
   name, extended across entities.

Then remove the two `@internal` tags in `packages/rack/src/lock.ts` and re-sweep
the enrolled packages for other over-tags this created.

## Acceptance criteria

- [ ] A private name on one entity no longer gates a same-named member of an
      unrelated entity in the same TS file.
- [ ] The `Mutex`/`DefaultMutex` `unlock` tags in `packages/rack/src/lock.ts` are
      removed and the rule stays clean.
- [ ] The enrolled packages are re-swept; any other over-tag introduced by
      file-level keying is removed.
- [ ] `scripts/api-compare/config.test.ts` and the manifest builder's tests pass.
- [ ] `Rails API/Test Comparison` green.
