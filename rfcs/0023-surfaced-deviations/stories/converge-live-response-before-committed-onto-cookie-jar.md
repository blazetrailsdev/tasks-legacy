---
title: "Converge Live::Response#beforeCommitted onto Rails' super + cookie-jar write"
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

`ActionController::Live::Response#beforeCommitted`
(packages/actionpack/src/action-controller/metal/live.ts:248-265) is a trails
invention: it flattens `this.cookies` into a single `set-cookie` header and
does NOT call `super`.

Rails' counterpart
(vendor/rails/actionpack/lib/action_controller/metal/live.rb:262-267) is:

```ruby
def before_committed
  super
  jar = request.cookie_jar
  # The response can be committed multiple times
  jar.write self unless committed?
end
```

After PR #6671 converged `Response#setCookie` / `#cookies` onto the
`Set-Cookie` header, the flatten is dead: the header already exists by the
time `beforeCommitted` runs, so both of its guards short-circuit. It was left
in place because deleting the override restores the `super` call, which also
runs `assignDefaultContentTypeAndCharsetBang` /
`mergeAndNormalizeCacheControlBang` / `handleConditionalGetBang` /
`handleNoContentBang` on live responses — a behavior change out of scope for
that story.

## Converged shape

```ts
protected beforeCommitted(): void {
  super.beforeCommitted();
  const jar = this.request.cookieJar;
  if (!this.committed) jar.write(this);
}
```

i.e. delete the flatten, call `super`, and write the request's cookie jar onto
the response. Needs `request.cookieJar` wired on the Live response's request
(`ActionDispatch::Cookies::CookieJar#write`, actionpack
lib/action_dispatch/middleware/cookies.rb) and the two trails-only tests in
live.test.ts:198-213 re-pointed at the Rails behavior.

## Acceptance criteria

- [ ] `beforeCommitted` mirrors live.rb:262-267 (super + jar write + committed guard).
- [ ] The trails-invention cookie flatten is gone.
- [ ] Live response tests reflect the Rails behavior.
