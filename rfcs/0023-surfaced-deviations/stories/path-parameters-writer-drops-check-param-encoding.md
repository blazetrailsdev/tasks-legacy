---
title: "path_parameters= drops Rails' check_param_encoding"
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

`vendor/rails/actionpack/lib/action_dispatch/http/parameters.rb:67-77` —
`path_parameters=` does two things trails omits:

```ruby
parameters = Request::Utils.set_binary_encoding(self, parameters, parameters[:controller], parameters[:action])
Request::Utils.check_param_encoding(parameters)
```

trails' `set pathParameters` in
`packages/actionpack/src/action-dispatch/http/parameters.ts` skips both, with
a call-site comment arguing JS strings are UTF-16 so there is nothing to
coerce. That justifies dropping `set_binary_encoding`, but NOT
`check_param_encoding` — Rails raises there so an invalid encoding surfaces at
the setter instead of triggering errors further downstream. trails silently
accepts anything.

`RequestUtils` already exists at
`packages/actionpack/src/action-dispatch/request/utils.ts` (trails already
calls `normalizeEncodeParams` from `Request#requestParameters`), so check
whether a `checkParamEncoding` counterpart is ported there before adding one.

Deviation surfaced during PR #5407 (writer→accessor convergence); the omission
predates that PR, which only moved the code.

## Acceptance criteria

- Decide, with the Rails source cited, whether `check_param_encoding` has
  meaningful semantics in a UTF-16 runtime. If it does, port it and call it
  from `set pathParameters`.
- If it genuinely has no JS analogue, replace the call-site comment with one
  that justifies each omitted call separately rather than covering both with
  the UTF-16 argument.
- `set_binary_encoding` stays omitted either way; record why at the call site.
