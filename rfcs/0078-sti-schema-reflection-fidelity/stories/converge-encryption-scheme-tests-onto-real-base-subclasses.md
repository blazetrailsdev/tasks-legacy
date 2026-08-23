---
title: "converge-encryption-scheme-tests-onto-real-base-subclasses"
status: claimed
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-23T16:27:27Z"
assignee: "converge-encryption-scheme-tests-onto-real-base-subclasses"
blocked-by: null
closed-reason: null
---

## Context

Rails' `encryption_schemes_test.rb:120-133` and `:166-180` declare `encrypts` on
a real ActiveRecord model (`Class.new(Author) { self.table_name = "authors";
encrypts :name, ... }`) and assert through the record round-trip — `create!`
then `find_by_name` — not through the resolved type.

PR #6807 moved the four affected cases in
`packages/activerecord/src/encryption/encryption-schemes.test.ts` off
`ActiveModel::Model` onto real `Base` subclasses over `authors`, which let the
`encrypts` decorator call `columns_hash` unconditionally as Rails does
(`encryptable_record.rb:91`; the concern is mixed into `ActiveRecord::Base`
alone, `base.rb:313`). What remains is the assertion shape: the two
Rails-named cases still assert on `typeForAttribute(...).previousTypes` and
`deserialize`, where Rails asserts a persisted round-trip.

## Acceptance criteria

- [ ] `don't use global previous schemes with a different deterministic nature`
      asserts `create!`/`name` round-trip as Rails does
      (encryption_schemes_test.rb:131-132).
- [ ] `... when performing queries` asserts `find_by_name` hits and misses as
      Rails does (encryption_schemes_test.rb:178-179).
- [ ] Test names unchanged; `encryption/` suites pass.
