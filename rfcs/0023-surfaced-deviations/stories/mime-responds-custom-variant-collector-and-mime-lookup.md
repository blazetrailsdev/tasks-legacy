---
title: "Port VariantCollector and mime-type normalization in MimeResponds Collector#custom"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #4997 ported `ActionController::MimeResponds::Collector#custom` first-wins
semantics but left two parts of the Rails body unported
(`vendor/rails/actionpack/lib/action_controller/metal/mime_responds.rb:270-277`):

```ruby
def custom(mime_type, &block)
  mime_type = Mime::Type.lookup(mime_type.to_s) unless mime_type.is_a?(Mime::Type)
  @responses[mime_type] ||= if block_given?
    block
  else
    VariantCollector.new(@variant)
  end
end
```

1. **No `VariantCollector`.** With no block, Rails stores a `VariantCollector`
   (`mime_responds.rb:301`), which is what makes the inline variant syntax
   `format.html.phone { ... }` work, and `response` (`:284`) unwraps it. trails
   stores a noop handler via `on()` and has an ad-hoc `variant()` method on the
   ActionDispatch collector instead.
2. **No mime-type normalization.** Rails runs `Mime::Type.lookup(mime_type.to_s)`
   so `custom("text/html")` and `custom(:html)` converge on one key; trails uses
   the raw string as the Map key.

## Acceptance criteria

- Port `VariantCollector` and have `custom` store one when no block is given.
- `response` resolves a stored `VariantCollector` per Rails `:284-296`.
- `custom` normalizes its mime-type argument before keying the responses map.
- Tests match Rails test names where counterparts exist.
