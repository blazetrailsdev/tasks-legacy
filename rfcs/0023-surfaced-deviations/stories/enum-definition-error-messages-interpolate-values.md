---
title: "assert_valid_enum_definition_values' three messages interpolate #{values} and drop the got: suffix"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`assertValidEnumDefinitionValues`
(`packages/activerecord/src/enum.ts:1095-1160`) raises the right error class at
the right sites, but none of its three messages is the Rails string.

Rails, `activerecord/lib/active_record/enum.rb:334-355`:

```ruby
def assert_valid_enum_definition_values(values)
  case values
  when Hash
    if values.empty?
      raise ArgumentError, "Enum values #{values} must not be empty."
    end
    if values.keys.any?(&:blank?)
      raise ArgumentError, "Enum values #{values} must not contain a blank name."
    end
  when Array
    if values.empty?
      raise ArgumentError, "Enum values #{values} must not be empty."
    end
    unless values.all?(Symbol) || values.all?(String)
      raise ArgumentError, "Enum values #{values} must only contain symbols or strings."
    end
    if values.any?(&:blank?)
      raise ArgumentError, "Enum values #{values} must not contain a blank name."
    end
```

Every Rails message interpolates the offending `values` collection; trails omits
it in all of them. The type message also diverges in both word order and a
trails-only suffix: Rails is `"...must only contain symbols or strings."`,
trails is `"Enum values must only contain strings or symbols, got: string,
number"`. There is no Rails counterpart for the `got:` list.

Surfaced while converging the Ruby-Symbol modelling in PR #6494 (RFC 0096); the
homogeneity guard (`all?(Symbol) || all?(String)`) is now faithful, the message
it raises is not.

## Converged shape

All three messages interpolate the values collection the way `#{values}` does
(Ruby's `Hash#to_s` / `Array#to_s` — `inspect` form) and the type message reads
`"Enum values #{values} must only contain symbols or strings."` with the `got:`
suffix deleted. Update the message expectations in
`packages/activerecord/src/enum.trails.test.ts` (the `/blank name/`,
`/finite numbers/`, `/symbols/` matchers) to the Rails strings.

## Acceptance criteria

- [ ] The empty, blank-name, and type messages match enum.rb:338/342/346/350/354
      verbatim, including the interpolated collection.
- [ ] The trails-only `got: <typeof list>` suffix is gone.
- [ ] `enum.test.ts` and `enum.trails.test.ts` green on all three adapters.
