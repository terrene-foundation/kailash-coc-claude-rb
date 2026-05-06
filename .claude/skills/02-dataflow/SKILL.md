---
name: dataflow
description: "Kailash DataFlow via Ruby Magnus binding — MANDATORY for DB/CRUD/bulk/migrations/multi-tenancy. Raw SQL/ORMs BLOCKED."
---

# Kailash DataFlow - Zero-Config Database Framework (Ruby)

DataFlow is a zero-config database framework built on Kailash Core SDK that automatically generates workflow nodes from database models. The Ruby gem wraps the Rust DataFlow engine via native extensions.

## Overview

- **Automatic Node Generation**: 11 nodes per model (DSL model definition)
- **Multi-Database Support**: PostgreSQL, MySQL, SQLite (SQL)
- **Enterprise Features**: Multi-tenancy, multi-instance isolation, transactions
- **Zero Configuration**: String IDs preserved, deferred schema operations
- **Express API**: Direct CRUD without workflow overhead
- **Sync Express**: Blocking API suitable for Ruby scripts, CLI tools, and Rack applications

## Install

```bash
gem install kailash-dataflow
```

Or in Gemfile:

```ruby
gem "kailash-dataflow"
```

## Quick Start

### Express API (Recommended for Simple CRUD)

```ruby
require "kailash/dataflow"

db = Kailash::DataFlow.new do |config|
  config.database_url = ENV["DATABASE_URL"]
  config.auto_migrate = true
end

# Define model via DSL
db.model "User" do |m|
  m.string :name
  m.string :email
  m.boolean :active, default: true
end

db.initialize!

# Express API -- direct CRUD, no workflow overhead
user = db.express.create("User", name: "Alice", email: "alice@example.com")
found = db.express.read("User", user["id"])
users = db.express.list("User", filter: { active: true })
count = db.express.count("User")
db.express.update("User", user["id"], name: "Bob")
db.express.delete("User", user["id"])
```

### Workflow API (For Multi-Step Operations)

Use WorkflowBuilder only when you need multiple nodes with data flow between them.

```ruby
require "kailash"
require "kailash/dataflow"

registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new

# Multi-node workflow with connections
builder.add_node("User_Create", "create_user", {
  "data" => { "name" => "John", "email" => "john@example.com" }
})

# Execute with block form (recommended for resource cleanup)
workflow = builder.build(registry)
Kailash::Runtime.open(registry) do |rt|
  result = rt.execute(workflow, {})
  user_id = result.results["create_user"]["result"]
end
workflow.close
```

## Generated Nodes (11 per model)

Each model definition generates:

1. `{Model}_Create` - Create single record
2. `{Model}_Read` - Read by ID
3. `{Model}_Update` - Update record
4. `{Model}_Delete` - Delete record
5. `{Model}_List` - List with filters
6. `{Model}_Upsert` - Insert or update (atomic)
7. `{Model}_Count` - Efficient COUNT(*) queries
8. `{Model}_BulkCreate` - Bulk insert
9. `{Model}_BulkUpdate` - Bulk update
10. `{Model}_BulkDelete` - Bulk delete
11. `{Model}_BulkUpsert` - Bulk upsert

## Model Definition DSL

```ruby
db.model "Product" do |m|
  m.string :name, null: false
  m.string :sku, unique: true
  m.decimal :price, precision: 10, scale: 2
  m.integer :quantity, default: 0
  m.boolean :active, default: true
  m.timestamp :created_at
  m.timestamp :updated_at
end
```

Supported field types: `string`, `integer`, `boolean`, `decimal`, `float`, `text`, `timestamp`, `date`, `json`.

## Critical Rules

- Always close DataFlow, Registry, and Workflow objects (use block form when possible)
- String IDs preserved (no UUID conversion)
- Deferred schema operations (safe for Rack/Puma startup)
- Multi-instance isolation (one DataFlow per database)
- Result access: `result.results["node_id"]["result"]`
- NEVER use raw SQL when DataFlow nodes exist
- NEVER use ActiveRecord/Sequel alongside DataFlow for the same tables
- Use keyword arguments for Express API: `db.express.create("User", name: "Alice")`

## Database Support Matrix

| Database   | Type | Nodes/Model | Driver         |
| ---------- | ---- | ----------- | -------------- |
| PostgreSQL | SQL  | 11          | sqlx (via FFI) |
| MySQL      | SQL  | 11          | sqlx (via FFI) |
| SQLite     | SQL  | 11          | sqlx (via FFI) |

The Ruby gem delegates all database operations to the Rust sqlx engine via native extensions. No Ruby database drivers are required.

## Integration Patterns

### With Nexus (Multi-Channel)

```ruby
require "kailash/dataflow"
require "kailash/nexus"

db = Kailash::DataFlow.new do |config|
  config.database_url = ENV["DATABASE_URL"]
end

db.model "User" do |m|
  m.string :name
  m.string :email
end

# Auto-generates API + CLI + MCP
nexus = Kailash::Nexus.new(db.workflows)
nexus.run(port: 8000)
```

### With Core SDK (Custom Workflows)

```ruby
require "kailash"
require "kailash/dataflow"

db = Kailash::DataFlow.new do |config|
  config.database_url = ENV["DATABASE_URL"]
end

# Use db-generated nodes in custom workflows
registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new
builder.add_node("User_Create", "user1", { "data" => { "name" => "Alice" } })
```

### With Rack Middleware

```ruby
# config.ru
require "kailash/dataflow"

db = Kailash::DataFlow.new do |config|
  config.database_url = ENV["DATABASE_URL"]
end
db.initialize!

# DataFlow is available to all Rack applications
use Kailash::DataFlow::Middleware, dataflow: db
run MyApp
```

## When to Use This Skill

Use DataFlow when you need to:

- Perform database operations in workflows
- Generate CRUD APIs automatically (with Nexus)
- Implement multi-tenant systems
- Work with existing databases
- Build database-first applications
- Handle bulk data operations

## Related Skills

- **[01-core-sdk](../01-core-sdk/SKILL.md)** - Core workflow patterns
- **[03-nexus](../03-nexus/SKILL.md)** - Multi-channel deployment
- **[04-kaizen](../04-kaizen/SKILL.md)** - AI agent integration
- **[05-kailash-mcp](../05-kailash-mcp/SKILL.md)** - MCP server integration

## Support

For DataFlow-specific questions, invoke:

- `dataflow-specialist` - DataFlow implementation and patterns
- `testing-specialist` - DataFlow testing strategies
- ``decide-framework` skill` - Choose between Core SDK and DataFlow
