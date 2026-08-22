---
title: "validateInverseOf is a trails invention — Rails validates via check_validity_of_inverse!"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 150
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `validateInverseOf` is a trails invention — Rails validates through `check_validity_of_inverse!`

## Context

Surfaced converging `inverse-of-association-error-takes-reflection` in PR
6563, which made `InverseOfAssociationNotFoundError` take the reflection as
Rails does. That left this helper as the last caller constructing the error
from a call site rather than from the reflection alone.

`packages/activerecord/src/associations.ts`:

    export function validateInverseOf(
      owner: typeof Base,
      targetModel: typeof Base,
      assocName: string,
      inverseOf: string,
    ): void {
      const targetAssocs = targetModel._associations ?? [];
      if (targetAssocs.length === 0) return;
      if (targetAssocs.some((a) => a.name === inverseOf)) return;
      throw new InverseOfAssociationNotFoundError(
        owner._reflectOnAssociation(assocName),
        targetModel,
      );
    }

Called from `associations/has-many-association.ts` (`findTarget`) and
`associations/singular-association.ts`, both at load time.

Rails has no `validate_inverse_of`. The check lives on the reflection:
`ActiveRecord::Reflection::AssociationReflection#check_validity_of_inverse!`,
`activerecord/lib/active_record/reflection.rb:264-273`:

    def check_validity_of_inverse!
      if !polymorphic? && has_inverse?
        if inverse_of.nil?
          raise InverseOfAssociationNotFoundError.new(self)
        end
        if inverse_of == self
          raise InverseOfAssociationRecursiveError.new(self)
        end
      end
    end

reached from `check_validity!` (reflection.rb) at association-load time.
trails already has `checkValidityOfInverseBang` and it is already Rails'
one-liner after #6563 — so the two paths now do the same job by different
routes, and only the trails one runs from the association loaders.

Note the `targetAssocs.length === 0` early return has no Rails counterpart
either: it silently skips validation for a model whose associations have not
been declared yet, which Rails would treat as a missing inverse.

## Converged shape

Delete `validateInverseOf` and its four call-site arguments; route the
association loaders through the reflection's `checkValidityOfInverseBang`
(itself reached from `checkValidityBang`, as Rails does), so there is one
validation path with Rails' name and Rails' trigger point.

## Acceptance criteria

- [ ] `validateInverseOf` is gone from `associations.ts`; no caller
      constructs `InverseOfAssociationNotFoundError` directly outside
      `reflection.rb`'s two ported raise sites.
- [ ] The rendered message and `Did you mean?` corrections are unchanged
      (`associations/inverse-associations.test.ts` covers both).
- [ ] The no-declared-associations early return is either traced to a Rails
      line or dropped.
- [ ] `pnpm parity:api:extra --package activerecord` does not grow; all three
      adapter lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
