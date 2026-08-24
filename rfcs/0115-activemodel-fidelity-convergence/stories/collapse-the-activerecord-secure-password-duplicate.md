---
title: "collapse-the-activerecord-secure-password-duplicate"
status: done
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6959
claim: "2026-08-23T22:40:29Z"
assignee: "collapse-the-activerecord-secure-password-duplicate"
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/secure-password.ts` is a SECOND, independent
implementation of `has_secure_password`, parallel to the ported one in
`packages/activemodel/src/secure-password.ts`. Rails has exactly one:
`activemodel/lib/active_model/secure_password.rb`; ActiveRecord adds only
`generates_token_for` wiring (`secure_password.rb:159-178`) and
`authenticate_by` (`activerecord/lib/active_record/secure_password.rb`).

The AR copy diverges from the Ruby in ways the ActiveModel copy does not:

- It hashes the password in a `beforeSave` callback
  (`secure-password.ts:182-200`), not in the generated `password=` writer.
  Rails hashes eagerly in the setter (`secure_password.rb:186-194`), so
  `user.password = "x"; user.password_digest` is populated before save in
  Rails and `null` here.
- It nulls out the raw password, confirmation and challenge inside that
  callback (`:194-197`) — Rails keeps all three for the record's life.
- Its digest is a bespoke `salt:hash` hex string with an `indexOf(":")` salt
  reader (`:129-140`), not BCrypt, so `password_salt` returns something no
  Rails dev would recognize.
- Reads and writes go through `_readAttribute` / `writeAttribute` rather than
  `public_send("#{attribute}_digest")` — the divergence PR #6941's follow-up
  converged in the ActiveModel copy.

## Converged shape

Delete the AR implementation of the shared surface and have ActiveRecord's
`hasSecurePassword` call ActiveModel's, keeping only what
`activerecord/lib/active_record/secure_password.rb` actually adds:
`authenticate_by` and the `generates_token_for` reset-token wiring (the
`resetToken` option the ActiveModel port already threads through).

## Acceptance criteria

- [ ] One `has_secure_password` implementation, in `activemodel`.
- [ ] The digest is BCrypt and is written by the `password=` writer, not a
      `beforeSave` callback.
- [ ] The plaintext, confirmation and challenge survive a save.
- [ ] `packages/activerecord/src/secure-password.test.ts` and
      `token-for.test.ts` green with no test renamed.
