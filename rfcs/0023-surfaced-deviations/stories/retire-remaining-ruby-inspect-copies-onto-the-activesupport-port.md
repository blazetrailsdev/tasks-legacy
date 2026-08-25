---
title: "Retire the three remaining private Ruby inspect/to_s copies onto core-ext/object/inspect"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
  - "activerecord"
  - "activesupport"
  - "arel"
deps: []
deps-rfc: []
est-loc: 280
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6571 (story `port-object-inspect-and-retire-private-to-s-copies`) ported
Ruby's `Object#inspect` / `Object#to_s` to
`packages/activesupport/src/core-ext/object/inspect.ts` and retired the two
private copies its acceptance criteria named (`xml-mini.ts`, `array-utils.ts`).

While filing that work, three MORE partial copies of the same Ruby semantics
turned up, none of them in that story's scope:

- `packages/activerecord/src/relation/ruby-inspect.ts` — `rubyInspect` /
  `rubyInspectHash`, used by `Relation#_renderExplainBinds` and
  `Migration#formatArguments`.
- `packages/arel/src/visitors/to-sql.ts` — `rubyToS` / `rubyInspect` /
  `rubyStringInspect` (added by #4974; the most complete escaping of the set,
  verified byte-identical against MRI across 21 adversarial inputs).
- `packages/actionpack/src/action-dispatch/journey/formatter.ts:328-343` —
  `rubyInspectHash` / `rubyInspectArray` / `rubyInspect`, which escapes nothing
  inside strings.

They disagree: the new activesupport port renders a Ruby Symbol key as `:b` and
a String key as `"b"` (the CLAUDE.md colon convention, MRI-verified), while
`ruby-inspect.ts` always emits the symbol form, and the arel copy has escaping
the other three lack.

## The prior decision this has to clear first

`consolidate-ruby-inspect-to-s-helpers` (0023, **closed**) proposed exactly this
consolidation and was closed as "Not Rails-convergent: extracting a shared Ruby
inspect/to_s helper into activesupport is an abstraction Rails does not have
(Ruby gets it from Object#inspect)."

What changed: that helper now EXISTS, at its Rails-facing location, put there by
a maintainer-authored story (RFC 0101). So this story is no longer "create an
abstraction Rails lacks" — it is "delete duplicates of a port that already
shipped", the same second half as
`port-object-blank-to-core-ext-and-retire-private-copies`. **Triage should
confirm that reading before this is picked up**; if the closure still stands,
block this rather than converging it.

## Converged shape

Retire the copies onto `activesupport/src/core-ext/object/inspect.ts`, folding
the arel copy's string escaping into the port first (it is the only one that
matches MRI on control characters, `\uXXXX`, and `\#`), so consolidation raises
fidelity instead of lowering it. Take one call site at a time — the four
sites have different Symbol-vs-String key expectations and cannot be swapped
blind.

Two known gaps in the shared port to fix while there, both currently documented
in its JSDoc rather than implemented, and both already described for the
activerecord copy by `ruby-inspect-object-fallback-and-hash-key-fidelity`:

- The default arm is `to_s`, not Ruby's `#<Foo:0x… @a=1>`.
- Hash-key rendering depends on the colon convention being applied at the call
  site.

## Acceptance criteria

- [ ] One Ruby `inspect`/`to_s` implementation; `ruby-inspect.ts`,
      `to-sql.ts` and `journey/formatter.ts` hold no private copy.
- [ ] The surviving port carries the arel copy's escaping, still byte-identical
      to MRI on #4974's 21 adversarial inputs.
- [ ] Each retired call site keeps its current rendering, asserted against
      `ruby -e` output.

## Absorbed: `ordered-options-inspect-uses-json-stringify-not-ruby-inspect`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "OrderedOptions#to_s renders values with JSON.stringify, not Ruby #inspect"

### Context

Surfaced porting `ordered_options_test.rb` in #6692.

`OrderedOptions#toString` and `InheritableOptions#toString`
(`packages/activesupport/src/ordered-options.ts`) render each value with
`JSON.stringify`, not Ruby's `#inspect`:

```ts
const pairs = [...this.data.entries()].map(([k, v]) => `${k}: ${JSON.stringify(v)}`);
return `{${pairs.join(", ")}}`;
```

Rails reaches Ruby's own `Hash#inspect` — `OrderedOptions` does not override
`to_s` at all (it inherits it), and `InheritableOptions#to_s` is `to_h.to_s`
(activesupport/lib/active_support/ordered_options.rb:111-117). `inspect` on both
wraps that rendering (ordered_options.rb:68-70, 108-110).

`JSON.stringify` and `#inspect` agree only for Strings. They diverge for:

- **Symbols** — a Ruby Symbol is a JS string carrying its colon (CLAUDE.md), so
  `:bar` is `":bar"` and renders as `":bar"` (quoted) instead of Ruby's `:bar`.
- **nil** — `null` instead of `nil`.
- **nested Hashes/Arrays** — `{"a":1}` instead of `{:a=>1}`.

This is what forced #6692's test to assert the quoted form:

```ts
const hash = `{foo: ":bar", baz: ":quz"}`;
expect(a.inspect()).toBe(`#<ActiveSupport::OrderedOptions ${hash}>`);
```

against Rails' `assert_equal "#<ActiveSupport::OrderedOptions #{{ foo: :bar, baz: :quz }}>", a.inspect`
(activesupport/test/ordered_options_test.rb:138-146). The assertion-value gate
skips that pair only because the Rails side is interpolated; the rendering is
still wrong.

### Converged shape

Render values through the `Object#inspect` trails already has —
`inspect` in `packages/activesupport/src/core-ext/object/inspect.ts:43`, which
handles the Symbol-as-colon-string case (`SYMBOL_RE`, :29-31), `nil`, nested
Arrays and Hashes, and is verified against MRI 3.3. Same package, no new import
edge.

One open question to settle first, because it changes the expected strings:
`inspect.ts` renders a Hash 3.3-style (`{:foo=>:bar}`), while the vendored Rails
tests interpolate a literal that renders 3.4-style (`{foo: :bar}`) — and
`ordered-options.ts` currently emits a third spelling (`{foo: ":bar"}`, bare key

- JSON value). Pick the one the vendored Ruby produces (check
  `vendor/rails/.ruby-version` / run `ruby -e 'p({foo: :bar}.inspect)'`) and make
  `inspect.ts` and `ordered-options.ts` agree.

Then tighten #6692's two `ordered-options.test.ts` sites (the `const hash` /
`const one` / `const all` locals in "ordered option inspect", "inheritable
option inspect", "ordered options to s", "inheritable options to s" and the two
`pp` tests) to the Ruby rendering.

Sibling call sites of the same class, already filed:
`check-constraint-raise-message-uses-json-stringify-not-ruby-hash-inspect`,
`batches-order-inspect-hand-rolls-symbol-rendering`.

## Absorbed: `batches-order-inspect-hand-rolls-symbol-rendering`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "batches :order message hand-rolls Symbol#inspect because ActiveSupport's Object#inspect is unreachable"

### Context

`vendor/rails/activerecord/lib/active_record/relation/batches.rb:324`
interpolates `order.inspect`, and `:order` takes Symbols, so Rails prints
`:invalid` and `[:asc, :sideways]`. trails spells a Ruby Symbol as a bare
string (CLAUDE.md, "A Ruby Symbol is a JS string"), so PR #6646 restored the
colon and hand-rolled the `Symbol#inspect` / `Array#inspect` rendering inline
at `packages/activerecord/src/relation/batches.ts` (in
`ensureValidOptionsForBatchingBang`):

```ts
const inspected = Array.isArray(order) ? `[${order.map((o) => `:${o}`).join(", ")}]` : `:${order}`;
```

ActiveSupport already carries the ported `Object#inspect` with exactly the
Symbol arm this needs — `packages/activesupport/src/core-ext/object/inspect.ts`,
whose `SYMBOL_RE` emits a colon-prefixed string bare rather than quoted — but
it is not exported from `packages/activesupport/src/index.ts`, so ActiveRecord
cannot reach it. #6646 tried exporting it and backed the export out:
`pnpm parity:api:extra --package activesupport` then reports
`core-ext/object/inspect.ts — 1 novel [no Rails counterpart]`, because
`Object#inspect` is Ruby core rather than Rails and has no Ruby counterpart in
the vendored ActiveSupport tree.

AR's own `packages/activerecord/src/relation/ruby-inspect.ts` (`rubyInspect`)
is the third copy of this, and implements only the `String#inspect` arm — it
renders a trails Symbol as `"invalid"`, which is what #6646's first revision
shipped and a reviewer caught.

### Acceptance criteria

- [ ] One reachable `Object#inspect` — decide whether ActiveSupport's port is
      exported (and how `parity:api:extra` should classify a Ruby-core method
      with no Rails counterpart, e.g. a `@noRailsEquivalent` receipt) or
      whether `rubyInspect` grows the Symbol arm and the ActiveSupport copy
      routes through it.
- [ ] The inline rendering in `ensureValidOptionsForBatchingBang` is deleted
      and calls that one inspector.
- [ ] The message output is unchanged: `got :invalid` and
      `got [:asc, :sideways]`, per batches.rb:324 (covered by the two tests in
      `packages/activerecord/src/batches.trails.test.ts`).
- [ ] `pnpm parity:api:extra` reports no new novel surface.

## Absorbed: `ruby-inspect-object-fallback-and-hash-key-fidelity`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Converge rubyInspect's non-plain-object fallback and hash-key rendering on Ruby"

### Context

`rubyInspect` (`packages/activerecord/src/relation/ruby-inspect.ts:23`) is now
used by `Migration#formatArguments` (`packages/activerecord/src/migration.ts`,
PR #5772) in addition to `Relation#_renderExplainBinds`, so two shortcuts in it
are user-visible in migration announce labels:

- Non-plain objects fall through to `String(value)` (ruby-inspect.ts:52-53).
  Ruby's `Object#inspect` gives `#<Widget ...>`; a plain JS class instance
  renders `[object Object]`.
- `rubyInspectHash` (ruby-inspect.ts:62) always emits `key: value`, matching
  Ruby's _symbol_-key Hash form. Ruby renders string keys as `"key" => value`.
  trails has no symbol/string key distinction, so every hash takes the symbol
  form — correct for options hashes, wrong for genuine string-keyed data.

Both predate #5772 and were left alone there to keep that PR scoped to
`format_arguments` itself.

### Acceptance criteria

- [ ] Decide and implement the non-plain-object fallback: either `#<Ctor>`
      (constructor-name based) or a documented deliberate deviation at the
      call site.
- [ ] Document (or converge) the hash-key rendering choice against Ruby's
      `Hash#inspect`.
- [ ] Existing `rubyInspect` consumers (`Relation#inspect`, explain binds,
      `formatArguments`) keep their current output where it is already
      Rails-faithful.

## Triage note (2026-08-18)

Merged story. The three absorbed rows are each a _site_ of the one deviation
this story names: a private partial copy of Ruby's `Object#inspect` semantics
that should read `packages/activesupport/src/core-ext/object/inspect.ts`
instead. Retiring the copy is the fix in every case, so the per-row LOC
estimates overlap heavily — `est-loc` is set to 280, not their 330 sum.

Sites, in the order they are cheapest to retire:

1. `packages/activesupport/src/ordered-options.ts` — `OrderedOptions#toString` /
   `InheritableOptions#toString` render values with `JSON.stringify`.
2. `packages/activerecord/src/relation/batches.ts` — the `:order` message
   hand-rolls `Symbol#inspect` / `Array#inspect` (batches.rb:324).
3. `packages/activerecord/src/relation/ruby-inspect.ts:23` — `rubyInspect`
   itself, whose non-plain-object fallback (`:52-53`) drops to `String(value)`
   and whose hash-key rendering diverges. Two consumers today:
   `Migration#formatArguments` and `Relation#_renderExplainBinds`.

If the whole set will not fit one PR, ship 1+2 and re-file 3 — do not re-file
1 or 2 separately, they are two-line reroutes once the shared port is reached.
