---
title: "Port ActionDispatch::Response::FileBody so send_file assigns it"
status: draft
updated: 2026-08-12
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

`ActionDispatch::Response#send_file` is `commit!` then
`@stream = FileBody.new(path)`
(`vendor/rails/actionpack/lib/action_dispatch/http/response.rb:374-377`).
`FileBody` is a real class (response.rb, `class FileBody`), with `to_path`,
`each` and `close`. trails has no ported `FileBody`, so
`packages/actionpack/src/action-dispatch/http/response.ts:398-415` assigns an
inline object literal with `toPath`/`body`/`each`. PR #6431 added the
`send_file / new` call-set baseline row for it.

## Converged shape

Port `FileBody` as a class in `http/response.ts` at Rails' position, and
`sendFile` assigns `new FileBody(path)`. The baseline row is then deleted by
hand (only-shrink).

## Acceptance criteria

- [ ] `FileBody` exists with Rails' members and `parity:api` counts it.
- [ ] `sendFile` matches response.rb:374-377 line for line.
- [ ] The `send_file / new` row is deleted; `pnpm parity:api:calls` green.
