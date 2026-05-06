---
name: kaizen
description: "Kailash Kaizen via Ruby Magnus binding — MANDATORY for AI agents/RAG/signatures/multi-agent. Raw LLM clients BLOCKED."
---

# Kailash Kaizen - AI Agent Framework (Ruby)

Kaizen is a production-ready AI agent framework built on Kailash Core SDK that provides signature-based programming and multi-agent coordination. The Ruby gem wraps the Rust Kaizen engine via native extensions.

## Features

- **Signature DSL**: Type-safe agent interfaces via Ruby blocks
- **BaseAgent Architecture**: Production-ready agent foundation with error handling, audit trails, cost tracking
- **Delegate Pattern**: Zero-boilerplate autonomous agents with streaming events
- **Multi-Agent Coordination**: Supervisor-worker, pipelines, ensemble, router patterns
- **Multi-Provider Support**: OpenAI, Anthropic, Google, Ollama via adapter registry
- **Tool System**: Register Ruby methods as agent tools with automatic schema generation
- **Memory System**: Session, shared, and persistent memory tiers
- **L3 Autonomy Primitives**: Envelope tracking, scoped context, agent factory, plan execution
- **PACT Governance**: GovernedSupervisor with progressive disclosure API
- **Budget Tracking**: Two-phase reserve/record with posture-budget integration
- **Enterprise Trust**: EATP trust chains, secure messaging, credential rotation

## Install

```bash
gem install kailash-kaizen
```

Or in Gemfile:

```ruby
gem "kailash-kaizen"
```

## Quick Start

### Delegate (Recommended for Autonomous Agents)

```ruby
require "kailash/kaizen"

delegate = Kailash::Kaizen::Delegate.new(model: ENV["LLM_MODEL"])

# Streaming execution with block
delegate.run("Analyze this data") do |event|
  case event
  when Kailash::Kaizen::TextDelta
    print event.text
  when Kailash::Kaizen::ToolCallStart
    puts "\nCalling tool: #{event.tool_name}"
  end
end

# Synchronous execution (for scripts/CLI)
result = delegate.run_sync("Summarize this document")
puts result
```

### BaseAgent (For Custom Agent Logic)

```ruby
require "kailash/kaizen"

class SummaryAgent < Kailash::Kaizen::BaseAgent
  signature do
    input :text, type: :string, description: "Text to summarize"
    output :summary, type: :string, description: "Generated summary"
  end

  configure do |config|
    config.model = "gpt-4"
    config.temperature = 0.7
  end

  def execute(inputs)
    # Agent logic here
    { summary: "Summary of: #{inputs[:text]}" }
  end
end

agent = SummaryAgent.new
result = agent.run(text: "Long text here...")
puts result[:summary]
```

### Pipeline Patterns (Orchestration)

```ruby
require "kailash/kaizen"

# Ensemble: Multi-perspective collaboration
pipeline = Kailash::Kaizen::Pipeline.ensemble(
  agents: [code_expert, data_expert, writing_expert],
  synthesizer: synthesis_agent,
  top_k: 3
)

result = pipeline.run(task: "Analyze codebase", input: "repo_path")

# Router: Intelligent task delegation
router = Kailash::Kaizen::Pipeline.router(
  agents: [code_agent, data_agent, writing_agent],
  routing_strategy: :semantic
)

# Supervisor-Worker: Hierarchical coordination
supervisor = Kailash::Kaizen::Pipeline.supervisor(
  supervisor: manager_agent,
  workers: [agent_a, agent_b, agent_c],
  strategy: :round_robin
)
```

## Key Concepts

### Signature DSL

Signatures define type-safe interfaces for agents using Ruby blocks:

```ruby
class MyAgent < Kailash::Kaizen::BaseAgent
  signature do
    input :query, type: :string, description: "User query"
    input :context, type: :hash, description: "Additional context", default: {}
    output :answer, type: :string, description: "Agent response"
    output :confidence, type: :float, description: "Confidence score"
  end
end
```

### Tool Registration

```ruby
delegate = Kailash::Kaizen::Delegate.new(model: ENV["LLM_MODEL"])

delegate.tool("search_web", description: "Search the web") do |params|
  # params[:query] available
  perform_search(params[:query])
end

delegate.tool("read_file", description: "Read a file") do |params|
  File.read(params[:path])
end
```

### GovernedSupervisor (PACT Governance)

```ruby
require "kailash/kaizen"

# Layer 1: Simple (2 params)
supervisor = Kailash::Kaizen::GovernedSupervisor.new(
  agents: [agent_a, agent_b],
  task: "Analyze the dataset"
)

# Layer 2: Configured (8 params)
supervisor = Kailash::Kaizen::GovernedSupervisor.new(
  agents: [agent_a, agent_b],
  task: "Analyze the dataset",
  budget: { max_tokens: 100_000, max_cost: 5.0 },
  strategy: :supervised,
  cascade: :monotonic
)

# Layer 3: Full governance (9 subsystems)
supervisor = Kailash::Kaizen::GovernedSupervisor.new(
  agents: [agent_a, agent_b],
  task: "Analyze the dataset",
  accountability: { tracker: true },
  budget: { max_tokens: 100_000, warnings: [0.8, 0.95] },
  cascade: { strategy: :monotonic },
  clearance: { level: :c2 },
  dereliction: { detect: true },
  bypass: { enabled: false },
  vacancy: { auto_designate: true },
  audit: { hash_chain: true }
)
```

### L3 Autonomy Primitives

```ruby
require "kailash/kaizen"

# Envelope tracking with gradient zones
tracker = Kailash::Kaizen::L3::EnvelopeTracker.new(
  budget: { max_tokens: 50_000 }
)

# Scoped context with access control
context = Kailash::Kaizen::L3::ScopedContext.new(
  projection: { allow: ["data.*"], deny: ["data.secret.*"] }
)

# Agent factory with lifecycle tracking
factory = Kailash::Kaizen::L3::AgentFactory.new
instance = factory.spawn(agent_spec, parent: supervisor)
```

## Memory System

```ruby
# Session memory (in-memory, per-conversation)
agent.memory.session.store("key", "value")
agent.memory.session.recall("key")

# Shared memory (across agents)
agent.memory.shared.store("shared_key", data)

# Persistent memory (DataFlow-backed, cross-session)
agent.memory.persistent.store("long_term", data)
```

## Integration Patterns

### With DataFlow (Data-Driven Agents)

```ruby
require "kailash/kaizen"
require "kailash/dataflow"

db = Kailash::DataFlow.new do |config|
  config.database_url = ENV["DATABASE_URL"]
end

delegate = Kailash::Kaizen::Delegate.new(model: ENV["LLM_MODEL"])
delegate.tool("query_users", description: "Query user database") do |params|
  db.express.list("User", filter: params[:filter])
end
```

### With Nexus (Multi-Channel Agents)

```ruby
require "kailash/kaizen"
require "kailash/nexus"

app = Kailash::Nexus::App.new(port: 3000)

app.handler("chat", description: "Chat with AI agent") do |params|
  delegate = Kailash::Kaizen::Delegate.new(model: ENV["LLM_MODEL"])
  result = delegate.run_sync(params[:message])
  { response: result }
end

app.start  # Agent accessible via API, CLI, and MCP
```

### With Core SDK (Custom Workflows)

```ruby
require "kailash"
require "kailash/kaizen"

registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new
builder.add_node("KaizenAgent", "agent1", {
  "agent" => "SummaryAgent",
  "input" => "Analyze this data"
})
```

## Critical Rules

- Define signatures before implementing agents
- Extend `Kailash::Kaizen::BaseAgent` for custom agents
- Use Delegate for autonomous tasks (zero boilerplate)
- Track costs in production environments
- Test agents with real LLM calls (real infrastructure recommended)
- Use block form for streaming to ensure proper cleanup
- NEVER skip signature definitions
- NEVER ignore cost tracking in production
- NEVER mock LLM calls in integration tests (real infrastructure recommended)

## When to Use This Skill

Use Kaizen when you need to:

- Build AI agents with type-safe interfaces
- Implement multi-agent systems with orchestration patterns
- Create autonomous agents with the Delegate pattern
- Implement chain-of-thought or ReAct reasoning
- Build supervisor-worker or ensemble architectures
- Track costs and performance of AI agents
- Add PACT governance to agent hierarchies
- Create production-ready agentic applications

**Pipeline pattern selection:**

- **Ensemble**: Diverse perspectives synthesized (code review, research)
- **Router**: Intelligent task delegation to specialists
- **Supervisor-Worker**: Hierarchical coordination with oversight
- **Parallel**: Bulk processing or voting-based consensus
- **Sequential**: Linear workflows with dependency chains

## Related Skills

- **[01-core-sdk](../01-core-sdk/SKILL.md)** - Core workflow patterns
- **[02-dataflow](../02-dataflow/SKILL.md)** - Database integration
- **[03-nexus](../03-nexus/SKILL.md)** - Multi-channel deployment
- **[05-kailash-mcp](../05-kailash-mcp/SKILL.md)** - MCP server integration

## Support

For Kaizen-specific questions, invoke:

- `kaizen-specialist` - Kaizen framework implementation
- `testing-specialist` - Agent testing strategies
- ``decide-framework` skill` - When to use Kaizen vs other frameworks
