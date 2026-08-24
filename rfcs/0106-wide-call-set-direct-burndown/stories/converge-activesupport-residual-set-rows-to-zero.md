---
title: "Retire activesupport's last 3 call-set rows so the package hits the RFC's zero"
status: claimed
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activesupport"]
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: "2026-08-24T13:47:15Z"
assignee: "converge-activesupport-residual-set-rows-to-zero"
blocked-by: null
closed-reason: null
---

## Context

RFC 0106's exit criterion is **0 `kind: "set"` rows for `activerecord`, `arel`
and `activesupport`**. Measured against `origin/main`'s exclude tree
(2026-08-24): `arel` is at 0, `activerecord` is at 39, and `activesupport` is at
**3** — the whole package fits in one story, and landing it retires a package
from the RFC's scope outright.

The three rows, from
`scripts/api-compare/call-mismatches-exclude/activesupport/`:

| tsFile                              | rubyName                         | call     |
| ----------------------------------- | -------------------------------- | -------- |
| `messages/metadata.ts`              | `extract_from_metadata_envelope` | `utc`    |
| `messages/metadata.ts`              | `pick_expiry`                    | `utc`    |
| `number-helper/number-converter.ts` | `format_options`                 | `merge!` |

Both reasons are honest and both are stale in the same way — they were written
when the missing name genuinely had no port:

- **`utc` ×2.** `messages/metadata.rb` spells `Time.now.utc >= parse_expiry(...)`
  (`extract_from_metadata_envelope`) and `expires_at.utc` /
  `Time.now.utc.advance(...)` (`pick_expiry`). The port reads the clock as a JS
  `Date`, which is already an absolute instant, so the `.utc` hop is a no-op on
  this seat — `metadata.ts:177` is `new Date(expiresAt).getTime()`. The row's own
  reason records that it "surfaced only once `DateTime#utc`
  (`date_time/calculations.rb:184`) was ported and `utc` entered the population",
  which is exactly the fact that makes it convergeable: there IS a ported `utc`
  now, and the question is whether this seat should route through the trails
  `Time`/`DateTime` analogue instead of a bare `Date`.
- **`merge!`.** `number_converter.rb:146` is
  `default_format_options.merge!(i18n_format_options)`; `number-converter.ts:153`
  spells it as an object spread (`{ ...this.formatOptions(), ...this.opts }`),
  which is not a call node. `number-converter.ts:148` already carries a JSDoc
  paragraph narrating this. `Hash#merge!` is Ruby core, not Rails, so there is no
  ported name for the gate to match — this is the RFC 0082 idiom class, and the
  disposition here is either a ported `Hash#merge!` idiom or a reviewed
  `@missingRailsCall` receipt at the call site.

Per the RFC's Verification section, a row that genuinely cannot converge leaves
as a reviewed one-line reason or a `@missingRailsCall` tag at the call site —
**not** as a widened allowlist and not as a new row. So both dispositions are in
scope for this story; what is out of scope is leaving the rows sitting in the
exclude tree.

## Acceptance criteria

- [ ] Each of the three rows is decided **per site against the Ruby body**, not
      by call name: either the TS body calls the ported name (`utc` on the trails
      `Time`/`DateTime` analogue; a ported `Hash#merge!` idiom), or the call site
      carries a `@missingRailsCall` receipt whose reason states why no TypeScript
      spelling can reach it.
- [ ] `scripts/api-compare/call-mismatches-exclude/activesupport/**` reports
      **0** rows with `kind: "set"`. Rows are deleted by hand via
      `serializeBaseline`; no `--write`, no reseed.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green, with
      `pnpm parity:api:calls:tighten <shard>` run on the affected shards to fix
      stale high-water marks.
- [ ] If either `utc` row converges by routing through the trails `Time`
      analogue, the existing `Messages::Metadata` tests still pass on all three
      database lanes (the expiry comparison is the only behavioural surface).
- [ ] Any `@missingRailsCall` receipt written here is classified honestly —
      CONVERGEABLE unless no TypeScript spelling can exist. Per CLAUDE.md, "a
      documented deviation is debt, not permission"; do not mark the `merge!`
      row PERMANENT just because `Hash#merge!` is core Ruby.
