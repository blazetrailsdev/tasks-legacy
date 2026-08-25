---
title: "converge-reverse-merge-bang-key-presence"
status: claimed
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-25T12:46:55Z"
assignee: "converge-reverse-merge-bang-key-presence"
blocked-by: null
closed-reason: null
---

## Context

`AttributeSet#reverse_merge!` is
`attributes.reverse_merge!(target_attributes.attributes) && self`
(`vendor/rails/activemodel/lib/active_model/attribute_set.rb:100-102`).
`Hash#reverse_merge!` keeps an entry whenever the **key is present** in the
receiver — Ruby `Hash#key?`.

`packages/activemodel/src/attribute-set.ts` (`reverseMergeBang`) instead skips a
name when `this.isKey(name)` is true, and trails' `isKey` is the port of
AttributeSet's own `key?` (`attribute_set.rb:44-46`), which is
`attributes.key?(name) && self[name].initialized?` — an _initialized_-value
check, not a map-presence check. So a receiver carrying `name` as an
uninitialized Attribute has the target's Attribute copied over it, where Rails'
Hash keeps the receiver's own.

Noted while landing `converge-attribute-deep-dup-onto-ruby-dup` (PR #TBD),
which converged the cloning half of `reverseMergeBang` (Rails clones nothing)
but left this guard alone as out of scope.

The only caller is `becomes` (`packages/activerecord/src/persistence.ts:1316`),
so the observable case is `becomes` on a partially-selected record.

## Acceptance criteria

- [ ] `reverseMergeBang` skips a name when the backing map already has it
      (`this._attributes.has(name)`), mirroring `Hash#reverse_merge!`, not when
      `isKey` reports it initialized.
- [ ] `becomes` behaviour verified against Rails for a partially-selected
      record (`Post.select(:id).first.becomes(SpecialPost)`); assertions take
      what Rails produces.
- [ ] activemodel + activerecord suites green on all lanes.
