---
title: "Preloader::Association#loaded?/#target_for swallow every error in a bare catch"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Preloader::Association#loaded? and #target_for swallow every error in a bare catch

## Context

Surfaced while converging `target_for -> wrap` in PR #6734 (RFC 0106 wave 4d).

Rails' two readers are bare one-liners with **no** `rescue`
(`activerecord/lib/active_record/associations/preloader/association.rb:176-182`):

```ruby
def loaded?(owner)
  owner.association(reflection.name).loaded?
end

def target_for(owner)
  Array.wrap(owner.association(reflection.name).target)
end
```

trails wraps both in a `try`/`catch` that swallows **every** error and
substitutes a neutral value
(`packages/activerecord/src/associations/preloader/association.ts:124-137`):

```ts
isLoaded(owner: Base): boolean {
  try {
    return (owner as any).association(this.reflection.name).loaded;
  } catch {
    return false;
  }
}

targetFor(owner: Base): Base[] {
  try {
    return wrap((owner as any).association(this.reflection.name).target);
  } catch {
    return [];
  }
}
```

`owner.association(name)` raises `AssociationNotFoundError` in Rails when the
name does not resolve, and the preloader is meant to propagate that. Here a
mis-resolved reflection name, a broken owner, or any unrelated bug inside
`association()` is silently reinterpreted as "not loaded" / "no target", so the
preloader quietly preloads nothing instead of raising. That is a swallowed-error
class of bug: the failure surfaces later as missing associated records with no
stack pointing at the cause.

The `catch` is not a TypeScript language shortcoming — nothing about the ported
body needs it. It reads as defensive scaffolding retained from an earlier port.

## Converged shape

Drop both `try`/`catch` blocks so the two methods are the bare one-liners Rails
has, and let `association()` raise:

```ts
isLoaded(owner: Base): boolean {
  return (owner as any).association(this.reflection.name).loaded;
}

targetFor(owner: Base): Base[] {
  return wrap((owner as any).association(this.reflection.name).target);
}
```

Expect to find call sites that were relying on the swallow — the point of the
story is to fix those properly (guard the name before calling, or let the error
propagate as Rails does), not to reinstate the `catch`.

## Acceptance criteria

- [ ] `isLoaded` and `targetFor` carry no `try`/`catch`; both bodies match
      `preloader/association.rb:176-182` one-for-one.
- [ ] Any caller that depended on the swallowed error is fixed at the caller,
      with the Rails behavior (raise) preserved.
- [ ] A regression test covers the raising path — an unresolvable association
      name reaching the preloader must raise rather than preload nothing.
      Verify it FAILS before the change.
- [ ] `pnpm parity:api:calls` / `:args` green; SQLite, PostgreSQL and
      MySQL/MariaDB lanes green.
