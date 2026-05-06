---
name: kailash-mcp
description: "Kailash MCP (Ruby): server, client, tools, resources, prompts, auth, transports (stdio/SSE/HTTP), testing. Use for AI agent integration."
---

# Kailash MCP - Model Context Protocol Integration (Ruby)

Production-ready MCP server implementation built into Kailash Core SDK for seamless AI agent integration. The Ruby gem wraps the Rust MCP engine via native extensions.

## Overview

Kailash's MCP module provides:

- **Full MCP Specification**: Complete implementation of Model Context Protocol
- **Multiple Transports**: stdio, SSE, HTTP support
- **Structured Tools**: Type-safe tool definitions via Ruby blocks
- **Resource Management**: Expose data sources to AI agents
- **Prompt Templates**: Pre-defined prompt templates for agents
- **Authentication**: Secure MCP server access
- **Progress Reporting**: Real-time operation status
- **Nexus Integration**: Automatic MCP channel when deployed via Nexus

## Install

```bash
gem install kailash-mcp
```

Or in Gemfile:

```ruby
gem "kailash-mcp"
```

## Quick Start

### Block-Based Tool Registration

```ruby
require "kailash/mcp"

server = Kailash::MCP::Server.new("my-server", "1.0")

# Register tools with blocks
server.tool("greet", description: "Greet someone") do |params|
  "Hello, #{params[:name]}!"
end

server.tool("search", description: "Search the web") do |params|
  results = perform_search(params[:query])
  { results: results, count: results.length }
end

# Run server (stdio transport by default)
server.run
```

### Resources

```ruby
require "kailash/mcp"

server = Kailash::MCP::Server.new("data-server", "1.0")

# Expose static resources
server.resource("config://settings", name: "Settings") do |_uri|
  '{"theme": "dark", "language": "en"}'
end

# Expose dynamic resources
server.resource("data://users", name: "Users") do |_uri|
  db.express.list("User").to_json
end

server.run
```

### Prompt Templates

```ruby
require "kailash/mcp"

server = Kailash::MCP::Server.new("prompt-server", "1.0")

server.prompt("summarize", description: "Summarize text") do |arguments|
  [{ role: "user", content: "Please summarize: #{arguments[:text]}" }]
end

server.prompt("code_review", description: "Review code") do |arguments|
  [
    { role: "system", content: "You are a senior code reviewer." },
    { role: "user", content: "Review this code:\n#{arguments[:code]}" }
  ]
end

server.run
```

## Key Concepts

### MCP Protocol

The Model Context Protocol enables AI agents to:

- **Tools**: Call structured functions
- **Resources**: Access data sources
- **Prompts**: Use pre-defined templates
- **Sampling**: Request LLM completions

### Transports

MCP supports multiple transport mechanisms:

- **stdio**: Standard input/output (default, simplest)
- **SSE**: Server-Sent Events (for web clients)
- **HTTP**: RESTful API (for HTTP clients)

```ruby
# stdio (default)
server.run

# SSE transport
server.run(transport: :sse, port: 8080)

# HTTP transport
server.run(transport: :http, port: 8080)
```

### Structured Tools

Tools are type-safe functions exposed to AI agents:

```ruby
server.tool("calculate", description: "Perform calculation",
  schema: {
    type: "object",
    properties: {
      expression: { type: "string", description: "Math expression" }
    },
    required: ["expression"]
  }
) do |params|
  eval(params[:expression]).to_s  # Simplified example
end
```

## Integration Patterns

### With Core SDK (Workflow Tools)

```ruby
require "kailash"
require "kailash/mcp"

server = Kailash::MCP::Server.new("workflow-server", "1.0")

server.tool("process_data", description: "Process data via workflow") do |params|
  registry = Kailash::Registry.new
  builder = Kailash::WorkflowBuilder.new
  builder.add_node("TransformNode", "transform", { "input" => params[:data] })
  workflow = builder.build(registry)

  Kailash::Runtime.open(registry) do |rt|
    result = rt.execute(workflow, {})
    result.results["transform"]["result"]
  end
ensure
  workflow&.close
  registry&.close
end

server.run
```

### With Nexus (Multi-Channel with MCP)

```ruby
require "kailash/nexus"

# Nexus automatically creates MCP channel
app = Kailash::Nexus::App.new(port: 3000, enable_mcp: true)

app.handler("summarize", description: "Summarize text") do |params|
  { summary: params[:text][0..99] }
end

app.start  # Includes MCP server
```

### With DataFlow (Database Access)

```ruby
require "kailash/mcp"
require "kailash/dataflow"

db = Kailash::DataFlow.new do |config|
  config.database_url = ENV["DATABASE_URL"]
end

server = Kailash::MCP::Server.new("db-server", "1.0")

server.resource("data://users", name: "Users") do |_uri|
  db.express.list("User").to_json
end

server.tool("create_user", description: "Create a user") do |params|
  db.express.create("User", name: params[:name], email: params[:email])
end

server.run
```

### With Kaizen (Agent Tools)

```ruby
require "kailash/mcp"
require "kailash/kaizen"

server = Kailash::MCP::Server.new("agent-server", "1.0")

server.tool("analyze", description: "Analyze text with AI") do |params|
  delegate = Kailash::Kaizen::Delegate.new(model: ENV["LLM_MODEL"])
  delegate.run_sync("Analyze: #{params[:text]}")
end

server.run
```

## Critical Rules

- Use stdio transport for local development
- Define clear tool schemas for complex inputs
- Implement progress reporting for long operations
- Test MCP servers with real MCP clients
- Use authentication for production servers
- Use block form for tool/resource registration (ensures proper cleanup)
- NEVER expose sensitive data without authentication
- NEVER skip input validation
- NEVER mock MCP protocol in tests (use real transports)

## Transport Selection

| Transport | Use Case         | Pros              | Cons          |
| --------- | ---------------- | ----------------- | ------------- |
| **stdio** | Local tools, CLI | Simple, reliable  | Local only    |
| **SSE**   | Web apps         | Real-time updates | Complex setup |
| **HTTP**  | APIs, services   | Standard protocol | No streaming  |

## Version Compatibility

- **MCP Specification**: Latest
- **Ruby**: 3.1+
- **Transports**: stdio, SSE, HTTP

## Related Skills

- **[01-core-sdk](../01-core-sdk/SKILL.md)** - Core workflow patterns
- **[02-dataflow](../02-dataflow/SKILL.md)** - Database resources
- **[03-nexus](../03-nexus/SKILL.md)** - Nexus includes MCP channel
- **[04-kaizen](../04-kaizen/SKILL.md)** - AI agents as MCP tools

## Support

For MCP-specific questions, invoke:

- `mcp-specialist` - MCP server implementation
- `testing-specialist` - MCP testing strategies
- ``decide-framework` skill` - MCP integration architecture
