# /sdk - Kailash Ruby SDK Quick Reference

## Install

```bash
gem install kailash
# Or in Gemfile:
gem "kailash"
```

## Minimal Workflow

```ruby
require "kailash"

registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new
builder.add_node("NoOpNode", "start", {})
wf = builder.build(registry)

Kailash::Runtime.open(registry) do |rt|
  result = rt.execute(wf, {})
  puts result.run_id
end

wf.close
registry.close
```

## Key Types

| Type | Purpose |
|------|---------|
| `Kailash::Registry` | Node type registry |
| `Kailash::WorkflowBuilder` | Build workflow DAGs |
| `Kailash::Runtime` | Execute workflows |
| `Kailash::Workflow` | Compiled workflow |

## Common Patterns

```ruby
# Multi-node workflow
builder.add_node("HttpRequestNode", "fetch", { "url" => url, "method" => "GET" })
builder.add_node("LogNode", "log", { "message" => "Done" })
builder.connect("fetch", "log")

# Parameters
result = rt.execute(wf, { "api_key" => ENV["API_KEY"] })

# Error handling
begin
  wf = builder.build(registry)
rescue Kailash::BuildError => e
  puts e.message
end
```

## Skills

Full documentation: `.claude/skills/01-core-sdk/SKILL.md`
