# /test - Ruby Testing Quick Reference

## 3-Tier Testing

| Tier | Tool | Mocking | Infrastructure |
|------|------|---------|---------------|
| 1 - Unit | RSpec | Allowed | Isolated |
| 2 - Integration | RSpec | NO | Real Kailash runtime |
| 3 - E2E | RSpec + real DB | NO | Real everything |

## Integration Test Pattern

```ruby
RSpec.describe "Workflow" do
  let(:registry) { Kailash::Registry.new }
  after { registry.close }

  it "executes" do
    builder = Kailash::WorkflowBuilder.new
    builder.add_node("NoOpNode", "n", {})
    wf = builder.build(registry)

    Kailash::Runtime.open(registry) do |rt|
      result = rt.execute(wf, {})
      expect(result.results).to have_key("n")
    end
    wf.close
  end
end
```

## Regression Test Pattern

```ruby
# spec/regression/issue_42_spec.rb
RSpec.describe "Issue #42" do
  it "preserves explicit ID" do
    # reproduce exact bug...
    expect(result["id"]).to eq("custom-id")
  end
end
```

## Run Tests

```bash
bundle exec rspec                    # All tests
bundle exec rspec spec/integration/  # Integration only
bundle exec rspec --tag regression   # Regression only
```

## Rules

- NO mocking in Tiers 2-3 — real Kailash runtime required
- Always close Registry/Workflow in `after` hooks
- String keys in hashes, not symbols
