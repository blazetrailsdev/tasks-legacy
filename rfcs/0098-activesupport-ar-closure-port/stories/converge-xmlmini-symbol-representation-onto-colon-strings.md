---
title: "converge-xmlmini-symbol-representation-onto-colon-strings"
status: done
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6919
claim: "2026-08-23T15:12:27Z"
assignee: "converge-xmlmini-symbol-representation-onto-colon-strings"
blocked-by: null
closed-reason: null
---

## Context

`XmlMini.inferTypeName` (`packages/activesupport/src/xml-mini.ts:66-80`) maps a
JS `symbol` value to `type="symbol"`, and `FORMATTING.symbol`
(`xml-mini.ts:417`) reads `value.description`. Both contradict CLAUDE.md's
ratified rule — "A Ruby Symbol is a JS string, never a JS `Symbol`" — so the
`TYPE_NAMES["Symbol"] => "symbol"` arm of
`vendor/rails/activesupport/lib/active_support/xml_mini.rb:29-43` is
unreachable for values in trails' actual Symbol representation, the
colon-prefixed string (`":yes"`).

The read direction is already correct: `PARSING.symbol` (`xml-mini.ts:149-152`)
returns `":yes"` for `<resident type="symbol">yes</resident>`, so `to_xml` and
`from_xml` do not round-trip a Symbol.

Surfaced porting `Hash#to_xml` in #6818: Rails'
`test_one_level_with_types`
(`vendor/rails/activesupport/test/core_ext/hash_ext_test.rb:505-515`) passes
`resident: :yes` and asserts `<resident type="symbol">yes</resident>`. That
term was left out of the trails port of the test rather than building on the
JS-`Symbol` deviation, so the arm is currently uncovered.

## Acceptance criteria

- [ ] `inferTypeName` resolves `type="symbol"` from the colon-prefixed string
      representation, per CLAUDE.md's Symbol rule, and `FORMATTING.symbol`
      emits the name (`.slice(1)`).
- [ ] The JS-`Symbol` arms are removed, not kept alongside.
- [ ] The dropped `resident: :yes` term is restored to
      `hash-ext.test.ts`'s `one level with types` and passes.
- [ ] `to_xml` -> `from_xml` round-trips a Symbol value.
- [ ] `pnpm parity:api:calls` / `:args` green with no new baseline rows.
