---
paths:
  - "spec/**"
  - "**/*_spec.rb"
  - "**/spec_helper.rb"
  - "**/.spec-coverage*"
  - "**/.test-results*"
  - "**/02-plans/**"
  - "**/04-validate/**"
---

# Ruby Testing Rules

## Audit Mode Rules (Red Team / /redteam)

When auditing test coverage, do NOT trust prior round outputs. Re-derive everything.

### MUST: Re-derive coverage from scratch each audit round

```bash
# DO: re-derive
bundle exec rspec --dry-run --format documentation
ls spec/**/*_spec.rb | wc -l

# DO NOT: trust the file
cat .test-results  # BLOCKED in audit mode
```

**Why:** A previous round may have written `.test-results` claiming tests pass — true, but those tests may cover OLD code while new spec modules have zero tests.

### MUST: Verify NEW modules have NEW specs

For every new module a spec creates, grep `spec/` for a corresponding `*_spec.rb` file. Zero specs = HIGH finding regardless of "tests pass".

```bash
# DO
ls spec/**/*_kaizen_wrapper_spec.rb 2>/dev/null
# Empty → HIGH: new module has zero spec coverage
```

**Why:** Counting passing examples at the suite level lets new functionality ship with zero coverage as long as legacy specs still pass. Per-module verification catches this.

### MUST: Verify security mitigations have specs

For every § Security Threats subsection in any spec, grep for a corresponding `*_security_spec.rb` or matching example. Missing = HIGH.

**Why:** Documented threats with no spec become "we said we'd handle it" claims that nothing actually verifies.

See `skills/spec-compliance/SKILL.md` for the full spec compliance verification protocol.

### 1. Real Infrastructure in Integration Tests

Integration and E2E specs MUST use real Kailash runtime, not mocks:

```ruby
# DO:
RSpec.describe "Workflow execution" do
  let(:registry) { Kailash::Registry.new }
  after { registry.close }

  it "executes workflow" do
    builder = Kailash::WorkflowBuilder.new
    builder.add_node("NoOpNode", "n1", {})
    wf = builder.build(registry)

    Kailash::Runtime.open(registry) do |rt|
      result = rt.execute(wf, {})
      expect(result.results).to have_key("n1")
    end
    wf.close
  end
end

# DO NOT:
let(:runtime) { instance_double(Kailash::Runtime) }
```

**Why:** Mocks hide FFI boundary issues. The native extension must be exercised in tests.

### 2. Cleanup in After Hooks

MUST close native objects in `after` hooks:

```ruby
let(:registry) { Kailash::Registry.new }
let(:workflow) { builder.build(registry) }

after do
  workflow&.close
  registry&.close
end
```

**Why:** RSpec `let` blocks are lazy. Without explicit cleanup, native resources leak across examples.

### 3. Regression Tests for Bug Fixes

Every bug fix MUST include a regression spec.

**Why:** Without a regression spec, the same bug silently re-appears in a future refactor with no signal until a user reports it again.

```ruby
# spec/regression/issue_42_spec.rb
RSpec.describe "Issue #42: explicit ID preserved" do
  it "does not silently drop the explicit id" do
    expect(result["id"]).to eq("custom-id-value")
  end
end
```

## Test Organization

```
spec/
├── regression/     # Permanent bug reproduction tests
├── unit/           # Isolated, mocking allowed
├── integration/    # Real Kailash runtime, no mocking
└── spec_helper.rb  # Shared setup
```

## MUST NOT

No mocking Kailash types in integration specs:

```ruby
# DO NOT:
allow(registry).to receive(:list_types)
instance_double(Kailash::Runtime)
class_double(Kailash::Registry)
```

**Why:** The FFI boundary is the most likely failure point. Mocking it defeats integration testing.
