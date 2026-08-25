---
title: "from-json-decodes-with-the-json-gem-not-activesupport"
status: ready
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::Serializers::JSON#from_json` decodes through ActiveSupport, not
the JSON gem:

```ruby
def from_json(json, include_root = include_root_in_json)
  hash = ActiveSupport::JSON.decode(json)
  ...
```

(`vendor/rails/activemodel/lib/active_model/serializers/json.rb:147`)

trails' `packages/activemodel/src/serializers/json.ts` calls
`globalThis.JSON.parse(json)` instead. That is a dropped delegation, not an
equivalent: `ActiveSupport::JSON.decode` applies Rails' own decoding —
notably `parse_json_times`, which converts ISO-8601-looking strings to `Time` /
`Date` objects when `ActiveSupport.parse_json_times` is on
(`vendor/rails/activesupport/lib/active_support/json/decoding.rb:22-32`), and
`convert_dates_from` walking nested hashes/arrays (`:56-70`). A `from_json`
round-trip of a timestamp therefore yields a String in trails where Rails
yields a Time.

Surfaced while converging `from_json`'s body in #7015
(`converge-from-json-body-onto-assign-attributes`): the rest of the body is now
line-for-line json.rb:146-150, and this is the one remaining call divergence.

## Converged shape

`from_json` calls `ActiveSupport::JSON.decode(json)`.

`packages/activesupport/src/json/` currently ships `Encoding` but no `decode`
entry point, so this depends on that surface existing — see the sibling story
`activesupport-json-decode-quirks-and-parse-json-times`
(0023-surfaced-deviations), which covers the decoder itself. This story is the
activemodel-side call site: once `decode` exists, `from_json` must route
through it rather than `globalThis.JSON.parse`.

## Acceptance criteria

- `serializers/json.ts`'s `from_json` calls ActiveSupport's `JSON.decode`,
  matching json.rb:147.
- A test covers the timestamp case Rails' decoder handles and the JSON gem's
  does not.
- `pnpm parity:api:calls` clean without a new baseline row for `decode`.
