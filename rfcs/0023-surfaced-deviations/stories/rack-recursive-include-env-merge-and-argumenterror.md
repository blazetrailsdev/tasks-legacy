---
title: "Rack::Recursive#include merges Rails' six env keys and raises ArgumentError"
status: draft
updated: 2026-08-22
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

`Rack::Recursive#include` (`vendor/rack/lib/rack/recursive.rb:52-64`) merges a
fixed set of CGI keys onto the env before re-dispatching:

```ruby
def include(env, path)
  unless path.index(@script_name) == 0 && (path[@script_name.size] == ?/ ||
                                           path[@script_name.size].nil?)
    raise ArgumentError, "can only include below #{@script_name}, not #{path}"
  end

  env = env.merge(PATH_INFO => path,
                  SCRIPT_NAME => @script_name,
                  REQUEST_METHOD => GET,
                  "CONTENT_LENGTH" => "0", "CONTENT_TYPE" => "",
                  RACK_INPUT => StringIO.new(""))
  @app.call(env)
end
```

The port (`packages/rack/src/recursive.ts:49-65`) instead parses `path` through
`new URL(path, "http://localhost")` and merges only `PATH_INFO`,
`QUERY_STRING`, and `SCRIPT_NAME`. Rails sets no `QUERY_STRING` here at all —
it merges `path` verbatim into `PATH_INFO` — and the port drops all four of
`REQUEST_METHOD`, `CONTENT_LENGTH`, `CONTENT_TYPE`, `RACK_INPUT`, so an included
sub-request keeps the outer request's method and body instead of being reset to
an empty `GET`.

Surfaced by #6889: dropping thrown constructions from the call-argument site
stream freed the pairing slot Ruby's `StringIO.new("")` had been consuming with
the port's `throw new Error(...)` guard, and it now pairs with `new URL(...)` —
baselined in `scripts/api-compare/call-mismatches-exclude/rack/recursive.json`
(`kind: "args"`, `rubyName: "include"`, `rubyArgs: ["str:"]`).

The guard itself also deviates: Rails raises `ArgumentError`
(`recursive.rb:55`), the port throws a bare `Error` (`recursive.ts:56`), and
Rails' prefix check is `path.index(@script_name) == 0 && (path[size] == ?/ ||
path[size].nil?)` rather than the port's `startsWith`/equality pair.

## Converged shape

`include(env, path)` merges exactly Rails' six keys onto the env — `PATH_INFO`
set to `path` verbatim, `SCRIPT_NAME`, `REQUEST_METHOD => GET`,
`"CONTENT_LENGTH" => "0"`, `"CONTENT_TYPE" => ""`, and `RACK_INPUT` set to an
empty body — with no `new URL` parse and no `QUERY_STRING` write, and raises
`ArgumentError` with Rails' message.

## Acceptance criteria

- `recursive.ts#include` mirrors `recursive.rb:52-64` key-for-key, including the
  error class and message.
- The `kind: "args"` row for `recursive.ts` `include` is deleted by hand from
  `scripts/api-compare/call-mismatches-exclude/rack/recursive.json` (only-shrink;
  no `--write`, no reseed).
- `pnpm parity:api:calls:args` and `pnpm parity:api:calls` green.
- Covered against `vendor/rack/test/spec_recursive.rb`.
