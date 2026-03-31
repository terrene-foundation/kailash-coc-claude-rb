---
paths:
  - "lib/**/trust/**"
  - "lib/**/eatp/**"
---

# EATP SDK Rules

## Scope

These rules apply when editing `lib/**/trust/**` files (excluding `plane/`).

## SDK Conventions

### Data Types

- Use frozen `Struct` or plain classes with `attr_reader` for all data types (NOT `OpenStruct` or mutable hashes)
- Every data type MUST have `#to_h` returning `Hash` and `.from_hash(hash)` class method returning a new instance
- Enums serialize as their string value, timestamps as `.iso8601`

```ruby
# DO: Frozen Struct for immutable data
VerificationResult = Struct.new(:status, :timestamp, :details, keyword_init: true) do
  def to_h
    { status: status.to_s, timestamp: timestamp.iso8601, details: details }
  end

  def self.from_hash(hash)
    new(
      status: hash[:status].to_sym,
      timestamp: Time.parse(hash[:timestamp]),
      details: hash[:details]
    )
  end
end

# DO: Plain class with attr_reader for complex types
class TrustAttestation
  attr_reader :signer_id, :signature, :algorithm, :created_at

  def initialize(signer_id:, signature:, algorithm:, created_at:)
    @signer_id  = signer_id.freeze
    @signature  = signature.freeze
    @algorithm  = algorithm.freeze
    @created_at = created_at.freeze
    freeze
  end
end

# DO NOT: Mutable hash or OpenStruct
result = OpenStruct.new(status: "verified")  # No type safety, no freeze
result = { status: "verified" }               # No method interface, mutable
```

### Module Structure

- `# frozen_string_literal: true` at the top of every file
- `# Copyright 2026 Terrene Foundation` + `# SPDX-License-Identifier: Apache-2.0` header
- `require "logger"` with module-level `LOGGER = Logger.new($stdout)` or use `Kailash.logger`
- Explicit module nesting under `Kailash::Trust` or `Kailash::EATP`
- String-backed constants or symbol enums for JSON-friendly serialization

```ruby
# frozen_string_literal: true

# Copyright 2026 Terrene Foundation
# SPDX-License-Identifier: Apache-2.0

module Kailash
  module Trust
    module VerificationStatus
      VERIFIED   = "verified"
      FLAGGED    = "flagged"
      DENIED     = "denied"
    end
  end
end
```

### Error Handling

- All errors MUST inherit from `Kailash::Trust::TrustError` (or `Kailash::EATP::EatpError`)
- All errors MUST include a `#details` method returning `Hash`
- Fail-closed: unknown/error states -> deny, NEVER silently permit

```ruby
class TrustError < Kailash::Error
  attr_reader :details

  def initialize(message, details: {})
    @details = details.freeze
    super(message)
  end
end

class VerificationError < TrustError; end
class SignatureError < TrustError; end
```

### Cryptography

- Ed25519 is the mandatory signing algorithm — use the `ed25519` gem
- HMAC is optional overlay (HMAC alone is NEVER sufficient for external verification) — use `OpenSSL::HMAC` from stdlib
- Constant-time comparison via `OpenSSL.fixed_length_secure_compare` or `Rack::Utils.secure_compare` — NEVER use `==` for signature comparison
- AWS KMS uses ECDSA P-256 (Ed25519 not available in KMS) — document the algorithm mismatch

```ruby
require "ed25519"
require "openssl"

# DO: Ed25519 signing
signing_key = Ed25519::SigningKey.generate
signature = signing_key.sign(message)
verify_key = signing_key.verify_key
verify_key.verify(signature, message)

# DO: Constant-time comparison
OpenSSL.fixed_length_secure_compare(computed_hmac, received_hmac)

# DO NOT: Timing-vulnerable comparison
computed_hmac == received_hmac  # <-- BLOCKED: timing side-channel
```

### Trust Model

- Monotonic escalation only: AUTO_APPROVED -> FLAGGED -> HELD -> BLOCKED (never downgrade)
- Bounded collections: `max_size: 10_000`, trim oldest 10% at capacity
- `nil` role = all-access (backward-compatible, no RBAC enforcement)

### Cross-SDK Alignment

- Python, Rust, and Ruby SDKs implement the spec independently (D6)
- Convention names may differ (Ruby snake_case vs Python snake_case vs Rust snake_case) but semantics MUST match
- New spec-level concepts require cross-SDK team coordination before implementation
- Ruby wraps the Rust SDK via native extensions — EATP cryptographic operations may delegate to the Rust layer for performance
