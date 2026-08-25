---
title: "Rack::Logger#call passes env to compute_tags where Rails passes a Request"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
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

`Rails::Rack::Logger#call` builds an `ActionDispatch::Request` from the raw env
and passes that request to both `compute_tags` and `call_app`
(`railties/lib/rails/rack/logger.rb:20-30`):

```ruby
def call(env)
  request = ActionDispatch::Request.new(env)

  env["rails.rack_logger_tag_count"] = if logger.respond_to?(:push_tags)
    logger.push_tags(*compute_tags(request)).size
  else
    0
  end

  call_app(request, env)
end
```

The port (`packages/trailties/src/rack/logger.ts:42,81`) never builds the
request and threads the raw `env` instead:

```ts
? this.logger.pushTags(...this.computeTags(env)).length
...
private computeTags(env: RackEnv): string[] {
  return this.taggers.map((t) => (typeof t === "function" ? t(env) : t));
}
```

This is a structural gap, not a rename: a tagger callable receives an env hash
where Rails hands it a `Request`, so any tagger written against Rails' documented
`config.log_tags` contract (`:uuid`, `:remote_ip`, or a lambda calling
`request.method` / `request.path`) sees the wrong object. It surfaced as an
RFC 0095 `naming` call-argument row (`compute_tags` ruby `ref:request` vs ts
`ref:env`) and PR #6353 left it alone as out of scope for a rename story.

## Converged shape

Build the request once at the top of `call` and pass it to `computeTags` (and to
`callApp`, which Rails also gives the request plus the env). `compute_tags`
itself then resolves each tagger against the request
(`rack/logger.rb:57-66` — `request.send(tag)` for a Symbol, `tag.call(request)`
for a callable, the tag itself otherwise), which is a second divergence in the
same method worth converging in the same pass.

## Acceptance criteria

1. `call` constructs the request and passes it to `computeTags` / `callApp`,
   matching rack/logger.rb:20-30.
2. `computeTags` resolves Symbol / callable / literal taggers against the
   request per rack/logger.rb:57-66.
3. The `compute_tags` `naming` row is gone from
   `pnpm parity:api:calls:args:report`, not baselined.
4. A test covers a lambda tagger reading a request accessor, which fails on the
   env-passing baseline.
