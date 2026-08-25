---
title: "GID URI query keys are camelCase where Rails emits snake_case"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #5951 converged `SignedGlobalID.create` onto Rails' inherited
`GlobalID.create`, whose param filter is
`options.except(:app, :verifier, :for)`
(`vendor/globalid/lib/global_id/global_id.rb:14`). Every other option key
therefore lands in the GID URI query — including SGID's expiration options,
which is exactly what Rails does.

Because trails spells options camelCase (CLAUDE.md), the emitted query key is
the camelCase one: `SignedGlobalID.create(person, { expiresIn: 60 })` produces
`gid://bcx/Person/5?expiresIn=60` where Ruby produces
`gid://bcx/Person/5?expires_in=60`. Unlike a method name, a URI query key is
**wire data**: it is embedded in the signed SGID payload and in any GID string
persisted or exchanged with a Ruby GlobalID peer, so a trails-issued GID does
not round-trip through Rails' `URI::GID` with the same params (Rails reads
`params["expires_in"]`, ours carries `expiresIn`).

Same question applies to arbitrary user-supplied param keys, which pass through
verbatim in both languages and so are NOT affected — only keys that trails
itself renames (the SGID option bag) diverge.

Decide whether GID URI param keys should be underscored on the way out
(`packages/globalid/src/global-id.ts`, `GlobalID.create`, and the parse side in
`packages/globalid/src/uri/gid.ts`), or whether the camelCase spelling is the
accepted trails wire format. Either way the decision should be recorded at the
call site rather than left implicit.

## Acceptance criteria

- A decision is made and justified against `vendor/globalid` on whether
  trails-renamed option keys are underscored in the GID query string.
- If converging: `SignedGlobalID.create(person, { expiresIn: 60 })` emits
  `?expires_in=60`, and the parse side maps it back; a test pins the round-trip.
- If keeping camelCase: the deviation is documented at the filtering site in
  `GlobalID.create` with the reason, and a test pins the emitted spelling so it
  can't drift silently.
- globalid tests pass; test names unchanged.
