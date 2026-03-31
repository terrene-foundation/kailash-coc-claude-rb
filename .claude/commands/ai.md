# /ai - Kaizen Quick Reference (Ruby)

## AI Agent Basics

```ruby
require "kailash"

registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new

# AI completion node
builder.add_node("LlmCompletionNode", "ask", {
  "model" => ENV["LLM_MODEL"],
  "prompt" => "Summarize this document"
})

wf = builder.build(registry)
Kailash::Runtime.open(registry) do |rt|
  result = rt.execute(wf, { "document" => text })
  puts result.results["ask"]["response"]
end
```

## Environment Variables

```bash
# .env
LLM_MODEL=claude-sonnet-4-20250514
ANTHROPIC_API_KEY=sk-ant-...
# OR
OPENAI_API_KEY=sk-...
```

## Key Rules

- ALL model names from `.env` — never hardcode
- ALL API keys from environment variables
- Use `dotenv` gem for local development
