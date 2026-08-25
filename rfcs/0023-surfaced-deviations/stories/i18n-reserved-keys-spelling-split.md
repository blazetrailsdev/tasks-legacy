---
title: "Split RESERVED_KEYS into option-stripping and pattern-matching lists"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "i18n"
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

`RESERVED_KEYS` in `packages/i18n/src/i18n.ts` is spelled camelCase
(`deepInterpolation`, `skipInterpolation`, `fallbackInProgress`,
`fallbackOriginalLocale`, `exceptionHandler`) because the same list doubles as
the translate-option keys, and trails options are camelCase.

Rails spells them as snake_case Symbols at `vendor/i18n/lib/i18n.rb:19-34`, and
the list feeds two different consumers:

1. `Utils.except(options, *RESERVED_KEYS)` (`backend/base.rb:56`) — stripping
   option keys. camelCase is right here, given trails' option spelling.
2. `I18n.reserved_keys_pattern` (`i18n.rb:50-52`) — a regex matched against
   **translation content**, e.g. `"an entry with %{scope}"`. Ruby-authored
   locale data uses the snake_case names, so camelCase is wrong here.

The current port papers over the split by building the pattern from both
spellings (`underscore()` in `i18n.ts`), which means trails rejects
`%{deep_interpolation}` _and_ `%{deepInterpolation}` where Rails rejects only
the former. The two consumers want different lists and should have them.

Related: `I18n.reserve_key` (`i18n.rb:44-47`) pushes a caller-supplied key onto
the same list, so whichever split is chosen has to keep working for user keys.

## Acceptance criteria

- The option-stripping list and the reserved-keys-pattern list are separated,
  each carrying the spelling its consumer actually needs.
- `reservedKeysPattern()` matches exactly the names Rails matches — no
  camelCase alternates — so a translation containing `%{deepInterpolation}`
  interpolates instead of raising.
- `reserveKey` feeds both lists correctly, and the pattern cache still
  invalidates.
