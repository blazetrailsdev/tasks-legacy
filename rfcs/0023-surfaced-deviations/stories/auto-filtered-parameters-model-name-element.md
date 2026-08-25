---
title: "auto-filtered-parameters-model-name-element"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

# AutoFilteredParameters#apply_filter: model_name.element, and the bespoke-mock tests that block it

## Context

Surfaced in review of #6719 (RFC 0106 wave 4c, encryption slice), which
converged `apply_filter`'s `compact.join(".")` shape but left the class-name
segment alone.

`vendor/rails/activerecord/lib/active_record/encryption/auto_filtered_parameters.rb:53`
builds the filter as:

    filter = [("#{klass.model_name.element}" if klass.name), attribute.to_s].compact.join(".")

`packages/activerecord/src/encryption/auto-filtered-parameters.ts:85` spells the
first segment `underscore(klass.name)`. The two diverge for a namespaced class:
`Admin::Post` has `model_name.element == "post"`, while `underscore` yields
`"admin/post"`, so a namespaced model's encrypted attributes land in
`filter_parameters` under the wrong key and are not redacted.

`ModelName#element` is ported (`packages/activemodel/src/naming.ts`) and
`klass.modelName` is already used on AR model classes elsewhere
(`packages/activerecord/src/integration.ts:122`), so the fix itself is one
expression.

**What blocks it** — and what this story actually has to do first. The two
`AutoFilteredParameters` tests in
`packages/activerecord/src/encryption/configurable.test.ts` (around :164 and the
one above it) drive `encrypts` against a bespoke mock:

    class PaymentModel {}
    const modelClass = Object.assign(PaymentModel, { _attributeDefinitions: new Map() });

A bare class has no `modelName`, so switching `apply_filter` to
`klass.modelName.element` reds both tests with
`Cannot read properties of undefined (reading 'element')` — measured on #6719,
which is why that branch reverted the one-line change rather than ship it with a
fallback.

Rails' own tests (`test/cases/encryption/configurable_test.rb:44-85`) use real
models — subclasses of `Pirate` with `self.table_name = "pirates"` — and a
`Struct.new(:config)` stand-in for the application, under a
`with_auto_filtered_parameters` helper. `pirates` is in the canonical schema and
`Pirate` is in `packages/activerecord/src/test-helpers/models/`.

## Acceptance criteria

- [ ] The `AutoFilteredParameters` tests in `configurable.test.ts` drop the
      bespoke `class PaymentModel {}` mock and drive `encrypts` against the
      canonical `Pirate` model the way `configurable_test.rb:44-85` does,
      including the `with_auto_filtered_parameters` helper shape. Rails test
      names verbatim where a Rails test exists; no invented table, no
      `defineSchema`.
- [ ] `apply_filter` builds the class-name segment with
      `klass.modelName.element`, matching `auto_filtered_parameters.rb:53`, with
      no fallback branch Rails does not have.
- [ ] A test covers the namespaced case the divergence is about — a model whose
      `name` demodulizes (Rails' `element` is the demodulized, underscored
      singular).
- [ ] `pnpm parity:api:calls`, `pnpm parity:api:calls:args` and
      `pnpm parity:api:extra --package activerecord` green with no new row.
- [ ] `pnpm parity:test` delta non-negative.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
