---
title: "Request#get_header must not apply the Headers#[] name conversion"
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

# `Request#get_header` must not apply the `Headers#[]` name conversion

## Context

Surfaced in PR #6667 while converging the Rack header-accessor cluster onto
`getHeader` / `setHeader` / `hasHeader` / `fetchHeader`.

Rack's reader is a bare env lookup — no name mapping at all
(`rack/lib/rack/request.rb:100-102`):

```ruby
def get_header(name)
  @env[name]
end
```

The HTTP-name mapping (`"Content-Type"` → `"CONTENT_TYPE"`, `"If-None-Match"` →
`"HTTP_IF_NONE_MATCH"`) belongs to a different object,
`ActionDispatch::Http::Headers#env_name`
(`actionpack/lib/action_dispatch/http/headers.rb:96-102`), reached through
`request.headers[…]`.

trails folds the two together
(`packages/actionpack/src/action-dispatch/http/request.ts`):

```ts
getHeader(name: string): any {
  return this.env[envName(name)];
}
```

`hasHeader` / `setHeader` / `deleteHeader` / `fetchHeader` on the same class
correctly do NOT convert, so the four accessors disagree about what a name
means: `getHeader("Content-Type")` reads `CONTENT_TYPE` while
`hasHeader("Content-Type")` is false for the same request.

This is currently harmless for the ported bodies — every key they pass is
already a CGI/Rack name (underscored or dotted), which `envName` returns
unchanged — but it is live for the ~20 call sites that pass HTTP-style names
(`request.getHeader("if-none-match")`, `getHeader("referer")`,
`getHeader("user-agent")` in `action-controller/base.ts`,
`metal/conditional-get.ts`, `action-controller/test-case.ts`).

## Converged shape

`getHeader` becomes the bare `this.env[name]` of rack/request.rb:100-102, and
the call sites that want the HTTP-name mapping go through `request.headers[…]`
(`Headers#[]`, headers.rb:57-59) — which is what Rails' own callers do.

## Acceptance criteria

- [ ] `getHeader` performs no name conversion; the four env accessors agree on
      what a name means.
- [ ] Every caller passing an HTTP-style name is moved to `headers[…]`.
- [ ] `pnpm parity:api:calls` / `:args` green, actionpack suite green.
