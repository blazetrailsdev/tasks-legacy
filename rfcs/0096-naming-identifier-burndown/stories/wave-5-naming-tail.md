---
title: "Wave 5: burn down the AR-closure naming tail — encryption, activemodel, arel, globalid, test-support"
status: claimed
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: api-compare
packages: ["activerecord", "activemodel", "arel", "globalid"]
deps: []
deps-rfc: []
est-loc: 70
priority: 15
pr: null
claim: "2026-08-23T14:27:28Z"
assignee: "resolve-serialization-thenable-hash-async-return"
blocked-by: null
closed-reason: null
---

## Context

RFC 0096's closing story `naming-gate-flip` is blocked on the AR require-closure
reaching **zero convergeable `naming` rows** (`burndown` +
`module-mixin-receiver`). Waves 1-4 drained the population it was scoped
against; a fresh reading of
`scripts/api-compare/output/call-arg-mismatches.json` (artifact of 2026-08-21,
rendered by `pnpm parity:api:calls:args:report`) shows **107 convergeable rows
still standing inside the closure**, across 67 files. This wave-5 band splits
those 107 into six non-overlapping file sets so the flip has a defined finish
line again.

Rules are unchanged from the RFC's `## Design`:

- **Rename to the Rails identifier, not to a better one.** If Rails writes `o`,
  the TS local is `o`, camelCased per `docs/ruby-ts-conventions.md`.
- **Body-local only.** No behavior change, no public surface change.
- **A row that is really an a1 (argument order) or a3 (invented helper /
  conversion) finding is NOT renamed away.** File it against the RFC that owns
  the file and leave the row. Several rows below look like that shape — e.g.
  `activesupport/notifications.ts#instrument` passes a different argument list,
  not a differently-named one — so read each pair against the vendored Ruby
  before renaming.
- `module-mixin-receiver` rows converge by rewiring to the `this`-typed mixin
  idiom (CLAUDE.md, Module mixins), not by renaming the parameter.

## Rows in this slot

12 rows across 11 files. **File set:** `activerecord/encryption/**`, `activemodel/`, `arel/`, `globalid/`, `activerecord-test-support/` only.

- `activerecord-test-support/schema-dumping-helper.ts` — 2
  - `dump_table_schema`: ruby `ref:pool` vs ts `ref:source`
  - `dump_all_table_schema`: ruby `ref:pool` vs ts `ref:source`
- `arel/nodes/function.ts` — 1
  - `initialize`: ruby `ref:aliaz` vs ts `ref:aliasNode`
- `activerecord/encryption/cipher/aes256-gcm.ts` — 1
  - `encrypt`: ruby `ref:cipher,ref:clearText` vs ts `ref:deterministic,ref:inputBuf`
- `activerecord/encryption/encryptor.ts` — 1
  - `encrypt`: ruby `ref:clearText,kwargs{cipherOptions=ref:cipherOptions,keyProvider=ref:keyProvider}` vs ts `ref:text,kwargs{cipherOptions=ref:cipherOptions,keyProvider=ref:keyProvider}`
- `activerecord/encryption/key-generator.ts` — 1
  - `derive_key_from`: ruby `ref:keyDerivationSalt,ref:length` vs ts `ref:salt,ref:length`
- `activerecord/encryption/message-pack-message-serializer.ts` — 1
  - `load`: ruby `ref:serializedContent` vs ts `ref:from`
- `activemodel/attribute-set.ts` — 1
  - `deep_dup`: ruby `ref:transformValues` vs ts `ref:newAttrs`
- `activemodel/attribute.ts` — 1
  - `value_for_database`: ruby `ref:valueForDatabase,ref:value` vs ts `ref:_cachedValueForDatabase,ref:value`
- `activemodel/serialization.ts` — 1
  - `serializable_attributes`: ruby `ref:n` vs ts `ref:key`
- `activemodel/validations/numericality.ts` — 1
  - `parse_as_number`: ruby `ref:Float,ref:precision,ref:scale` vs ts `ref:float,ref:precision,ref:scale`
- `globalid/locator.ts` — 1
  - `find_records`: ruby `ref:modelClass` vs ts `ref:klass`

## Acceptance criteria

1. Every convergeable (`burndown` / `module-mixin-receiver`) `naming` row in the
   file set above is either converged to the Rails identifier, rewired to the
   `this`-typed mixin idiom, or re-filed as an a1/a3 finding against the RFC
   that owns the file — with the re-filed story id named in the PR body.
2. `pnpm parity:api:calls:args:report` shows this slot's convergeable count at
   **zero**; no row in the file set is added to any
   `call-mismatches-exclude/` shard (CLAUDE.md — converge, never ratify).
3. No public surface, method name, field name or behavior changes; the diff is
   locals and parameters only (plus mixin-receiver rewiring where it applies).
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls:args` stays green.

## Notes for the claimer

The per-file counts above are from the 2026-08-21 parity artifact and are
**advisory**. Re-run
`API_COMPARE_FORCE=1 pnpm parity:api --calls && pnpm parity:api:calls:args:report`
at claim time and work from the fresh reading — counts drift as sibling RFCs
touch the same files.
