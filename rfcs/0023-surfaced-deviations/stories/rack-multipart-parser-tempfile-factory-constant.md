---
title: "Port Rack's Parser::TEMPFILE_FACTORY and drop the nullable tmpfile arm"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "rack"
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

## Context

Rack defines a default tempfile factory as a constant on the parser:

```ruby
TEMPFILE_FACTORY = lambda { |filename, content_type|
  extension = ::File.extname(filename.gsub("\0", '%00'))[0, 129]

  Tempfile.new(["RackMultipart", extension])
}
```

(`vendor/rack/lib/rack/multipart/parser.rb:41-45`), and `Rack::Multipart.parse_multipart`
falls back to it: `tempfile = env[RACK_MULTIPART_TEMPFILE_FACTORY] || Parser::TEMPFILE_FACTORY`
(`vendor/rack/lib/rack/multipart.rb:59`).

trails has no `TEMPFILE_FACTORY`. `Parser.parse` / the `Parser` constructor take
`tmpfile: ((filename: string, contentType: string) => any) | null`
(`packages/rack/src/multipart/parser.ts:311-338`), and `Collector#onMimeHead`
has to non-null-assert it (`this.tempfile!(filename, contentType ?? "")`,
parser.ts:~236) — Rails' `@tempfile.call` is unconditional because a factory is
always present. The nullable parameter is what forced
`packages/rack/src/multipart/parser.test.ts` to hand-roll a tempfile double as
its default; Rack's own tests get the constant.

`packages/rack/src/multipart.ts` also never routes through `Parser` — it builds
tempfiles inline at multipart.ts:271-286 — so the fallback chain at
multipart.rb:59 has no counterpart either.

## Acceptance criteria

- [ ] `Parser.TEMPFILE_FACTORY` exists and matches parser.rb:41-45 (extension
      from the filename with NUL escaped, truncated to 129 bytes).
- [ ] The `tmpfile` parameter loses its `| null` arm and `onMimeHead` calls
      `this.tempfile(...)` without a non-null assertion, as parser.rb:155 does.
- [ ] The parser test uses the constant instead of a hand-rolled double.
- [ ] `pnpm parity:api:calls` / `parity:api:calls:args` stay green.
