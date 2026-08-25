---
title: "Retarget the pre-existing minitest citations in assertions.ts to the pinned 5.27.0"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Retarget the pre-existing minitest citations in assertions.ts to the pinned 5.27.0

## Context

`scripts/parity/pipeline/schema/ruby/Gemfile.lock:32` pins minitest **5.27.0**,
but the minitest citations that predate #6525 in
`packages/activesupport/src/testing/assertions.ts` carry **5.25.4** line
numbers — roughly a 15-line offset through that part of the file. #6525
retargeted every citation it added and deliberately left these alone rather
than widen its diff:

- `UnexpectedError` — cited `minitest.rb:1078-1110`; 5.27.0 is **1074-1108**.
- Its `message`/`backtrace` note — cited `:1097-1107`; 5.27.0 is
  **1092-1102**.
- `UnexpectedError::BASE_RE` — cited `:1101`; 5.27.0 is **1096**.
- `classNameOf`'s note — cited `:1105`; 5.27.0 is **1100**.
- `BacktraceFilter` — cited `:1173-1199`; `MT_RE` `:1176`; `#filter`
  `:1191-1201`; the `$DEBUG` guard `:1194`.
- The `Minitest` module seat — cited `minitest.rb:43` / `:1204` / `:365-369`.

The code is textually identical between the two versions in these sections, so
this is a citation-accuracy fix only, not a behavioral one.

## Acceptance criteria

- [ ] Every `minitest.rb:` citation in
      `packages/activesupport/src/testing/assertions.ts` resolves to the right
      lines of the pinned minitest 5.27.0 (verify each against the installed
      gem, do not offset by hand).
- [ ] No behavioral change; the file's tests still pass.
