---
title: "include() is unusable for a module of Rails-private members (ar-models plugin synthesizes a public Included<> merge)"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activerecord-cli"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR 6766, whose first shape used the documented mixin idiom —
`include()` from `@blazetrails/activesupport` — and had to be backed out.

CLAUDE.md ("Module mixins") names `include()` / `Included<>` as the settled
port of Ruby `include SomeModule`. But a file-scope `include(...)` call makes
the tsc-wrapper's ar-models plugin
(`packages/activerecord-cli/src/tsc-wrapper/ar-models-plugin.ts:38-52`, the
`INCLUDE_CALL_PATTERN` pre-filter) synthesize an
`interface X extends Included<typeof Mod>` declaration merge, and that
synthesized interface is **public**. So `include()` cannot be used for a module
whose members the class declares `protected` — the merge fails with TS2320 /
TS2430 under `pnpm test:types:virtualized`.

`ThroughAssociation` hits exactly this: `target_scope`, `stale_state`,
`foreign_key_present?` and `transaction` are all private in Rails
(`vendor/rails/activerecord/lib/active_record/associations/through_association.rb:9`)
and `protected` on `Association` in trails. The PR shipped the older
`Object.assign(Klass.prototype, { ...Mod })` spread instead, which works but is
not the idiom the guide points at — and the next agent reaching for `include()`
on a private Rails module will rediscover this the same expensive way.

## Acceptance criteria

- [ ] The synthesized interface merge preserves member visibility (or the
      plugin skips members the class already declares), so `include()` works
      for a module of Rails-private members.
- [ ] `pnpm test:types:virtualized` green with `ThroughAssociation` installed
      via `include()` in
      `packages/activerecord/src/associations/has-{one,many}-through-association.ts`.
- [ ] CLAUDE.md's "Module mixins" section states which spelling applies to a
      private/protected module, so the choice is not re-derived per call site.
