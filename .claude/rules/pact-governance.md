---
paths:
  - "lib/**/pact/**"
  - "lib/**/governance/**"
---

# PACT Governance Rules

## Scope

These rules apply when editing `lib/**/pact/**` files.

These rules supplement `.claude/rules/security.md` and `.claude/rules/trust-plane-security.md`. All three apply to PACT governance files.
Violations during code review by intermediate-reviewer are BLOCK-level findings.

## MUST Rules

### 1. Frozen GovernanceContext

Agents MUST receive a frozen `GovernanceContext`, NEVER `GovernanceEngine`. The engine reference is private.

```ruby
# DO:
ctx = Kailash::Pact::GovernanceContext.new(
  envelope: envelope,
  engine: engine,
  frozen: true
)
agent.set_governance(ctx)

# DO NOT:
agent.set_governance(engine)  # Exposes mutable engine — self-modification attack vector
agent.instance_variable_set(:@engine, engine)  # Private field bypass — same risk
```

**Why**: `GovernanceContext` is an immutable view. If an agent receives the engine directly, it can modify its own governance constraints at runtime — defeating the entire trust model. Use `freeze` to enforce immutability.

```ruby
class GovernanceContext
  attr_reader :envelope, :resolved_address

  def initialize(envelope:, engine:, frozen: false)
    @envelope = envelope
    @_engine = engine  # Private, no accessor
    self.freeze if frozen
  end
end
```

### 2. Monotonic Tightening

Child envelopes MUST be equal to or more restrictive than parent envelopes. `intersect_envelopes` takes the min/intersection of every field.

```ruby
# DO:
child_envelope = Kailash::Pact.intersect_envelopes(parent_envelope, requested_envelope)
# child_envelope.max_cost <= parent_envelope.max_cost (always)
# child_envelope.allowed_tools is a subset of parent_envelope.allowed_tools (always)

# DO NOT:
child_envelope = requested_envelope  # Bypasses parent constraints entirely
child_envelope.max_cost = parent_envelope.max_cost + 100  # Widening is forbidden
```

**Why**: Governance flows downward through the org hierarchy. A child can never have more permissions than its parent — this is the monotonic tightening invariant. Violating it allows privilege escalation through delegation.

### 3. D/T/R Grammar

Every Department or Team MUST be immediately followed by exactly one Role in any Address.

```ruby
# DO:
address = Kailash::Pact::Address.new(org: "acme", dept: "engineering", role: "developer")
address = Kailash::Pact::Address.new(org: "acme", dept: "engineering", team: "backend", role: "senior")

# DO NOT:
address = Kailash::Pact::Address.new(org: "acme", dept: "engineering")                    # Missing Role
address = Kailash::Pact::Address.new(org: "acme", dept: "engineering", team: "backend")   # Missing Role after Team
```

**Why**: The D/T/R grammar ensures every address resolves to a concrete governance envelope. An address without a terminal Role is ambiguous — it could match multiple envelopes with different constraints.

### 4. Fail-Closed Decisions

All `verify_action` and `check_access` error paths MUST return BLOCKED/DENY, never raise or return permissive results.

```ruby
# DO:
def verify_action(action)
  envelope = resolve_envelope(action.agent_address)
  evaluate(action, envelope)
rescue => e
  Kailash::Pact::Decision::BLOCKED  # Fail-closed on ANY error
end

# DO NOT:
def verify_action(action)
  envelope = resolve_envelope(action.agent_address)
  evaluate(action, envelope)
rescue Kailash::Pact::EnvelopeNotFoundError
  Kailash::Pact::Decision::ALLOWED  # Fail-open — missing envelope permits everything!
rescue => e
  raise  # Unhandled exception bypasses governance entirely
end
```

**Why**: Governance is a security boundary. An error in constraint resolution or evaluation MUST NOT result in permissive access. Unknown states are denied — this is the same principle as trust-plane-security.md's monotonic escalation.

### 5. Default-Deny Tool Registration

Tools MUST be explicitly registered via `register_tool` before execution. Unregistered tools are BLOCKED.

```ruby
# DO:
engine.register_tool("web_search", Kailash::Pact::ToolPolicy.new(allowed_roles: ["researcher"]))
engine.register_tool("code_execute", Kailash::Pact::ToolPolicy.new(allowed_roles: ["developer"]))
# Only registered tools can be invoked

# DO NOT:
def check_tool(tool_name)
  return true unless @registered_tools.key?(tool_name)  # Unknown tool — allow by default (DANGEROUS)
  # ...
end
```

**Why**: Default-allow for tools means any new or misspelled tool name bypasses governance. The tool registry is the allowlist — anything not on it is denied.

### 6. NaN/Infinity on Financial Fields

All numeric constraint fields MUST be validated with `Float#finite?`. Extends trust-plane-security.md rule 3 to governance envelopes.

```ruby
# DO:
def validate!
  unless max_cost.nil? || max_cost.finite?
    raise ArgumentError, "max_cost must be finite"
  end
  unless budget_limit.nil? || budget_limit.finite?
    raise ArgumentError, "budget_limit must be finite"
  end
end

# DO NOT:
def validate!
  raise ArgumentError, "negative" if max_cost && max_cost < 0
  # NaN < 0 is false — NaN passes silently!
end
```

**Why**: `Float::NAN` poisons all numeric comparisons (`Float::NAN < x` is always `false`, `Float::NAN > x` is always `false`). If `NaN` enters a governance envelope's financial field, all budget checks pass silently. `Float::INFINITY` similarly defeats upper-bound checks. Every numeric field in `GovernanceEnvelope` MUST validate with `#finite?`.

### 7. Compilation Limits

Org compilation MUST enforce `MAX_COMPILATION_DEPTH` (50), `MAX_CHILDREN_PER_NODE` (500), `MAX_TOTAL_NODES` (100,000).

```ruby
# DO:
MAX_COMPILATION_DEPTH  = 50
MAX_CHILDREN_PER_NODE  = 500
MAX_TOTAL_NODES        = 100_000

def compile_org(root, depth: 0)
  if depth > MAX_COMPILATION_DEPTH
    raise Kailash::Pact::CompilationError, "Max depth #{MAX_COMPILATION_DEPTH} exceeded"
  end
  if root.children.size > MAX_CHILDREN_PER_NODE
    raise Kailash::Pact::CompilationError, "Max children #{MAX_CHILDREN_PER_NODE} exceeded"
  end
  @total_nodes += 1
  if @total_nodes > MAX_TOTAL_NODES
    raise Kailash::Pact::CompilationError, "Max total nodes #{MAX_TOTAL_NODES} exceeded"
  end
  root.children.each { |child| compile_org(child, depth: depth + 1) }
end

# DO NOT:
def compile_org(root)
  root.children.each { |child| compile_org(child) }  # No depth limit — stack overflow
  # No node count limit — OOM on large orgs
end
```

**Why**: Org trees can be arbitrarily deep and wide. Without compilation limits, a malicious or malformed org definition can cause stack overflow (unbounded recursion), memory exhaustion (millions of nodes), or CPU exhaustion (exponential traversal). These limits are defense-in-depth against denial-of-service.

### 8. Thread Safety

All `GovernanceEngine` and store methods MUST acquire a lock before accessing shared state. Use `Mutex` for thread safety.

```ruby
# DO:
class GovernanceEngine
  def initialize
    @mutex = Mutex.new
    @envelopes = {}
  end

  def resolve_envelope(address)
    @mutex.synchronize do
      @envelopes[address.key]
    end
  end

  def update_envelope(address, envelope)
    @mutex.synchronize do
      @envelopes[address.key] = envelope
    end
  end
end

# DO NOT:
class GovernanceEngine
  def resolve_envelope(address)
    @envelopes[address.key]  # Race condition — concurrent reads/writes corrupt state
  end
end
```

**Why**: `GovernanceEngine` is shared across agent threads. Without locking, concurrent envelope updates and reads can produce torn reads (partially updated envelopes) or lost updates. Both lead to incorrect governance decisions.

## MUST NOT Rules

### 1. MUST NOT Expose GovernanceEngine to Agent Code

Agent code receives `GovernanceContext` only. Direct `GovernanceEngine` access enables self-modification attacks.

```ruby
# DO:
class Agent
  def initialize(ctx:)
    @ctx = ctx  # Frozen, read-only view
  end
end

# DO NOT:
class Agent
  def initialize(engine:)
    @engine = engine  # Agent can call engine.update_envelope on itself!
  end
end
```

**Why**: If an agent holds a reference to the engine, it can modify its own governance envelope — removing tool restrictions, raising budget limits, or escalating privileges. The `GovernanceContext` wrapper provides read-only access to the agent's resolved envelope without exposing mutation methods.

### 2. MUST NOT Bypass Monotonic Tightening

No code path may widen a child envelope beyond its parent.

```ruby
# DO:
child = Kailash::Pact.intersect_envelopes(parent, requested)
# child.max_cost <= parent.max_cost
# child.allowed_tools is a subset of parent.allowed_tools

# DO NOT:
child = Kailash::Pact::GovernanceEnvelope.new(max_cost: parent.max_cost * 2)  # Widening!
child.allowed_tools = parent.allowed_tools | Set["dangerous_tool"]              # Superset!
```

**Why**: Monotonic tightening is the foundational invariant of hierarchical governance. If any code path can widen a child envelope, the entire governance hierarchy is compromised — a leaf agent could accumulate permissions exceeding the root.

### 3. MUST NOT Use Bare Exception for Governance Errors

All governance errors MUST inherit from `Kailash::Pact::PactError` with structured `#details`.

```ruby
# DO:
class GovernanceViolationError < Kailash::Pact::PactError
  def initialize(message, details: {})
    @details = details.freeze
    super(message)
  end
end

raise GovernanceViolationError.new(
  "Budget exceeded",
  details: { max_cost: 100.0, attempted_cost: 150.0, agent: "agent-001" }
)

# DO NOT:
raise ArgumentError, "Budget exceeded"          # No structured details, wrong hierarchy
raise RuntimeError, "something went wrong"      # Bare exception — uncatchable by governance handlers
```

**Why**: Governance errors carry structured context (`details` hash) that audit systems, dashboards, and parent agents consume. Bare exceptions lose this context and cannot be distinguished from unrelated errors in `rescue` handlers.

## Cross-References

- `.claude/rules/trust-plane-security.md` — Trust-plane security patterns (NaN/Inf, bounded collections, fail-closed)
- `.claude/rules/eatp.md` — EATP SDK conventions (data types, error hierarchy, cryptography)
- `lib/**/pact/governance/agent.rb` — Anti-self-modification defense
- `lib/**/pact/governance/envelopes.rb` — Monotonic tightening
