---
paths:
  - "lib/**/trust/**"
  - "lib/**/eatp/**"
---

# EATP SDK Rules (Ruby)

## Data Types

- Use frozen `Struct` or plain classes with `attr_reader` (NOT `OpenStruct` or mutable hashes)
  **Why:** `OpenStruct` allocates a new method per attribute access, is 10x slower than Struct, and was deprecated for this reason in Ruby 3.4+.
- Every data type MUST have `#to_h` and `.from_hash(hash)` class method
  **Why:** Without round-trip serialization methods, EATP records cannot be persisted to JSON stores or transmitted across SDK boundaries.
- Enums serialize as string value, timestamps as `.iso8601`
  **Why:** Symbol serialization produces Ruby-specific `:symbol` notation that other SDKs (Python, Rust) cannot parse, breaking cross-SDK interop.

```ruby
VerificationResult = Struct.new(:status, :timestamp, :details, keyword_init: true) do
  def to_h
    { status: status.to_s, timestamp: timestamp.iso8601, details: details }
  end

  def self.from_hash(hash)
    new(status: hash[:status].to_sym, timestamp: Time.parse(hash[:timestamp]), details: hash[:details])
  end
end
```

## Module Structure

- `# frozen_string_literal: true` at top of every file
  **Why:** Without the frozen literal pragma, every string allocation creates a new mutable object, increasing GC pressure in hot paths like trust verification loops.
- `# Copyright 2026 Terrene Foundation` + `# SPDX-License-Identifier: Apache-2.0` header
  **Why:** Missing SPDX headers make the gem ineligible for automated license scanners, blocking adoption in enterprises with strict OSS compliance policies.
- Explicit module nesting under `Kailash::Trust` or `Kailash::EATP`
  **Why:** Implicit nesting via `class Kailash::Trust::Foo` skips the enclosing module's constants and refinements, causing silent `NameError` when referencing sibling classes.
- String-backed constants for JSON-friendly serialization
  **Why:** Symbol constants require `.to_s` on every serialization path; forgetting one produces `:symbol` in JSON output that breaks cross-SDK deserialization.

## Error Handling

- All errors MUST inherit from `Kailash::Trust::TrustError`
  **Why:** Non-inheriting errors escape `rescue Kailash::Trust::TrustError` handlers, bubbling up as unhandled exceptions that crash the process instead of triggering fail-closed logic.
- All errors MUST include `#details` returning `Hash`
  **Why:** Without structured `#details`, error handlers cannot programmatically inspect failure context, forcing string-parsing of `#message` for diagnostics.
- Fail-closed: unknown/error states → deny, NEVER silently permit
  **Why:** A permissive fallback on unknown state means any new, untested trust state automatically grants access, turning bugs into security holes.

```ruby
class TrustError < Kailash::Error
  attr_reader :details
  def initialize(message, details: {})
    @details = details.freeze
    super(message)
  end
end
```

## Cryptography

- Ed25519 mandatory signing algorithm — use `ed25519` gem
  **Why:** Ed25519 is the EATP-mandated algorithm across all three SDKs; using a different curve breaks cross-SDK signature verification.
- HMAC optional overlay (HMAC alone NEVER sufficient for external verification)
  **Why:** HMAC proves message integrity but not origin authenticity, so accepting HMAC-only verification allows any party with the shared key to forge trust records.
- Constant-time comparison via `OpenSSL.fixed_length_secure_compare` — NEVER `==`
  **Why:** Ruby's `==` on strings short-circuits on the first differing byte, enabling timing side-channel attacks that extract HMAC digests one character at a time.

## Trust Model

- Monotonic escalation only: AUTO_APPROVED → FLAGGED → HELD → BLOCKED (never downgrade)
  **Why:** Allowing downgrade lets a compromised agent reset its trust state to AUTO_APPROVED, bypassing all governance escalation.
- Bounded collections: `max_size: 10_000`, trim oldest 10% at capacity
  **Why:** Unbounded trust record collections grow monotonically in long-running processes, eventually causing OOM when the Ruby heap cannot accommodate the audit trail.
- `nil` role = all-access (backward-compatible)
  **Why:** Treating `nil` as "no access" would break all pre-PACT code that omits role, so `nil` must mean unrestricted for backward compatibility.

## Cross-SDK Alignment

- Python, Rust, and Ruby SDKs implement spec independently (D6)
  **Why:** Shared implementation code between SDKs creates coupling that prevents each language from using idiomatic patterns and release cadences.
- Semantics MUST match across SDKs even if naming conventions differ
  **Why:** Divergent semantics mean a trust record created by the Python SDK cannot be verified by the Ruby SDK, breaking the cross-platform trust chain.
- Ruby wraps Rust SDK via native extensions — EATP crypto may delegate to Rust layer
  **Why:** Delegating crypto to the Rust layer avoids maintaining a separate Ruby crypto implementation and inherits Rust's memory-safe, constant-time guarantees.
