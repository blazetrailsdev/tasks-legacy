---
title: "Scheme#key_provider adds an encryptor: early return Rails has no equivalent for"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `Scheme#initialize` onto `context_properties`
(PR #6368, `converge-scheme-encryptor-context-properties`). Left alone there
because it is control flow in a different method, not the constructor.

Rails `Scheme#key_provider`
(`activerecord/lib/active_record/encryption/scheme.rb:56-58`):

```ruby
def key_provider
  @key_provider_param || key_provider_from_key || deterministic_key_provider || default_key_provider
end
```

Four alternatives, no guard. trails
(`packages/activerecord/src/encryption/scheme.ts`, `get keyProvider`) adds a
leading early return Rails does not have:

```ts
get keyProvider(): unknown {
  // When an explicit encryptor is provided, key providers are irrelevant —
  // the encryptor handles encryption without needing key material from here.
  if (this._opts.encryptor !== undefined) return this._keyProviderParam ?? undefined;
  return (
    this._keyProviderParam ?? this.keyProviderFromKey() ?? ...
  );
}
```

With a custom `encryptor:`, Rails still resolves the full chain and would hand
back `default_key_provider`; trails returns `undefined`. Any caller that reads
`scheme.keyProvider` to build encryption options therefore sees a different
value than Rails would.

Two smaller divergences ride along in the same file:

- `isSupportUnencryptedData` / `isFixed` read `this._opts.*` rather than the
  ivars Rails memoizes (`@support_unencrypted_data`, `@fixed ||= ...`);
  `fixed?` in particular is `@deterministic && (!@deterministic.is_a?(Hash) ||
@deterministic[:fixed])` (scheme.rb:52-55), which trails reduces to
  `this._opts.fixed ?? this.deterministic`.

## Converged shape

Drop the early return so `key_provider` is the four-way `||` chain
scheme.rb:57 is, and satisfy whatever the guard was protecting at the reading
call site instead. Establish first WHY it was added — find the caller that
breaks without it and cite it — since removing it changes what a scheme with a
custom `encryptor:` reports.

## Acceptance criteria

- [ ] `keyProvider` is the unguarded four-alternative chain of scheme.rb:57.
- [ ] The condition the guard protected is satisfied at its own call site, with
      the Rails file:line that justifies it — or the guard is shown to be
      unnecessary and simply deleted.
- [ ] `fixed?` ports scheme.rb:52-55 including the Hash arm.
- [ ] Encryption suites green.

## Absorbed: `converge-scheme-validate-config-raises`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Scheme#validate_config! has a fifth raise and five reworded messages"

### Context

Surfaced while converging `Scheme#initialize` (PR #6368,
`converge-scheme-encryptor-context-properties`). That PR moved
`validate_config!` to Rails' position in the constructor and repointed two of
its checks at the new ivars, but did NOT touch the raises themselves.

CLAUDE.md, "Fidelity is the job": _"Errors. Same error class, same message
string, same raise site."_ All five raises diverge.

Rails `validate_config!`
(`activerecord/lib/active_record/encryption/scheme.rb:71-76`) — FOUR raises:

```ruby
raise Errors::Configuration, "ignore_case: can only be used with deterministic encryption" if @ignore_case && !@deterministic
raise Errors::Configuration, "key_provider: and key: can't be used simultaneously" if @key_provider_param && @key
raise Errors::Configuration, "compressor: can't be used with compress: false" if !@compress && @compressor
raise Errors::Configuration, "compressor: can't be used with encryptor" if @compressor && @context_properties[:encryptor]
```

trails (`packages/activerecord/src/encryption/scheme.ts`,
`validateConfigBang`) — FIVE, every message reworded:

| Rails                                                         | trails                                             |
| ------------------------------------------------------------- | -------------------------------------------------- |
| `ignore_case: can only be used with deterministic encryption` | `ignoreCase requires deterministic encryption`     |
| — (no such raise)                                             | `downcase requires deterministic encryption`       |
| `key_provider: and key: can't be used simultaneously`         | `key and keyProvider can't be used simultaneously` |
| `compressor: can't be used with compress: false`              | `compressor can't be used with compress: false`    |
| `compressor: can't be used with encryptor`                    | `compressor can't be used with encryptor`          |

The extra `downcase && !deterministic` raise is the substantive one: Rails
accepts `downcase: true` without `deterministic:` (the constructor does
`@downcase = downcase || ignore_case`, scheme.rb:22, and only `ignore_case`
gates), so trails rejects a configuration Rails allows. The message strings are
mechanical but are what a Rails dev greps for, and the Rails ones keep the
trailing colon that names the offending kwarg.

### Converged shape

Port the four raises verbatim — same order, same guards, same message strings
including the kwarg colons — and delete the `downcase` raise unless a Rails
`file:line` can be produced that rejects that configuration.

### Acceptance criteria

- [ ] `validateConfigBang` raises exactly the four raises of scheme.rb:71-76,
      in order, with the Rails message strings byte-for-byte.
- [ ] `new Scheme({ downcase: true })` no longer throws, or the extra raise is
      justified with a Rails citation.
- [ ] `SchemeTest`'s "validates config options when using encrypted attributes"
      still passes; any assertion that depended on the reworded strings is
      updated to the Rails strings (test NAMES unchanged).
- [ ] Encryption suites green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
