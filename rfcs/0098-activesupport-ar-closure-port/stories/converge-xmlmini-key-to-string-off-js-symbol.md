---
title: "Converge XmlMini keyToString off the JS-Symbol arm"
status: claimed
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: "2026-08-23T15:42:31Z"
assignee: "excluding-must-drain-a-scheduled-relation"
blocked-by: null
closed-reason: null
---

## Context

`keyToString` (`packages/activesupport/src/xml-mini.ts:463-466`) still models a
Ruby Symbol **key** as a JS `Symbol`:

```ts
function keyToString(key: unknown): string {
  return typeof key === "symbol" ? (key.description ?? "") : String(key);
}
```

This is the key-side twin of the value-side deviation converged in #6919, and
contradicts CLAUDE.md's ratified rule — "A Ruby Symbol is a JS string, never a
JS `Symbol`". Rails' `to_tag` simply calls `key.to_s`
(`vendor/rails/activesupport/lib/active_support/xml_mini.rb:118-146`), and every
Hash key in trails is already a string, so the `Symbol` arm is dead code that
also teaches the wrong representation to anyone reading the file.

Note the asymmetry with the value side: a Symbol _key_ renders as its bare name
in Rails (`{ resident: :yes }.to_xml` gives `<resident ...>`), so the converged
shape is `String(key)` with no colon stripping — the colon is not part of a key
in trails because keys are plain strings, never `":resident"`.

## Acceptance criteria

- [ ] `keyToString`'s JS-`Symbol` arm is removed (or the helper collapses into
      the `key.to_s` call site if that is the closer mirror of xml_mini.rb:118).
- [ ] `xml-mini.test.ts` / `hash-ext.test.ts` remain green, including
      `#to_tag should dasherize the space when passed a symbol with spaces as a key`
      (`vendor/rails/activesupport/test/xml_mini_test.rb:172`).
- [ ] No other JS-`Symbol` arm survives in `xml-mini.ts`.
