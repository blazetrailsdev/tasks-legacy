---
title: "MessagePack::Serializer#load collapses unpacker's block form into an argument"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6558 (RFC 0096 wave-4 naming burndown for activesupport). A
`class: "naming"` call-argument row on
`packages/activesupport/src/message-pack/serializer.ts` survives because trails
turned Rails' block form into an argument.

Rails, `activesupport/lib/active_support/message_pack/serializer.rb:19-25`:

    def load(dumped)
      message_pack_pool.unpacker do |unpacker|
        unpacker.feed_reference(dumped)
        raise "Invalid serialization format" unless unpacker.read == SIGNATURE_INT
        unpacker.full_unpack
      end
    end

`unpacker` takes no arguments and yields a pooled unpacker; the dumped bytes are
fed to it inside the block via `feed_reference`.

trails, `message-pack/serializer.ts:42-47`:

    load(dumped: Buffer): unknown {
      const unpacker = this.messagePackPool().unpacker(dumped);
      if (unpacker.read() !== SIGNATURE_INT)
        throw new MessagePackError("Invalid serialization format");
      return unpacker.read();
    }

Two divergences: `unpacker(dumped)` collapses Rails' no-arg call plus
`feed_reference` into one call, and the final `unpacker.read()` stands in for
Rails' `full_unpack`. `messagePackPool()` itself is documented as the factory
playing the role of Ruby's pool (there are no threads to pool against), which is
defensible; the block-to-argument collapse is not.

## Converged shape

`unpacker()` taking no argument and yielding (a callback, per the trails block
idiom), with `feedReference(dumped)` and `fullUnpack()` as separate calls inside
it, mirroring serializer.rb:20-24. The `unpacker` naming row then clears in
`pnpm parity:api:calls:args:report`.

## Acceptance criteria

- [ ] `load` mirrors serializer.rb:19-25 call for call, including
      `feed_reference` and `full_unpack` as distinct calls.
- [ ] The `unpacker` naming row clears; no new `shape` row; no baseline row added.
- [ ] activesupport message-pack suites green, including the encryption
      message-pack serializer tests.
