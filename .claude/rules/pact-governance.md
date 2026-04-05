---
paths:
  - "lib/**/pact/**"
  - "lib/**/governance/**"
---

# PACT Governance Rules

### 1. Frozen GovernanceContext

Agents MUST receive a frozen `GovernanceContext`, NEVER `GovernanceEngine`.

```ruby
# DO:
ctx = Kailash::Pact::GovernanceContext.new(envelope: envelope, engine: engine, frozen: true)
agent.set_governance(ctx)

# DO NOT:
agent.set_governance(engine)  # Exposes mutable engine — self-modification attack vector
```

**Why:** If an agent receives the engine directly, it can modify its own governance constraints at runtime.

### 2. Monotonic Tightening

Child envelopes MUST be equal to or more restrictive than parent. `intersect_envelopes` takes min/intersection.

```ruby
# DO:
child_envelope = Kailash::Pact.intersect_envelopes(parent_envelope, requested_envelope)

# DO NOT:
child_envelope = requested_envelope  # Bypasses parent constraints
child_envelope.max_cost = parent_envelope.max_cost + 100  # Widening forbidden
```

**Why:** A child can never have more permissions than its parent — violating this allows privilege escalation.

### 3. D/T/R Grammar

Every Department or Team MUST be followed by exactly one Role in any Address.

**Why:** An address without a Role resolves to no envelope, causing `verify_action` to fail-closed and blocking all agent operations under that department.

```ruby
# DO:
address = Kailash::Pact::Address.new(org: "acme", dept: "engineering", role: "developer")

# DO NOT:
address = Kailash::Pact::Address.new(org: "acme", dept: "engineering")  # Missing Role
```

### 4. Fail-Closed Decisions

All `verify_action` and `check_access` error paths MUST return BLOCKED/DENY.

**Why:** A fail-open error path means any exception (network timeout, config error, nil reference) silently grants full access to the requesting agent.

```ruby
# DO:
def verify_action(action)
  envelope = resolve_envelope(action.agent_address)
  evaluate(action, envelope)
rescue => e
  Kailash::Pact::Decision::BLOCKED
end

# DO NOT:
rescue Kailash::Pact::EnvelopeNotFoundError
  Kailash::Pact::Decision::ALLOWED  # Fail-open!
```

### 5. Default-Deny Tool Registration

Tools MUST be registered via `register_tool`. Unregistered tools are BLOCKED.

**Why:** Default-allow means any new tool added to the system is immediately usable by all agents without governance review, bypassing the entire envelope model.

```ruby
# DO:
engine.register_tool("web_search", Kailash::Pact::ToolPolicy.new(allowed_roles: ["researcher"]))

# DO NOT:
return true unless @registered_tools.key?(tool_name)  # Default-allow — DANGEROUS
```

### 6. NaN/Infinity on Financial Fields

All numeric constraint fields MUST be validated with `Float#finite?`.

```ruby
# DO:
raise ArgumentError, "max_cost must be finite" if max_cost && !max_cost.finite?

# DO NOT:
raise ArgumentError, "negative" if max_cost && max_cost < 0  # NaN passes silently
```

**Why:** `Float::NAN < x` is always `false` — budget checks pass silently.

### 7. Compilation Limits

Org compilation MUST enforce: `MAX_COMPILATION_DEPTH` (50), `MAX_CHILDREN_PER_NODE` (500), `MAX_TOTAL_NODES` (100,000).

**Why:** Without limits, a deeply nested or circular org tree causes stack overflow or unbounded memory allocation during envelope compilation.

### 8. Thread Safety

All `GovernanceEngine` and store methods MUST acquire `@mutex` before accessing shared state.

**Why:** Puma runs requests on multiple threads sharing the same process; unsynchronized Hash access causes corrupted reads and lost writes under concurrency.

```ruby
def resolve_envelope(address)
  @mutex.synchronize { @envelopes[address.key] }
end
```

## MUST NOT

- Expose `GovernanceEngine` to agent code — agents receive `GovernanceContext` only
  **Why:** Engine access lets an agent call `register_tool` or modify envelopes at runtime, enabling self-granted privilege escalation.
- Bypass monotonic tightening — no code path may widen a child envelope beyond parent
  **Why:** A widened child envelope breaks the invariant that delegation never increases permissions, allowing a sub-agent to exceed its parent's authority.
- Use bare exceptions for governance errors — all MUST inherit `Kailash::Pact::PactError` with `#details`
  **Why:** Bare `RuntimeError` or `StandardError` is indistinguishable from non-governance failures, making it impossible to programmatically handle policy violations.
