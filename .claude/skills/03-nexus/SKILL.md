---
name: nexus
description: "Kailash Nexus (Ruby): zero-config multi-channel platform deploying workflows as API+CLI+MCP. Rack middleware, sessions, health, plugins, event system, workflow registration."
---

# Kailash Nexus - Multi-Channel Platform Framework (Ruby)

Nexus is a zero-config multi-channel platform built on Kailash Core SDK that deploys workflows as API + CLI + MCP simultaneously. The Ruby gem wraps the Rust Nexus engine via native extensions and provides Rack middleware integration.

## Features

- **Zero Configuration**: Deploy workflows instantly without boilerplate
- **Multi-Channel Access**: API, CLI, and MCP from a single deployment
- **Unified Sessions**: Consistent session management across all channels
- **Enterprise Features**: Health monitoring, plugins, event system, logging
- **DataFlow Integration**: Automatic CRUD API generation from database models
- **Rack Integration**: Mount Nexus as Rack middleware alongside Rails/Sinatra
- **Handler DSL**: Register Ruby blocks as handlers deployed to all channels

## Install

```bash
gem install kailash-nexus
```

Or in Gemfile:

```ruby
gem "kailash-nexus"
```

## Quick Start

```ruby
require "kailash/nexus"

# Create workflow
workflow = create_my_workflow

# Deploy to all channels at once
nexus = Kailash::Nexus.new(workflow)
nexus.run(port: 8000)

# Now available via:
# - HTTP API: POST http://localhost:8000/api/workflow/{workflow_id}
# - CLI: nexus run {workflow_id} --input '{"key": "value"}'
# - MCP: Connect via MCP client (Claude Desktop, etc.)
```

### Handler Registration

```ruby
require "kailash/nexus"

app = Kailash::Nexus::App.new(port: 3000)

# Register handler -- deployed to all channels at once
app.handler("greet", description: "Greet a user") do |params|
  { message: "Hello, #{params[:name]}!" }
end

app.start

# HTTP: POST http://localhost:3000/api/greet
# CLI:  nexus run greet --name "World"
# MCP:  Available as MCP tool
```

## Key Concepts

### Zero-Config Platform

Nexus eliminates boilerplate:

- **No manual routes** - Automatic API generation from workflows
- **No CLI arg parsing** - Automatic CLI creation
- **No MCP server setup** - Automatic MCP integration
- **Unified deployment** - One command for all channels

### Multi-Channel Architecture

Single deployment, three access methods:

1. **HTTP API**: RESTful JSON endpoints
2. **CLI**: Command-line interface
3. **MCP**: Model Context Protocol server

### Unified Sessions

Consistent session management:

- Cross-channel session tracking
- Session state persistence
- Session-scoped workflows
- Concurrent session support

### Enterprise Features

Production-ready capabilities:

- Health monitoring endpoints
- Plugin system for extensibility
- Event system for integrations
- Comprehensive logging and metrics

## Integration Patterns

### With DataFlow (Database-Backed Handlers)

```ruby
require "kailash/nexus"
require "kailash/dataflow"

db = Kailash::DataFlow.new do |config|
  config.database_url = ENV["DATABASE_URL"]
end

db.model "User" do |m|
  m.string :name
  m.string :email
end

# Auto-generates CRUD endpoints for all models
nexus = Kailash::Nexus.new(db.workflows)
nexus.run(port: 8000)

# GET    /api/User/list
# POST   /api/User/create
# GET    /api/User/read/:id
# PUT    /api/User/update/:id
# DELETE /api/User/delete/:id
```

### With Kaizen (Agent Platform)

```ruby
require "kailash/nexus"
require "kailash/kaizen"

app = Kailash::Nexus::App.new(port: 3000)

app.handler("agent_chat", description: "Chat with AI agent") do |params|
  delegate = Kailash::Kaizen::Delegate.new(model: ENV["LLM_MODEL"])
  result = delegate.run_sync(params[:message])
  { response: result }
end

app.start  # Agents accessible via API, CLI, and MCP
```

### With Core SDK (Custom Workflows)

```ruby
require "kailash"
require "kailash/nexus"

registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new
builder.add_node("NoOpNode", "start", {})
workflow = builder.build(registry)

nexus = Kailash::Nexus.new(workflow)
nexus.run(port: 8000)
```

### With Rack (Middleware)

```ruby
# config.ru
require "kailash/nexus"

nexus = Kailash::Nexus.new(my_workflows)

# Mount Nexus as Rack middleware
use Kailash::Nexus::Middleware, nexus: nexus
run MyRailsApp
```

### Standalone Platform

```ruby
require "kailash/nexus"

app = Kailash::Nexus::App.new(
  host: "0.0.0.0",
  port: 3000,
  preset: :enterprise
)

app.cors(origins: ["https://app.example.com"])
app.rate_limit(max_requests: 100, window_secs: 60)

app.handler("status", description: "Platform status") do |_params|
  app.health_check
end

app.start
```

## Deployment Patterns

### Development

```ruby
nexus = Kailash::Nexus.new(workflows)
nexus.run(port: 8000)  # Single process
```

### Production (Docker)

```ruby
app = Kailash::Nexus::App.new(
  host: "0.0.0.0",
  port: 3000,
  preset: :enterprise
)

# Register handlers...

app.start  # Host/port configured at init
```

### With Puma (Multi-Worker)

```ruby
# config/puma.rb
workers 4
threads 2, 4
port 3000

# config.ru
require "kailash/nexus"
nexus = Kailash::Nexus.new(my_workflows)
use Kailash::Nexus::Middleware, nexus: nexus
run MyApp
```

### With Load Balancer

```bash
# Deploy multiple Nexus instances behind nginx/traefik
docker-compose up --scale nexus=3
```

## Critical Rules

- Use Nexus for workflow platforms instead of building raw Rack/Sinatra/Rails API routes
- Register workflows or handlers, not individual routes
- Leverage unified sessions across channels
- Enable health monitoring in production
- Use plugins for custom behavior
- NEVER mix raw Rack routes with Nexus for the same endpoints
- NEVER implement manual API/CLI/MCP servers when Nexus can do it
- NEVER skip health checks in production

## Channel Comparison

| Feature       | API  | CLI       | MCP         |
| ------------- | ---- | --------- | ----------- |
| **Access**    | HTTP | Terminal  | MCP Clients |
| **Input**     | JSON | Args/JSON | Structured  |
| **Output**    | JSON | Text/JSON | Structured  |
| **Sessions**  | yes  | yes       | yes         |
| **Auth**      | yes  | yes       | yes         |
| **Streaming** | yes  | yes       | yes         |

## Related Skills

- **[01-core-sdk](../01-core-sdk/SKILL.md)** - Core workflow patterns
- **[02-dataflow](../02-dataflow/SKILL.md)** - Auto CRUD API generation
- **[04-kaizen](../04-kaizen/SKILL.md)** - AI agent deployment
- **[05-kailash-mcp](../05-kailash-mcp/SKILL.md)** - MCP channel details

## Support

For Nexus-specific questions, invoke:

- `nexus-specialist` - Nexus implementation and deployment
- `release-specialist` - Production deployment patterns
- ``decide-framework` skill` - When to use Nexus vs other approaches
