---
paths:
  - "spec/**"
  - "**/*_spec.rb"
  - "**/spec_helper.rb"
---

# Ruby Testing Rules

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
