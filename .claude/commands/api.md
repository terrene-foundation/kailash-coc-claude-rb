# /api - Nexus Quick Reference (Ruby)

## Multi-Channel Deployment

Nexus deploys workflows as API + CLI + MCP simultaneously. Ruby applications consume the API channel.

## HTTP Client Pattern

```ruby
require "net/http"
require "json"

# Nexus API endpoint
uri = URI("http://localhost:3000/api/v1/workflows/execute")
http = Net::HTTP.new(uri.host, uri.port)

request = Net::HTTP::Post.new(uri)
request["Content-Type"] = "application/json"
request.body = JSON.generate({
  workflow: "data-pipeline",
  parameters: { "input" => "value" }
})

response = http.request(request)
result = JSON.parse(response.body)
puts result["run_id"]
```

## With Faraday

```ruby
require "faraday"

conn = Faraday.new(url: "http://localhost:3000") do |f|
  f.request :json
  f.response :json
end

response = conn.post("/api/v1/workflows/execute", {
  workflow: "data-pipeline",
  parameters: { input: "value" }
})

puts response.body["run_id"]
```
