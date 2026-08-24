---
title: "Converge the _default_attributes reset points onto Rails' reload_schema_from_cache"
status: blocked
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: 6962
claim: "2026-08-24T01:24:28Z"
assignee: "anchor-jsdoc-tag-recognition-to-line-start"
blocked-by: "trails' schema load is async (loadSchemaFromAdapter, model-schema.ts:1170), so a caller can force _defaultAttributes before the columns land and latch a memo built without them. Ruby cannot: _default_attributes (attributes.rb:241-252) reads columns_hash through the SYNCHRONOUS load_schema (model_schema.rb:530-546), which re-raises after reload_schema_from_cache when the load fails, so a Ruby memo is built from loaded columns by construction. Removing the trails-only reset in applyColumnsHash was tried in PR #6962 and reddened SQLite, PG and MariaDB: Topic's cold memo survived its load and every read raised UnknownAttributeError: unknown attribute 'title'. Withholding the memo until isSchemaLoaded instead drops eager defineAttribute writes (DefineAttributeSTITest red). Converging needs a sync schema load or a recorded replay of the eager writes; the reset is now justified at the call site with the Rails cites."
closed-reason: null
---

## Context

Rails' `ActiveRecord::Attributes::ClassMethods#define_default_attribute`
(`activerecord/lib/active_record/attributes.rb:277-291`) assigns straight into
the memoized `_default_attributes`
(`attributes.rb:241-252`), and that memo is dropped only by
`reload_schema_from_cache` (`attributes.rb:267-270`) — i.e. by
`reset_column_information`, not by an ordinary schema load. So in Rails a
`define_attribute` write survives the model's first column reflection.

trails (`packages/activerecord/src/attributes.ts`, `_defaultAttributes`) rebuilds
the set from `columnsHash` plus the pending-modification replay every time the
`_cachedDefaultAttributes` memo is cleared, and trails clears it from more
places than Rails does — notably the post-reflection path in
`packages/activerecord/src/model-schema.ts` (`applyColumnsHash` →
`resetDefaultAttributes`). An eager write made by `define_default_attribute`
before schema load is therefore silently dropped, where Rails keeps it. PR #6961
converged the two method bodies; this is the remaining divergence in the surface
they write into.

Same asymmetry shows up for `attribute_types[name] = cast_type`
(`attributes.rb:237`): trails' `attributeTypes()` memo is invalidated by the same
resets.

## Converged shape

Audit trails' `resetDefaultAttributes` / `reloadSchemaFromCache` call sites
against Rails' (`attributes.rb:267-270`, `model_schema.rb:553-568`,
`activemodel/lib/active_model/attribute_registration.rb:88-95`) and drop the
trails-only resets, so `_default_attributes` is invalidated exactly where Rails
invalidates it and an eager `define_default_attribute` write survives a first
schema load.

## Acceptance criteria

- [ ] trails' `_defaultAttributes` memo is dropped only at the Rails reset
      points; the trails-only post-reflection reset is gone or justified at the
      call site with a Rails cite.
- [ ] A test covers `defineAttribute` before schema load surviving the load.
- [ ] `pnpm parity:api:calls` / `:args` add zero rows; parity deltas non-negative.
