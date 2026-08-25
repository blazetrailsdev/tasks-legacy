---
title: "Fix TS4094 declaration emit on Application's anonymous executor/reloader classes"
status: ready
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["trailties"]
deps: []
deps-rfc: []
est-loc: 15
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

`packages/trailties/src/application.ts:41` and `:45` declare:

```ts
readonly executor = class extends Executor {};
readonly reloader = class extends Reloader {};
```

These mirror Rails' `@executor = Class.new(ActiveSupport::Executor)` and
`@reloader = Class.new(ActiveSupport::Reloader)` (`application.rb:122-123`) —
a per-application subclass so one app's prepare callbacks don't leak into
another's. The Rails shape is correct and must be preserved.

`typescript@5.9.3` emits declarations for these fine. `typescript@7.0.2`
refuses:

```text
packages/trailties/src/application.ts(41,12): error TS4094: Property '#private'
  of exported anonymous class type may not be private or protected.
packages/trailties/src/application.ts(45,12): error TS4094: …
```

`Executor` and `Reloader` carry `#private` fields, and TS 7 will not emit a
`.d.ts` for an exported anonymous class expression that inherits them. This is
the **only** diagnostic difference between TS 5.9.3 and TS 7.0.2 across all 18
projects and 3,472 `.ts` files (measured 2026-08-25, RFC
`0000-typescript-7-ground-floor` § "The spike"), and it is the one thing that
makes `trailties/dist/application.d.ts` fail to emit under TS 7.

Worth fixing on its own merits regardless of whether trails ever adopts TS 7:
an anonymous exported class type is a `.d.ts` shape that no consumer can name.

## Acceptance criteria

- [ ] `packages/trailties/src/application.ts` no longer produces TS4094 under
      `typescript@7.0.2`.
- [ ] The Rails semantics are preserved: `executor` and `reloader` remain
      **per-Application subclasses**, not shared singletons — two `Application`
      instances must not share prepare callbacks.
- [ ] The Rails-citation JSDoc on both properties is kept.
- [ ] `pnpm build` and `pnpm typecheck` stay green under the repo's pinned
      `typescript@5.9.3`, with no `.d.ts` shape change visible to consumers of
      `@blazetrails/trailties`.

## Definition of done

Deleting the anonymous-subclass shape (e.g. assigning `Executor` itself) does
not close this story — that drops the per-application isolation Rails has.
A `// @ts-expect-error` or an `eslint-disable` does not close it either.

## Verification

```bash
# with typescript@7.0.2 installed OUTSIDE the tree (do not commit it):
node /path/to/ts7/bin/tsc -b packages/trailties   # expect: zero diagnostics
pnpm build && pnpm typecheck                      # expect: green on 5.9.3
```

## Notes

Likely fixes, in order of preference: give each class expression a name
(`class ApplicationExecutor extends Executor {}`), or add an explicit type
annotation to the property so the emitter has a nameable type to write.
