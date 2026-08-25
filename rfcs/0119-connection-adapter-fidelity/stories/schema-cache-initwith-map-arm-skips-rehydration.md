---
title: "SchemaCache initWith Map arm skips rehydration"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`SchemaCache#initWith`
(`packages/activerecord/src/connection-adapters/schema-cache.ts`) accepts two
coder shapes per key: a `Map` (a live in-process cache handing over its own
state) and a plain object (the JSON payload read back from disk). Only the plain
-object arm normalizes the entries — `rehydrateColumn` for `columns`,
`rehydrateIndex` for `indexes` (added in PR #5890). The `instanceof Map` arm
assigns the map straight through under a cast, so a `Map` carrying plain rows
silently installs structural objects where `Column` / `IndexDefinition`
instances are expected, and the cast hides it from the compiler.

Rails has no such fork: `init_with` reads `coder["columns"]` / `coder["indexes"]`
from a single deserialized shape whose YAML tags already carry the classes
(`vendor/rails/activerecord/lib/active_record/connection_adapters/schema_cache.rb:281-291`).
The dual-shape handling is a trails invention layered on JSON.

No caller is known to pass unhydrated `Map` values today, which is why #5890
left the arm alone — but the divergence is latent, exactly like the one #5890
fixed.

## Acceptance criteria

- `initWith` takes the single object shape Rails has (`schema_cache.rb`). The
  `Map` arm is deleted if nothing calls it; if a real caller exists, its
  entries route through `rehydrateColumn` / `rehydrateIndex` so both arms
  produce the same instance types, and the casts go.
- A test covers `initWith` with the retained shape, asserting the resulting
  cache yields `Column` / `IndexDefinition` instances rather than plain rows.
- Either way, `initWith` afterwards has exactly one rehydration path — a second
  arm that skips rehydration is the defect, not the `Map` type itself.
