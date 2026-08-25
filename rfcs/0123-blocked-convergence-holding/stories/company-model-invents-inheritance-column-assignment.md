---
title: 'Company sets inheritance_column = "type", which Rails'' company.rb does not'
status: blocked
updated: 2026-08-25
rfc: "0123-blocked-convergence-holding"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: "2026-08-18T22:11:19Z"
assignee: "suppressed-call-lets-sibling-claim-ts-candidate"
blocked-by: 'Blocked on the same missing sync schema signal as converge-new-sti-gate-drop-stienabled-disjunct (the story it says to sequence after, still blocked). MEASURED 2026-08-18 on this branch: deleting `Company.inheritanceColumn = "type"` (test-helpers/models/company.ts:644) reds 2 cases immediately — inheritance.test.ts:197 ("Company.find on a row whose type is bad_class_name") stops raising SubclassNotFound and returns the base-class record, plus one case in inheritance-sti-new-gate.trails.test.ts. stiEnabled (inheritance.ts:455) is `_inheritanceColumn != null`, so with the assignment gone findStiClassForRow degrades to the base class instead of raising, exactly as the story predicted. Converging needs columnNames()/attribute_types readable SYNCHRONOUSLY at find/new time (Rails inheritance.rb:61 -> model_schema.rb load_schema), which trails does not have. AUDIT of the other assignments under test-helpers/models/ is DONE and can be reused: post.ts 709/759/796/827/839/856/875/934 are all real (post.rb:247,268,280,302,310,320,331,349 `self.inheritance_column = :disabled`); parrot.ts:28 is real (parrot.rb:4 `:parrot_sti_class`); vegetables.ts:13 is real in substance but SPELLED differently — vegetables.rb:6 overrides `def self.inheritance_column` returning "custom_type" where trails assigns it; membership.ts:62 `this.inheritanceColumn = "type"` is INVENTED — membership.rb has no inheritance_column at all, only `enum :type, %i(...)` — and is the same class of deletion as company.ts, so it blocks on the same signal.'
closed-reason: null
---

# Company sets `inheritance_column = "type"`, which Rails' company.rb does not

## Context

Surfaced in PR #6720 while measuring
`converge-new-sti-gate-drop-stienabled-disjunct` (blocked in the same RFC).

`packages/activerecord/src/test-helpers/models/company.ts:644`:

```ts
Company.inheritanceColumn = "type";
```

Rails' `vendor/rails/activerecord/test/models/company.rb` has no such
assignment — `Company < AbstractCompany` relies on the DEFAULT inheritance
column, which is already `"type"`, and on the real `companies.type` column in
`vendor/rails/activerecord/test/schema/schema.rb`. Per CLAUDE.md the canonical
test models must mirror Rails' exactly, so this is an invented line in a
canonical model file.

It is not cosmetic. `stiEnabled` (`packages/activerecord/src/inheritance.ts:455`)
is `_inheritanceColumn != null`, i.e. it reports "someone assigned the column",
and this line is what makes it true for the whole Company/Client/Firm tree. That
signal is load-bearing in at least:

- `subclassFromAttributesForNew`'s gate
  (`packages/activerecord/src/inheritance.ts`), whose `!stiEnabled(...)` disjunct
  is the deviation `converge-new-sti-gate-drop-stienabled-disjunct` exists to
  remove — it stands in for Rails' `_has_attribute?(inheritance_column)`
  (`vendor/rails/activerecord/lib/active_record/inheritance.rb:61`) over the
  window where reflection is still cold;
- `findStiClassForRow` (`inheritance.ts`), which raises vs. degrades to the base
  class on that same flag.

So deleting the line in isolation will change STI dispatch for the canonical
company fixtures, and the two stories are coupled: the blocked one cannot
converge while `stiEnabled` is doing work that only this invented assignment
enables.

## Converged shape

Delete `Company.inheritanceColumn = "type"` so the model matches
`company.rb`, and let the `type` column come from reflection of the canonical
schema, as it does in Rails. That requires the consumers above to stop reading
`stiEnabled` as a proxy for "this hierarchy is STI" — the same synchronous
schema-signal problem recorded as the blocker on
`converge-new-sti-gate-drop-stienabled-disjunct`, so sequence this after (or
together with) that one.

Audit the other `inheritanceColumn` assignments in
`packages/activerecord/src/test-helpers/models/` against their Rails
counterparts at the same time — `parrot.ts:28`
(`parrot_sti_class`), `membership.ts:62` and the `post.ts` `"disabled"` ones are
each either real in Rails or not, and only the real ones should survive.

## Acceptance criteria

- [ ] `packages/activerecord/src/test-helpers/models/company.ts` carries no
      `inheritanceColumn` assignment, matching
      `vendor/rails/activerecord/test/models/company.rb`.
- [ ] Every remaining `inheritanceColumn` assignment under
      `test-helpers/models/` is present in the mirrored Rails model file, cited
      by `file.rb:LINE`.
- [ ] STI dispatch for the company tree still matches Rails at `new`,
      `instantiate` and association build; the Rails-mirrored inheritance suites
      stay green.
- [ ] `parity:api` / `parity:test` deltas non-negative.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Evidence from `vegetables-model-assigns-inheritance-column-rails-overrides-method` (PR pending, 2026-08-18)

`vegetables.ts` was converged onto Rails' `def self.inheritance_column` spelling
(vegetables.rb:6) — a `static override get inheritanceColumn()` — which leaves
`_inheritanceColumn` unset, so `stiEnabled(Vegetable)` is now **false** for that
tree while `Vegetable.inheritanceColumn` still answers `"custom_type"`.

STI dispatch was UNCHANGED: `inheritance.test.ts` stays green, including
`Vegetable.find(1) instanceof Cucumber` / `Vegetable.find(2) instanceof Cabbage`
(inheritance.test.ts:208-211) and the `becomes`/`becomesBang` cases. So the
database-row dispatch paths this tree exercises do NOT in fact depend on the
`stiEnabled` sentinel, which is direct evidence that the disjunct is reading a
trails-invented signal rather than a load-bearing one.
