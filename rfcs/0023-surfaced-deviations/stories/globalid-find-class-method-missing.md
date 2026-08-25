---
title: "GlobalID.find class method is missing"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails defines `GlobalID.find` as a class method
(`vendor/globalid/lib/global_id/global_id.rb:21-23`):

```ruby
def find(gid, options = {})
  parse(gid)&.find(options)
end
```

trails has only the **instance** `GlobalID#find`
(`packages/globalid/src/global-id.ts:199-202`); there is no static counterpart,
so `GlobalID.find(param)` — the documented one-step "parse a param and locate
the record" entry point — does not exist.

Surfaced while porting `global_id_test.rb` in PR #6651: two Rails tests call it
directly and both had to route around it at the call site.

- `global_id_test.rb:33-36` (`GlobalIDParamEncodedTest#finding`) is
  `found = GlobalID.find(@gid.to_param)`; the port does
  `GlobalID.parse(gid.toParam())!.find()`.
- `global_id_test.rb:210-212` (`model class`) asserts
  `assert_raise(ArgumentError) { GlobalID.find 'gid://bcx/SignedGlobalID/5' }`;
  the port reaches the raise through
  `GlobalID.parse("gid://bcx/SignedGlobalID/5")!.modelClass`.

Both call sites carry a comment saying trails has no `GlobalID.find`. Adding it
removes the workaround and the comments.

## Converged shape

A static on `GlobalID`, mirroring the Ruby line for line — `parse` then the
safe-navigated instance `find`. Note the instance `find` is async in trails
(it reaches `Locator` through a dynamic import to avoid the
global-id ↔ signed-global-id ↔ locator TDZ cycle documented at
`global-id.ts:3-7`), so the static is async too:

```ts
static async find(gid: string | GlobalID, options?: LocateOptions): Promise<unknown | null> {
  return (await this.parse(gid))?.find(options) ?? null;
}
```

`parse` is sync, so no await is needed on it; the `?? null` supplies Ruby's
`nil` for the unparseable case.

## Acceptance criteria

- `GlobalID.find` exists as a static mirroring `global_id.rb:21-23`.
- The two `global_id_test.rb` call sites use it and drop their workaround
  comments.
- `pnpm parity:api --package globalid` does not regress; the new name has a Ruby
  counterpart so it must not appear in `parity:api:extra`.
