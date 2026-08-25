---
title: "errors_to_json asserts via JSON.stringify instead of ActiveSupport::JSON.encode"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting `validations_test.rb` (PR #6647).

`test_errors_to_json` (`vendor/rails/activemodel/test/cases/validations_test.rb:212-221`)
asserts:

```ruby
assert_equal t.errors.to_json, hash.to_json
```

`Errors#to_json` is not defined on `Errors` — it comes from
`ActiveSupport::ToJsonWithActiveSupportEncoder#to_json`
(`vendor/rails/activesupport/lib/active_support/core_ext/object/json.rb:34-50`),
which routes to `ActiveSupport::JSON.encode(self, options)`, which in turn calls
`as_json`. So the Ruby chain is `to_json` -> `ActiveSupport::JSON.encode` ->
`as_json`.

PR #6647 ported the assertion as:

```ts
expect(JSON.stringify(t.errors.asJson())).toEqual(JSON.stringify(hash));
```

with a `// Deviation:` note claiming `Object#to_json` is unported. That claim was
wrong on the second half: **`ActiveSupport.JSON.encode` is ported** and already
routes through `asJson` —
`packages/activesupport/src/json.ts:114` (`encode`) delegating to
`packages/activesupport/src/json/encoding.ts:41` (`value = asJson(value, this.options)`).
So the encoder half of Rails' chain exists; only the `Object#to_json` core_ext
entry point is missing (tracked separately by
`0072-api-compare-parity-burndown/port-to-json-with-active-support-encoder`,
marked done, though no `toJson` symbol is greppable in `activesupport/src`).

Using `JSON.stringify` bypasses the ActiveSupport encoder entirely, so the
assertion does not exercise the escaping/encoding behaviour Rails' `to_json` has
(HTML-entity escaping, `Float` handling, `escapeHtmlEntitiesInJson`).

## Converged shape

```ts
import { JSON as ActiveSupportJSON } from "@blazetrails/activesupport";

expect(ActiveSupportJSON.encode(t.errors)).toEqual(ActiveSupportJSON.encode(hash));
```

`encode` calling `asJson` reproduces Rails' `to_json` -> `encode` -> `as_json`
chain, and `Errors#asJson` already exists (`packages/activemodel/src/errors.ts`).
If `Errors#toJson` is added (see the related story), prefer `t.errors.toJson()`
so the trails call site reads like the Ruby.

## Acceptance criteria

- `it("errors to json")` in `packages/activemodel/src/validations.test.ts` routes
  through `ActiveSupport.JSON.encode` (or `Errors#toJson` once it exists) on both
  sides, not `JSON.stringify`, and the `// Deviation:` comment is removed.
- The corrected claim is reflected wherever it was recorded: `JSON.stringify` was
  chosen on the false premise that no ActiveSupport encoder was available.
- `pnpm parity:test -- --assertions --package activemodel` keeps
  `validations_test.rb` at 0 count / 0 kind / 0 value mismatches.
