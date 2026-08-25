---
title: "uniqueConstraintForBang renders the column array as JS strings, not Ruby symbols"
status: draft
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
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

Surfaced while landing
`check-constraint-raise-message-uses-json-stringify-not-ruby-hash-inspect`
(PR #7046), which routed the schema-statement "has no ... constraint" raise
messages through `rubyInspectHash` so they render Ruby's `{name: "x"}` instead
of JSON's `{"name":"x"}`.

One half of `uniqueConstraintForBang` was left short of Rails. Rails
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:1115`):

```ruby
raise(ArgumentError, "Table '#{table_name}' has no unique constraint for #{column || options}")
```

`column` there is a Symbol or an Array of Symbols, and `#{}` calls `to_s`:
`#{:name}` renders `name`, `#{[:a, :b]}` renders `[:a, :b]` (Array#to_s ==
inspect, so the elements are INSPECTED as symbols).

trails
(`packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts`,
in `uniqueConstraintForBang`) renders the array arm with `rubyInspectArray`
over JS strings, so it emits `["a", "b"]` — double-quoted strings — where Ruby
emits `[:a, :b]`. The scalar arm is correct (a bare string, matching `to_s`).

The same question applies to `exclusionConstraintForBang`
(schema_statements.rb:1095) if its `expression`/options ever carry symbol-ish
values.

## Acceptance criteria

- [ ] The array arm renders Ruby's symbol spelling — per CLAUDE.md a Ruby
      Symbol is the `":name"` string, so decide and document whether these
      column values carry the leading colon and render accordingly
      (`[:a, :b]`), rather than JS-string inspect (`["a", "b"]`).
- [ ] Check `exclusionConstraintForBang` and `foreignKeyForBang` for the same
      gap while in the file.
- [ ] If a Rails test asserts either message verbatim, port it; watch the
      assertion-value ratchet — the comparer keeps an interpolated string's
      literal PREFIX only, so mirror Ruby's `#{}` structure rather than
      inlining the resolved literal (this cost a CI round on PR #7046).
