---
title: "CookieJar.parse is invented surface shadowing three Rails parse methods"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
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

Surfaced by PR #6665 (RFC 0108 foreign-receiver call marking) and baselined
there with a citation
(`call-mismatches-exclude/actiondispatch/middleware/cookies.json`, `parse` →
`load`). The row exists because the call gate paired Rails'
`SerializedCookieJars#parse` with a TS `parse` that is not its counterpart at
all.

Rails declares `parse` three times in
`actionpack/lib/action_dispatch/middleware/cookies.rb` — the no-op
`AbstractCookieJar#parse` (`:554`), `SerializedCookieJars#parse` (`:594-606`,
which does `serializer.load(dumped)` and the `reserialize?` re-write), and the
`SignedCookieJar` / `EncryptedCookieJar` overrides (`:635-639`, `:685-691`).
None of them parses a `Cookie:` header.

trails
(`packages/actionpack/src/action-dispatch/middleware/cookies.ts:264-273`)
declares a `static parse(cookieHeader, options): CookieJar` on `CookieJar` that
splits a raw `Cookie:` header on `;` and `=`. That is invented surface: Rails
builds the jar from the request's already-parsed `cookies` hash
(`Request#cookies` → `Rack::Request#cookies`), never from a header string, and
nothing in Rails' cookies.rb is named `parse` for that job. Because it carries
the Rails NAME of three real methods, the gate pairs a Ruby `parse` body
against it and reports every call the real body makes as missing.

## Converged shape

- Either delete `CookieJar.parse` and have callers take the parsed cookie hash
  Rack already produces, or — if a header splitter is genuinely needed at that
  seam — give it a name Rails does not use for a different method and tag it
  `@noRailsEquivalent <reason>`, so the gate stops pairing a Ruby `parse` body
  against it.
- With the collision gone, port `SerializedCookieJars#parse` (`cookies.rb:594-606`)
  under the Rails name if it is not already covered, and delete the
  `parse` → `load` row from
  `scripts/api-compare/call-mismatches-exclude/actiondispatch/middleware/cookies.json`
  by hand (only-shrink; do not reseed), then
  `pnpm parity:api:calls:tighten actiondispatch/middleware/cookies.json`.

## Acceptance criteria

- No TS member named `parse` in cookies.ts stands in for a Ruby `parse` it does
  not port; `pnpm parity:api:extra --package actionpack` reports no untagged
  extra surface for it.
- The `parse` → `load` baseline row is deleted, not reworded.
- Existing cookies tests stay green (no test renames).
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
