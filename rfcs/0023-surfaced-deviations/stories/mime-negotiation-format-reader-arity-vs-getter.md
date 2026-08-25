---
title: "mime-negotiation-format-reader-arity-vs-getter"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the MimeNegotiation writers onto Rails-named
accessors (PR #5385), and it is the reason the reader half could not move onto
the class module as a `get` accessor.

Rails' reader takes an optional argument —
`vendor/rails/actionpack/lib/action_dispatch/http/mime_negotiation.rb:63`:

```ruby
def format(_view_path = nil)
  formats.first || Mime::NullType.instance
end
```

trails is split on this:

- `packages/actionpack/src/action-dispatch/http/mime-negotiation.ts` keeps the
  module function `format(this, _viewPath?)`, which matches Rails' arity.
- `packages/actionpack/src/action-dispatch/http/request.ts` exposes `format` as
  a `get`/`set` accessor pair on the class, so `request.format(viewPath)` is not
  callable — only `request.format` is.

Rails ignores the argument entirely (the parameter is underscore-prefixed), so
today this is a signature deviation rather than a behaviour one. It matters
because it blocks the accessor-pair shape: a TS getter cannot take a parameter,
so as long as the Rails reader is arity-1 the reader and writer cannot share one
prototype slot in the module file.

Decide and record which way this converges: either accept the getter and note
the dropped argument as an intentional deviation, or keep `format` a method on
`Request` and find another home for the `format=` writer.

## Acceptance criteria

- A single decision applied consistently across `mime-negotiation.ts` and
  `request.ts` — the reader is either a method (Rails arity) in both, or a
  getter in both.
- If the argument is dropped, the deviation is justified at the call site per
  CLAUDE.md, not only in a PR body.
- `pnpm parity:api` arity for `format` is either clean or has a reasoned
  `arity-exclude.json` entry.
