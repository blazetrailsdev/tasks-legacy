---
title: "extra-surface: walkMixin ignores methodFile, widening a reopening file's allow-set"
status: draft
updated: 2026-08-24
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing PR #6978 (`extra-surface-allow-reopened-module-method-files`),
which taught `buildPackageReport` to register an entity under every Ruby file
its methods declare, not just its primary `info.file`, so a port placed where
Rails REOPENS a module is no longer scored as drift.

Those reopening registrations carry `methodFile: rubyFile`, and
`collectAllowedNames` honours it for the entity's own methods —
`scripts/api-compare/extra-surface.ts:1188`:

```ts
if (methodFile !== undefined && m.file !== methodFile) continue;
```

But the mixin walk two lines below it does **not** take the filter
(`extra-surface.ts:1267-1268`):

```ts
for (const inc of info.includes ?? []) walkMixin(inc, fqn);
for (const ext of info.extends ?? []) walkMixin(ext, fqn);
```

`walkMixin` adds every method of every included/extended module to the
allow-set regardless of which `.rb` declares it. So for a file that merely
reopens `ActiveModel::Validations`, the allow-set gains the full method set of
everything `Validations` includes — names that reopening file declares none of.
Any TS name in that file matching one of them is silently allowed instead of
scored, which is a false negative in the direction the tool exists to prevent.

The asymmetry predates #6978 (the original reopened-CLASS recovery path had it
too), but #6978 widened its reach from "files absent from `rubyFiles`" to "every
file any method names", so it is worth closing now rather than later.

## Converged shape

`walkMixin` takes the same `methodFile` the entity was registered with and
applies the `m.file !== methodFile` filter to the mixin's contributed methods,
so a reopening registration allows exactly the names that Ruby file declares —
the invariant #6978's second unit test
(`"does not widen the allow-set beyond the methods the reopening file declares"`,
`scripts/api-compare/extra-surface.test.ts`) asserts for own-methods, extended
to mixins.

Entities registered WITHOUT `methodFile` (the ordinary primary-site case) must
keep today's unfiltered behaviour — a class's mixin methods are legitimately
allowed in its own file.

## Acceptance criteria

- [ ] `walkMixin` is method-file-filtered when the entity carries `methodFile`,
      and unfiltered when it does not.
- [ ] A unit test covers a reopening file whose module includes a mixin: the
      mixin's methods are NOT in that file's allow-set, but ARE in the primary
      file's.
- [ ] `pnpm parity:api:extra:gate` stays green; any package whose numbers DROP
      is tightened with `pnpm parity:api:extra:tighten`, never raised.
- [ ] If the tighter filter surfaces genuinely-novel names, they are removed or
      tagged `@noRailsEquivalent` — not baselined.
