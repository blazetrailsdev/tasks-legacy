---
title: "port-inflections-uncountables-collection"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

Surfaced by RFC 0096 story `naming-burndown-activesupport`, which converged 50
of the 85 `naming` call-argument rows in `packages/activesupport/src/`. Seven
rows in `inflector/inflections.ts` are NOT identifier renames and were left in
place per that story's acceptance criterion 4.

Rails' `@uncountables` is an `Inflections::Uncountables < Array`
(`vendor/rails/activesupport/lib/active_support/inflector/inflections.rb:22-63`)
whose `add`/`delete`/`<<` downcase internally:

```ruby
def delete(entry)
  super entry.downcase
end
def add(words)
  words = words.flatten.map(&:downcase)
  ...
end
```

So Rails' call sites pass the bare local — `@uncountables.delete(rule)`
(inflections.rb:152), `.delete(replacement)` (:153), `.delete(singular)` /
`.delete(plural)` (:175-176), `.add(words)` (:209).

trails has no `Uncountables` class; `inflections.ts:41,44,50,53,58,59,83`
pushes the downcasing to every call site as `rule.toLowerCase()` etc. That is
an invented conversion at the call site, and it is what the comparator flags.

## Acceptance criteria

1. Port `ActiveSupport::Inflector::Inflections::Uncountables`
   (inflections.rb:22-63) as its own type, with Rails' `add`, `delete`, `<<`,
   `uncountable?`, and the private `to_regex`, owning the downcasing.
2. `plural`, `singular`, `irregular`, and `uncountable` in `inflections.ts`
   pass the bare local, matching Rails' call sites.
3. The seven `naming` rows for `inflector/inflections.ts` clear in
   `pnpm parity:api:calls:args:report`; no new `shape` rows.
4. `pnpm vitest run packages/activesupport/src/inflector` green.
