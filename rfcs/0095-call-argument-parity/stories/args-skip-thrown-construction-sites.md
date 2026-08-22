---
title: "Stop pairing throw new Error guards with Ruby constructors in the args gate"
status: done
updated: 2026-08-22
rfc: "0095-call-argument-parity"
cluster: api-compare
packages: ["activerecord", "arel"]
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6889
claim: "2026-08-22T23:01:35Z"
assignee: "args-skip-thrown-construction-sites"
blocked-by: null
closed-reason: null
---

## Context

Twelve `kind: "args"` shape rows are the same tooling artifact: a TypeScript
`throw new Error("…")` guard paired against an unrelated Ruby constructor,
because both normalize to a call named `new`.

| Package        | TS file                                  | Ruby method                                      | Rails actually passes     |
| -------------- | ---------------------------------------- | ------------------------------------------------ | ------------------------- |
| arel           | `nodes/binary.ts`                        | `to_cte`                                         | `name, right`             |
| activerecord   | `connection-adapters/sqlite3-adapter.ts` | `initialize`                                     | `connection_pool: pool`   |
| activerecord   | `encryption/cipher/aes256-gcm.ts`        | `encrypt`                                        | `CIPHER_TYPE`             |
| activerecord   | `encryption/cipher/aes256-gcm.ts`        | `decrypt`                                        | `CIPHER_TYPE`             |
| activerecord   | `encryption/config.ts`                   | `support_sha1_for_non_deterministic_encryption=` | `hash_digest_class: SHA1` |
| activerecord   | `migration/command-recorder.ts`          | `change_table`                                   | `delegate`                |
| activerecord   | `migration/command-recorder.ts`          | `invert_transaction`                             | `delegate`                |
| activesupport  | `testing/deprecation.ts`                 | `assert_deprecated`                              | `escape`                  |
| actiondispatch | `journey/parser.ts`                      | `parse_group`                                    | `node`                    |
| actiondispatch | `system-testing/driver.ts`               | `register_cuprite`                               | `app, merge`              |
| rack           | `lint.ts`                                | `call`                                           | `app, env`                |
| rack           | `multipart/parser.ts`                    | `initialize`                                     | `dup`                     |

In every one the TS side is a one-argument string — the guard message — and the
Ruby side is a real instantiation somewhere else in the body. Nothing is wrong
with either body; the sites should never have been paired.

## The mechanism, and why the fix is small

The extractor already knows about this shape.
`extract-ts-api.ts#isThrownConstruction` (`:2842`) identifies a `new` that is
the direct operand of a `throw`, and its doc comment states the reasoning
exactly:

> `throw new Foo(msg)` — the operand of a `throw`, which Rails spells EITHER
> `raise Foo, msg` (no `.new` call, and `raise` itself is filtered as noise on
> the Ruby side) or `raise Foo.new(msg)` (a `new` call in that position). The
> position is therefore ambiguous […]

That reasoning is applied to the call SEQUENCE stream at `:3044`, which drops
the name rather than pinning `constructor` at the front of the order (the
`order:constructor,…` rows #6404 baselined). It is **not** applied to the
call-ARGUMENT site stream, which pushes every `new` unconditionally
(`extract-ts-api.ts:2820`). So the args gate inherits precisely the defect the
order stream was already fixed for.

Because Rails may spell the raise with no `new` call at all, the Ruby side
often has no counterpart site — which is why the TS guard pairs with whatever
other instantiation the Ruby body makes.

## Two ways to fix it, and the tradeoff

1. **Flag it.** Emit a flag (`thrownConstruction`) on the site and add it to
   `UNCOMPARABLE_FLAGS` in `call-args.ts`, so it skips at `:1180` alongside the
   existing `uncomparableFlag` reason. Minimal, and reuses the skip taxonomy
   the report already prints.
2. **Drop it**, exactly as the order stream does at `:3044`. This is the more
   faithful option: keeping the site (even as uncomparable) leaves it occupying
   a pairing slot, so a Ruby `Cipher.new(CIPHER_TYPE)` still consumes the
   thrown guard rather than pairing with the real TS construction later in the
   body — the row disappears, but the comparison it should have made never
   happens.

Option 2 is preferred unless it destabilises pairing for bodies that both throw
and construct; the aes256-gcm and command-recorder rows are the test cases,
since each body does both.

## Acceptance criteria

- All twelve rows stop flagging, and the skip taxonomy accounts for them
  (either a new skip reason in the report or the existing `uncomparableFlag`).
- The corresponding `kind: "args"` entries are **deleted by hand** from
  `scripts/api-compare/call-mismatches-exclude/**` — fixing the tooling makes
  them stale and the only-shrink gate will go red until they are removed. No
  `--write`, no reseed.
- A regression test in `scripts/api-compare/call-args.test.ts` covering a body
  that both throws and constructs, asserting the real construction still pairs.
- `pnpm parity:api:calls:args` green; `pnpm parity:api:calls` unaffected (the
  call SET is unchanged — whichever way Rails spells the raise, an
  instantiation happens).

## Related

The sibling classifier gap is `args-dl-to-fs-receiver-shape` (RFC 0099): the
receiver-as-first-argument shape is recognised by `receiver-as-first-arg.ts`
and `naming-taxonomy.ts` but not consulted by the args comparison. Same class
of defect — a known idiom the args gate alone does not know about — and worth
solving with the same mechanism if one lands first.
