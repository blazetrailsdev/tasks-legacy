---
title: "discriminate-class-for-record-should-call-find-sti-class"
status: ready
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

Rails `vendor/rails/activerecord/lib/active_record/inheritance.rb:301` is
`find_sti_class(record[inheritance_column])`. trails
`packages/activerecord/src/inheritance.ts:867` routes
`discriminateClassForRecord` through `findStiClassForRow` instead — a trails
variant that matches only within `baseClass`'s tracked subtree, because JS has
no autoloader to resolve a class name the way Rails' `find_sti_class` does. The
bare `findStiClass` is reached only when STI is explicitly enabled
(inheritance.ts:1075-1078).

That registry-safe variant is trails-invented surface on a hot STI path, and it
diverges from Rails whenever the discriminator names a class outside the tracked
subtree. Surfaced by the RFC 0106 call-set gate (the `find_sti_class` row, now a
CONVERGEABLE `@missingRailsCall` tag at the call site).

Related: [[project_sti_schema_host_redirect_is_a_trails_invention]] territory —
the STI registry is the standing trails/Rails seam.

## Acceptance criteria

- [ ] `discriminateClassForRecord` calls `findStiClass`, matching inheritance.rb:301, with the registry lookup folded into `findStiClass` itself rather than a separate reader.
- [ ] `findStiClassForRow` is deleted, or reduced to nothing callers outside `findStiClass` reach.
- [ ] The `@missingRailsCall find_sti_class` tag on `inheritance.ts` is deleted, not re-justified.
- [ ] `pnpm parity:api:extra --package activerecord` shows no new extra name; all three DB lanes green.
