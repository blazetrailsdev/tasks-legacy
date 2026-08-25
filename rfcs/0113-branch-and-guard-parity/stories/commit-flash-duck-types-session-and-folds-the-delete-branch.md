---
title: "commit_flash duck-types session.enabled? and folds Rails' second branch"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: guard-parity
packages: []
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

`ActionDispatch::Flash::RequestMethods#commit_flash`
(`vendor/rails/actionpack/lib/action_dispatch/middleware/flash.rb:71-82`) is:

```ruby
def commit_flash
  return unless session.enabled?

  if flash_hash && (flash_hash.present? || session.key?("flash"))
    session["flash"] = flash_hash.to_session_value
    self.flash = flash_hash.dup
  end

  if session.loaded? && session.key?("flash") && session["flash"].nil?
    session.delete("flash")
  end
end
```

trails' port (`packages/actionpack/src/action-dispatch/middleware/flash.ts:75-95`)
diverges on three points:

- it guards with a duck-type, `if (session.isEnabled && !session.isEnabled()) return;`,
  rather than Rails' `return unless session.enabled?`. The duck-type dated from
  `Request#session` answering a plain object; PR #6696 converged that reader onto
  Rack's `fetch_header … default_session` (`vendor/rack/lib/rack/request.rb:207-211`,
  `.../http/request.rb:505-507`), so `session` is now always a `Session` and the
  guard can be Rails' straight `enabled?` check.
- it folds Rails' second `if` into the first as an `Object.keys(value).length === 0`
  → `session.delete("flash")` branch, instead of Rails' separate
  `session.loaded? && session.key?("flash") && session["flash"].nil?` block.
- Rails writes `session["flash"] = flash_hash.to_session_value` unconditionally
  inside the first branch; trails only writes when the value is non-empty.

## Acceptance criteria

- [ ] `commitFlash` mirrors `flash.rb:71-82` branch for branch: the
      `session.isEnabled()` early return, the single write of
      `flashesForSession()` plus `this.flash = hash.dup()`, and the separate
      `isLoaded() && hasKey("flash") && get("flash") == null` delete block.
- [ ] No duck-type guard remains; `session` is typed as `Session`.
- [ ] Existing flash tests stay green, and any trails-only test that encodes the
      folded-delete behaviour is retired or re-pointed at the Rails shape.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
