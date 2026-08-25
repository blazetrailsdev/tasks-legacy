---
title: "writer_method's hash_from_multiparameter_assignment arm is not ported"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing PR #6828 (`composed-of-local-derivations`, RFC 0099).

Rails' `writer_method` has a multiparameter arm trails' `writerMethod`
(`packages/activerecord/src/aggregations.ts:171-243`) does not port at all
(`vendor/rails/activerecord/lib/active_record/aggregations.rb:266-271`):

```ruby
hash_from_multiparameter_assignment = part.is_a?(Hash) &&
  part.keys.all?(Integer)
if hash_from_multiparameter_assignment
  raise ArgumentError unless part.size == part.each_key.max
  part = klass.new(*part.sort.map(&:last))
end
```

trails routes multiparameter aggregate assignment through
`packages/activerecord/src/multiparameter-attribute-assignment.ts:131`, which
builds the value object itself and hands the writer a finished object, so the
Rails arm has no counterpart here — including its `ArgumentError` on a sparse
index set, which `multiparameter_attributes_test.rb:375-384`
(`test_multiparameter_assignment_of_aggregation_with_large_index`) covers.

## Converged shape

Port the arm into `writerMethod` and route the multiparameter assignment path
to it (pass the integer-keyed hash straight through), so the size/max guard and
the `klass.new(*part.sort.map(&:last))` construction live where Rails puts them.

## Acceptance criteria

- [ ] `writerMethod` ports `hash_from_multiparameter_assignment` including the
      `raise ArgumentError unless part.size == part.each_key.max` guard.
- [ ] `multiparameter-attributes.test.ts` "multiparameter assignment of
      aggregation with large index" / "with missing values" stay green.
