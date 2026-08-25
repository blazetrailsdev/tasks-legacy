---
title: "Session#inspect's loaded arm renders no instance variables"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
  - "activesupport"
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

`Session#inspect` (`vendor/rails/actionpack/lib/action_dispatch/request/session.rb:222-228`)
was ported in PR #6700:

```ruby
def inspect
  if loaded?
    super
  else
    "#<#{self.class}:0x#{(object_id << 1).to_s(16)} not yet loaded>"
  end
end
```

The not-yet-loaded arm is faithful. The **loaded** arm — Ruby's `super`, i.e.
`Object#inspect` — is not: `packages/actionpack/src/action-dispatch/request/session.ts`
renders the bare identity form `#<ActionDispatch::Request::Session:0x…>`, while
MRI's `Object#inspect` renders the identity form **plus every instance
variable**, e.g.
`#<ActionDispatch::Request::Session:0x… @by=#<...>, @req=#<...>, @delegate={"a"=>1}, @loaded=true, ...>`.

So a loaded session logged or printed in a debugger shows none of its contents,
which is the whole reason Rails leaves the loaded arm to `super`.

The same gap exists in `KeyGenerator#inspect`
(`packages/activesupport/src/key-generator.ts:69`), which is where the
identity-form spelling came from, so this is likely one convergence with two
call sites — check [[ruby-inspect-object-fallback-and-hash-key-fidelity]] and
[[retire-remaining-ruby-inspect-copies-onto-the-activesupport-port]] for
overlap before starting; the right home is probably a real `Object#inspect`
arm in `activesupport/src/core-ext/object/inspect.ts`, whose own JSDoc already
records that its default arm is `to_s` rather than `#<Foo:0x… @a=1>` "because
reproducing that needs an object id JS does not expose" — an object id that
`objectIdHex`/`key-generator.ts` now both have.

## Converged shape

An `Object#inspect` arm in `activesupport/src/core-ext/object/inspect.ts` that
renders `#<ClassName:0x<id> @ivar=<inspect>, …>` over a class instance's own
enumerable fields, with the object id it already needs supplied by the WeakMap
idiom currently duplicated in `session.ts` and `key-generator.ts`. Then
`Session#inspect`'s loaded arm and `KeyGenerator#inspect` both call it, and the
two private WeakMap copies collapse into one.

Verify the ivar rendering against MRI directly (`ruby` is on PATH) rather than
deriving it — including whether `@delegate`'s Hash renders through the existing
`inspect()` Hash arm unchanged.

## Acceptance criteria

- [ ] `Session#inspect`'s loaded arm renders instance variables, not just the identity form.
- [ ] The object-id WeakMap exists once, not once per caller.
- [ ] Rendering pinned against MRI output in a test.
- [ ] `pnpm parity:api:calls` / `:args` green; no new baseline row.
