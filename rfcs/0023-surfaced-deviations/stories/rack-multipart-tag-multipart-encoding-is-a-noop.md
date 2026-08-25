---
title: "Port Rack's tag_multipart_encoding / find_encoding instead of no-op stubs"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "rack"
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

`Rack::Multipart::Parser#tag_multipart_encoding` (`vendor/rack/lib/rack/multipart/parser.rb:456-487`)
forces UTF-8 on the part name, then — only for non-file parts — parses the
`content_type` parameter list, and for `text/plain` reads its `charset` param
through `find_encoding` (`parser.rb:489-493`, `Encoding.find enc` with an
`ArgumentError` rescue to `Encoding::BINARY`) and force-encodes the body with it.

trails ports both as no-ops:

```ts
private tagMultipartEncoding(_filename, _contentType, _name, _body) {}
private findEncoding(enc) { return enc ?? "UTF-8"; }
```

(`packages/rack/src/multipart/parser.ts`, bottom of the class). The empty body
is why `handle_mime_head -> find_encoding` sits in
`scripts/api-compare/call-mismatches-exclude/rack/multipart/parser.json` — the
ported `handleMimeHead`/`result` path never reaches the charset arm at all.

The trails test `it("sets BINARY encoding on things without content type")`
(parser.test.ts) currently only asserts the parsed value, because there is no
encoding to assert on.

## Acceptance criteria

- [ ] `tagMultipartEncoding` ports parser.rb:456-487 branch for branch: the
      `return if filename` guard, the `content_type.split(';')` list, the
      `TEXT_PLAIN == type_subtype` arm, the quoted-value strip, and the
      `charset` key.
- [ ] `findEncoding` mirrors parser.rb:489-493 including the fallback arm.
- [ ] The `handle_mime_head -> find_encoding` row is DELETED from
      `call-mismatches-exclude/rack/multipart/parser.json` (only-shrink, no
      reseed), and `parity:api:calls:tighten` run for the stale mark.
