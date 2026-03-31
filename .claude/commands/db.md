# /db - DataFlow Quick Reference (Ruby)

## Setup

```ruby
require "kailash"

# DataFlow operations use the workflow engine
registry = Kailash::Registry.new
builder = Kailash::WorkflowBuilder.new
```

## CRUD Nodes

```ruby
# Create
builder.add_node("CreateNode", "create_user", {
  "table" => "users",
  "fields" => { "name" => "Alice", "email" => "alice@example.com" }
})

# Read
builder.add_node("ReadNode", "list_users", {
  "table" => "users",
  "filter" => { "active" => true }
})

# Update
builder.add_node("UpdateNode", "update_user", {
  "table" => "users",
  "filter" => { "id" => 1 },
  "fields" => { "name" => "Bob" }
})

# Delete
builder.add_node("DeleteNode", "remove_user", {
  "table" => "users",
  "filter" => { "id" => 1 }
})
```

## Key Rules

- Use **string keys**, not symbols
- Always verify writes with a read-back in tests
- DataFlow silently ignores unknown parameter names
