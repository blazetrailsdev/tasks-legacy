---
title: "GlobalID#uri is the string form, not Rails' URI::GID object"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: wrapper-vs-subclass
packages: []
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

Rails' `attr_reader :uri` on GlobalID holds the **URI::GID object**
(`vendor/globalid/lib/global_id/global_id.rb:43`), and `app`, `model_name`,
`model_id`, `params`, `to_s` and `deconstruct_keys` are all
`delegate ... to: :uri` (`global_id.rb:44`).

trails' `GlobalID.uri` is the **string** form
(`packages/globalid/src/global-id.ts:39`). PR #6221 made the readers real
delegations by holding the parsed `GID` in a private `_uri` field alongside it,
but that leaves two fields for Rails' one, and the public `uri` still has the
wrong type. Every trails caller reads `uri` as a string:
`global-id.ts` (`toParam`, `toString`, `equals`), `signed-global-id.ts:85,197,222`,
`locator.ts:304,318`, plus assertions in `global-id.test.ts:141-142`,
`global-identification.test.ts`, `signed-global-id.test.ts`.

Note Rails' own test asserts the object form:
`vendor/globalid/test/cases/global_id_test.rb:163-168` —
`assert_equal URI('gid://bcx/Person/5'), @person_gid.uri`.

## Converged shape

`readonly uri: GID`, with the private `_uri` field deleted and the delegated
readers going through `this.uri` directly. Call sites that want the string move
to `this.uri.toString()` / `gid.toString()`. `global-id.test.ts`'s "as URI" test
converges onto comparing GID objects, mirroring `global_id_test.rb:163-168`.

## Acceptance criteria

- [ ] `GlobalID#uri` is the parsed `GID`, not its string form; no second field.
- [ ] Delegated readers (`app`, `modelName`, `modelId`, `params`,
      `deconstructKeys`) go through the public `uri`.
- [ ] All string call sites above updated; globalid suite green.
- [ ] `pnpm parity:api --package globalid` stays 80/80.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
