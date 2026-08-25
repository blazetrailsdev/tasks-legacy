---
title: "I18nValidationTest builds ad-hoc Topic models where Rails uses replied_topic"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing PR #6975. Rails' `I18nValidationTest`
(`vendor/rails/activerecord/test/cases/validations/i18n_validation_test.rb:62-68`,
`:71-77`, and the global-default case) drives its `validates_associated` cases
off `replied_topic` — the canonical `Topic` model with its declared
`has_many :replies` (`test/models/topic.rb:49`) and real fixture replies.

`packages/activerecord/src/validations/i18n-validation.test.ts` instead defines
an ad-hoc `class Topic extends Base` per test, registers it under an invented
name (`I18nAssociatedTopic`, `I18nCustomKeyTopic`, `I18nGlobalKeyTopic`),
re-declares `has_many :replies` inline, registers the canonical `Reply` under
its Rails name from inside the describe, and seeds the association with a
`FakeReply` subclass that overrides `isValid()`. PR #6975 moved this toward
Rails (the association is now declared and `FakeReply` is a real `Reply`) but
did not retire the ad-hoc models — that was outside its acceptance criteria.

## Converged shape

The test uses the canonical `Topic` / `Reply` models and the `topics` fixtures,
with a `replied_topic` equivalent, exactly as the Rails test does — no ad-hoc
model classes, no `registerModel` inside the describe, no `seedAssociationCache`
(see `retire-seed-association-cache-test-helper`).

## Acceptance criteria

- [ ] No ad-hoc `Topic` subclass or invented registry name remains in
      `i18n-validation.test.ts`.
- [ ] Test names are unchanged; `pnpm parity:test` delta non-negative.
