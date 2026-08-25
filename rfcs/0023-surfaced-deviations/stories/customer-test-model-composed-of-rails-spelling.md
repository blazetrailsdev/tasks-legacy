---
title: "Customer test model declares composed_of with class objects where Rails uses class-name Strings and inferred mappings"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 100
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing PR #6828 (`composed-of-local-derivations`, RFC 0099), which
made `composedOf` accept Rails' class-name String and derive the
`name.camelize` default, resolving it through activesupport's `constantize`
registry inside the generated reader/writer
(`vendor/rails/activerecord/lib/active_record/aggregations.rb:227`, `:249-253`,
`:262`).

`packages/activerecord/src/test-helpers/models/customer.ts:94-146` still passes
the value-object CLASS to every declaration and spells out a `mapping` even
where Rails infers it. Rails
(`vendor/rails/activerecord/test/models/customer.rb:6-14`):

```ruby
composed_of :address, mapping: [ %w(address_street street), ... ], allow_nil: true
composed_of :balance, class_name: "Money", mapping: %i(balance amount)
composed_of :gps_location, allow_nil: true
composed_of :fullname_no_converter, mapping: %w(name to_s), class_name: "Fullname"
```

Note `:gps_location` carries no mapping at all — Rails infers
`[ :gps_location, :gps_location ]`, which lands on the snake_case column. The
trails partId is `gpsLocation`, so the inferred pair does not name the
`gps_location` column; converging that arm needs the attribute-name side
settled, not just the declaration rewritten.

## Converged shape

`registerConstant` the value classes (`Address`, `Money`, `GpsLocation`,
`Fullname`) alongside their definitions in `customer.ts`, then declare each
aggregation with Rails' `class_name: "Money"` String spelling and drop the
mappings Rails infers.

## Acceptance criteria

- [ ] Every `composedOf` call in `customer.ts` uses a class-name String.
- [ ] `gpsLocation` declares no `mapping`, or the story is closed with the
      attribute-naming blocker written down.
- [ ] `aggregations.test.ts`, `aggregations.trails.test.ts` and
      `reflection.test.ts` stay green.
