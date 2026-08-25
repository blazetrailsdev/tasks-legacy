---
title: "Active Support never mixes in I18n Backend::Fallbacks that instance_or_fallback depends on"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::Inflector::Inflections.instance_or_fallback(locale)` walks
`I18n.fallbacks[locale]` and returns the first already-built instance
(`vendor/rails/activesupport/lib/active_support/inflector/inflections.rb:70-75`).
The port landed in PR #6604 and reads `I18n.fallbacks()` from
`@blazetrails/i18n` directly.

Rails only has `I18n.fallbacks` because `active_support/i18n.rb` requires
`i18n/backend/fallbacks` — which trails has NOT wired up yet, as
`packages/activesupport/src/i18n.ts` states in its own header ("`run_load_hooks(:i18n)`
and `i18n/backend/fallbacks` have no port yet"). So `instanceOrFallback` reaches
a fallbacks implementation that no `Simple` backend is actually mixing in, and
the locale chain it walks is the gem default rather than one Active Support
configured.

## Converged shape

Port `active_support/i18n.rb`'s `require "i18n/backend/fallbacks"` equivalent:
mix `Fallbacks` into the backend the way `i18n.rb` does, so
`I18n.fallbacks` and the backend's own lookup agree. `instanceOrFallback` then
needs no change — it is already the Rails body.

## Acceptance criteria

- [ ] The Active Support I18n bootstrap mixes in `Backend::Fallbacks`, mirroring
      `active_support/i18n.rb`.
- [ ] `Inflections.instanceOrFallback` resolves through the same chain the
      backend uses; a regression test covers a non-`en` locale falling back.
- [ ] `pnpm parity:api` / `parity:test` deltas non-negative.
