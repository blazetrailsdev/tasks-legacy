---
title: "Converge the encryption decorator's column default onto columns_hash"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' encryption decorator resolves the column default with one expression
(`vendor/rails/activerecord/lib/active_record/encryption/encryptable_record.rb:91`):

```ruby
ActiveRecord::Encryption::EncryptedAttributeType.new(
  scheme: scheme, cast_type: cast_type, default: columns_hash[name.to_s]&.default)
```

trails routes that through two helpers with no Rails counterpart, in
`packages/activerecord/src/encryption/encryptable-record.ts`:

- `EncryptableRecord.columnDefaultFor(modelClass, name, def)` — peeks a cache,
  else falls back to the attribute definition's `defaultValue`.
- `EncryptableRecord.cachedColumnDefaultFor(modelClass, name)` — a query-free,
  connection-less peek at the raw `SchemaCache` (`internalSchemaCache`, else
  `connectionPool().poolConfig.schemaCache`), wrapped in a bare `try/catch`
  returning a `NOT_CACHED` sentinel.

And `pushEncryptionDecorator` hands `columnDefaultFor` a raw
`modelClass._attributeDefinitions.get(attrName)` entry, which Rails never reads
here.

The stated reason for the cache peek is that `columnsHash`/`loadSchema` warm the
shared schema cache and perturb sibling encryption tests — a test-isolation
concern, not a TypeScript language shortcoming, so it is debt rather than a
ratifiable deviation.

## Converged shape

The decorator body reads `columnsHash[name]?.default` (Rails' `columns_hash`)
directly, and `columnDefaultFor` / `cachedColumnDefaultFor` / the `NOT_CACHED`
sentinel / the `_attributeDefinitions` peek all come out. If a sibling test is
perturbed by the schema load Rails also performs, fix the test's isolation
rather than keeping the bespoke reader.

## Acceptance criteria

- `columnDefaultFor` and `cachedColumnDefaultFor` are deleted from
  `EncryptableRecord`, along with the `NOT_CACHED` sentinel.
- `pushEncryptionDecorator` threads `default:` from `columnsHash`, mirroring
  encryptable_record.rb:91, and no longer reads `_attributeDefinitions`.
- `pnpm parity:api:extra --package activerecord` shows two fewer novel names in
  `encryption/encryptable-record.ts`.
- The `packages/activerecord/src/encryption` suite stays green, including the
  column-default coverage in `encryptable-record.test.ts`.
