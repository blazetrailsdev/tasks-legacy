---
title: "after_initialize callbacks do not run on Model.new"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Found while writing `associations/build-record-block-ordering.trails.test.ts` in
PR #6380.

Rails runs the initialize callbacks at the end of `Core#initialize`
(`activerecord/lib/active_record/core.rb:479`):

```ruby
def initialize(attributes = nil)
  @new_record = true
  @attributes = self.class._default_attributes.deep_dup

  init_internals
  initialize_internals_callback

  assign_attributes(attributes) if attributes

  yield self if block_given?
  _run_initialize_callbacks
end
```

So `after_initialize` fires on EVERY `Model.new`, after attribute assignment and
after the block.

In trails it appears not to fire at all for a plain `new`. Reproduced with a
model that declares its attributes and takes no association:

```ts
class DbgPost extends Base {
  seen: unknown = "UNSET";
  static {
    this._tableName = "posts";
    this.attribute("title", "string");
  }
}
afterInitialize(DbgPost, function (this: any) {
  this.seen = this.readAttribute("title");
});

new DbgPost({ title: "hello" });
// seen === "UNSET"  (callback never ran); readAttribute("title") === "hello"
```

Registered both ways — `afterInitialize(Model, fn)` from `callbacks.ts` and a
`this.afterInitialize(...)` call in the class's static block — with the same
result. Note the sibling story `dup-sets-attributes-before-after-initialize`
(done) covers `dup`, not `new`.

This is why PR #6380's block-ordering test asserts only the
`initialize_attributes`-visibility half of `build_record`
(`association.rb:383-388`) and not the "block runs before `after_initialize`"
half — the second half is unobservable while the callback does not run.

## Converged shape

`Model.new` runs `_run_initialize_callbacks` as its last step, after attribute
assignment and after yielding the block (`core.rb:479`), so an
`after_initialize` callback observes the fully-assigned record. Confirm first
whether the gap is in the registration surface or in `Core#initialize`; the
repro above is a plain `new`, so start there.

## Acceptance criteria

1. A registered `after_initialize` callback runs on `Model.new`, after
   `assign_attributes` and after the constructor block, per `core.rb:479`.
2. A test pins it, and pins the ordering against the constructor block (the
   block runs first).
3. `associations/build-record-block-ordering.trails.test.ts` grows the
   `after_initialize` assertion its header comment describes, closing the half
   PR #6380 could not assert.
4. The Rails-named `after_initialize` tests in the AR suite stay green.
