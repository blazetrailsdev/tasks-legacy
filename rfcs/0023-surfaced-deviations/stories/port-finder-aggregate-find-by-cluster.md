---
title: "port-finder-aggregate-find-by-cluster"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
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

## Context

Split off from faithful-port-finder-find-by-cluster (PR TBD). The finder_test.rb
aggregate find_by / hash-condition cluster in
packages/activerecord/src/finder.test.ts still uses synthetic inline-stub Post
tests with weak `toBeDefined()` assertions instead of faithful ports onto the
real Customer composed_of fixtures Rails uses:

- test_find_by_one_attribute_that_is_an_aggregate (Customer.find_by_address)
- test_find_by_one_attribute_that_is_an_aggregate_with_one_attribute_difference
- test_find_by_two_attributes_that_are_both_aggregates
- test_find_by_two_attributes_with_one_being_an_aggregate
- the `test_hash_condition_find_*_with_aggregate_* family`
- test_find_on_hash_conditions_with_explicit_table_name_and_aggregate

BLOCKER: trails' predicate builder does NOT expand composed_of value objects in
hash conditions (`where({ address: addressObj })`). `TableMetadata#aggregatedWith`
(table-metadata.ts:110) exists but is never called from the where-clause path.
Rails expands aggregate hash conditions via
PredicateBuilder#convert_dictionary_to_predicates → aggregation reflection
component mappings. That expansion must be implemented first (or as part of this
story) before these tests can ride Customer/aggregations faithfully.

## Acceptance criteria

- [ ] Implement composed_of hash-condition expansion in the predicate builder
      so `Customer.findBy({ address })` / `where({ balance })` expand to the
      mapped component columns (street/city/country, balance amount, etc.).
- [ ] Port the aggregate find_by + hash-condition cluster onto canonical
      Customer + customers/aggregations fixtures; drop the synthetic stubs.
- [ ] Test names match finder_test.rb verbatim; require-canonical-schema clean.
