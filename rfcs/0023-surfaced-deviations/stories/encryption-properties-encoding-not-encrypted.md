---
title: "Properties has an invented `encrypted` accessor where Rails has `encoding`"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Properties has an invented `encrypted` accessor where Rails has `encoding`

## Context

Surfaced while adding the `compressed` accessor in PR #6719 (RFC 0106 wave 4c,
encryption slice), which required reading `DEFAULT_PROPERTIES` in full.

`vendor/rails/activerecord/lib/active_record/encryption/properties.rb:22-40`
generates one reader/writer pair per entry of:

```ruby
DEFAULT_PROPERTIES = {
  encrypted_data_key: "k",
  encrypted_data_key_id: "i",
  compressed: "c",
  iv: "iv",
  auth_tag: "at",
  encoding: "e"
}
```

`packages/activerecord/src/encryption/properties.ts:78-88` has an accessor pair
named **`encrypted`** over key `"e"`. That is not a Rails member: `"e"` is
`encoding`. So the port simultaneously

- **invents** `encrypted` (a name a Rails dev reads as "is this encrypted?",
  which is `EncryptableRecord#encrypted_attribute?` / `Type#encrypted?` — a
  different concept entirely), and
- **omits** `encoding`, the header Rails writes to carry a payload's string
  encoding (`encrypted_attribute_type.rb` reads it on the deserialize path).

The `encrypted` writer also hand-rolls its own already-set check
(`properties.ts:82-87`) instead of going through the shared `set`, duplicating
the `EncryptedContentIntegrity` raise that `properties.rb:49-53` keeps in one
place.

## Converged shape

- Rename the pair to `encoding`, keeping key `"e"`, so the accessor set is
  exactly `DEFAULT_PROPERTIES` in `properties.rb:23-30` order (k, i, c, iv, at,
  e).
- Route the writer through the shared `set`, deleting the duplicated
  already-set guard so the raise site stays single (`properties.rb:49-53`).
- Update every caller — grep `\.encrypted\b` under
  `packages/activerecord/src/encryption/`. Some may genuinely want the
  `encoding` semantics; any that wanted "is encrypted" is a separate bug this
  rename will expose, and should be fixed, not preserved.
- Once `encoding` exists, check whether the deserialize path in
  `encrypted-attribute-type.ts` should be reading it the way Rails does.

Related but distinct: `encryption-properties-js-collection-surface` covers the
`get`/`set`/`has`/`entries`/`size`/`toJSON` surface, not the
`DEFAULT_PROPERTIES` membership this story is about.

## Acceptance criteria

- [ ] `properties.ts`'s generated-accessor set matches `DEFAULT_PROPERTIES`
      exactly — `encoding` present, `encrypted` gone, no other additions.
- [ ] The `encrypted` writer's bespoke already-set guard is deleted; the raise
      comes from `set`.
- [ ] Every former `.encrypted` caller is resolved to `encoding` or to the
      correct is-encrypted predicate, with the Rails line cited at the call site
      where the answer was not mechanical.
- [ ] `pnpm parity:api:extra --package activerecord` shows one fewer novel name
      for `encryption/properties.ts`.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative; the encryption
      suites and the message-serializer round-trip tests green on all three
      adapter lanes.
