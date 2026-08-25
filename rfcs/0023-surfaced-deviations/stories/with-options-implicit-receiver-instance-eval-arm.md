---
title: "with-options-implicit-receiver-instance-eval-arm"
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/core-ext/object/with-options.ts` ports
`Object#with_options`
(`vendor/rails/activesupport/lib/active_support/core_ext/object/with_options.rb:92`),
but only one of its two block arms:

```ruby
block.arity.zero? ? option_merger.instance_eval(&block) : block.call(option_merger)
```

The `block.call(option_merger)` arm is ported. The zero-arity arm rebinds
`self` to the merger with `instance_eval`, so a block that names no parameter
still reaches the merger's `method_missing`. JavaScript has no way to rebind
the scope a function body already closed over, so a zero-parameter block
cannot reach the merger at all and the arm has no form.

The consequence is that implicit-receiver `with_options` is unsupported:

```ruby
@options.with_options foo: "bar" do
  merge! fizz: "buzz"
end
```

which is what `vendor/rails/activesupport/test/option_merger_test.rb:101-107`
(`test_option_merger_implicit_receiver`) exercises. That test is therefore
absent from `packages/activesupport/src/option-merger.test.ts`.

This was surfaced in review of PR #6201, which ported `ActiveSupport::OptionMerger`
and `Object#with_options`. The gap is documented in the `withOptions` JSDoc.

## Acceptance criteria

- [ ] Either establish a settled trails idiom that lets a zero-parameter block
      reach the merger (a `this`-bound block invoked as `block.call(merger)` is
      the only candidate, and it only works for `function` blocks, not arrows),
      and port `test_option_merger_implicit_receiver` with the Rails name; or
- [ ] `pnpm tasks block` this with the specific language shortcoming, and
      record it where the repo records language-forced gaps rather than only in
      a JSDoc comment.
