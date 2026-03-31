---
paths:
  - "spec/e2e/**"
  - "test/e2e/**"
  - "**/*e2e*"
---

# E2E God-Mode Testing Rules

## Scope

These rules apply to ALL end-to-end testing, validation, and browser-based test runs against applications built with the Kailash Ruby SDK.

## ABSOLUTE RULES

### 1. God-Mode: Create ALL Missing Records

When running E2E tests and a required record is missing (404, 403, empty response):

**MUST**: Create the missing record immediately using the appropriate API or direct database access.
**MUST NOT**: Skip the test, document it as a "gap", or report it as "expected behavior".

**Pattern:**

```
1. Attempt operation (e.g., Net::HTTP request)
2. If 404/403/missing -> identify what's missing
3. Create the missing record via API (use admin credentials)
4. Retry the original operation
5. NEVER skip or move on
```

**Ruby Example:**

```ruby
require "net/http"
require "json"

uri = URI("#{base_url}/api/users/#{user_id}")
resp = Net::HTTP.get_response(uri)

if resp.code == "404"
  # God-mode: create the missing user
  post_uri = URI("#{base_url}/api/users")
  Net::HTTP.post(post_uri, { name: "test_user", role: "admin" }.to_json,
                 "Content-Type" => "application/json")
  # Retry original operation
  resp = Net::HTTP.get_response(uri)
  expect(resp.code).to eq("200")
end
```

**RSpec with Capybara Example:**

```ruby
RSpec.describe "User management", type: :e2e do
  it "displays user profile" do
    # Attempt to visit user profile
    visit "/users/#{user_id}"

    if page.status_code == 404
      # God-mode: create the missing user via API
      create_user_via_api(name: "test_user", role: "admin")
      visit "/users/#{user_id}"
    end

    expect(page).to have_content("test_user")
  end
end
```

### 2. Adapt to Data Changes

Test data WILL change between runs. User emails, IDs, names may all differ.

**MUST**: Query the API to discover actual records before testing.
**MUST NOT**: Hardcode user emails, IDs, or other test data.

**Pattern:**

```
1. Before testing, query the list endpoint to get actual records
2. Find the record matching the role/type, not a hardcoded name
3. Use the ACTUAL values from the query
4. If no matching record exists, CREATE one
```

**Ruby Example:**

```ruby
# RIGHT -- discover actual records
uri = URI("#{base_url}/api/users")
resp = Net::HTTP.get_response(uri)
users = JSON.parse(resp.body)
admin = users.find { |u| u["role"] == "admin" }

# Create if missing
unless admin
  post_uri = URI("#{base_url}/api/users")
  Net::HTTP.post(post_uri, { name: "test_admin", role: "admin" }.to_json,
                 "Content-Type" => "application/json")
  resp = Net::HTTP.get_response(uri)
  admin = JSON.parse(resp.body).find { |u| u["role"] == "admin" }
end

# WRONG -- hardcoded
admin_email = "admin@example.com"  # Will break when data changes!
```

### 3. Implement Missing Endpoints

If an API endpoint doesn't exist and testing needs it:

**MUST**: Implement the endpoint immediately using the Kailash SDK (Nexus handler, Sinatra route, Rails controller, or equivalent).
**MUST NOT**: Document it as a "limitation" and move on.

### 4. Follow Up on Failures

When an operation fails gracefully (error message displayed, no crash):

**MUST**: Investigate the root cause and implement a fix.
**MUST NOT**: Report "graceful failure" and move to next test.

**Pattern:**

```
1. Operation fails with error
2. Check service logs for root cause (Rails.logger, stdout, log/*.log)
3. If missing API key -> verify .env is loaded (dotenv gem)
4. If missing record -> create it (Rule 1)
5. If missing endpoint -> implement it (Rule 3)
6. Retry the operation
7. Only move on after SUCCESS or explicit user instruction to skip
```

### 5. Assume Correct Role

During multi-persona testing, assume the role needed for each operation.

**Pattern:**

```
1. Need admin actions -> authenticate as admin/owner (JWT or API key)
2. Need to test restricted views -> authenticate as restricted user
3. Need to test RBAC -> try each role and verify access/denial
```

**Ruby Example:**

```ruby
def authenticate_as(role)
  uri = URI("#{base_url}/api/auth/login")
  credentials = test_credentials_for(role)
  resp = Net::HTTP.post(uri, credentials.to_json, "Content-Type" => "application/json")
  token = JSON.parse(resp.body)["token"]
  { "Authorization" => "Bearer #{token}" }
end

admin_headers = authenticate_as(:admin)
user_headers = authenticate_as(:user)
```

## Pre-E2E Checklist

Before starting ANY E2E test run:

- [ ] Application server running (`bundle exec rails s`, `bundle exec rackup`, or equivalent)
- [ ] Frontend dev server running (if applicable)
- [ ] `.env` loaded and verified (check `MODEL` and `API_KEY` vars; `dotenv` gem or `source .env`)
- [ ] Database migrations applied (`bundle exec rake db:migrate`)
- [ ] Required users exist (query API, create if missing)
- [ ] Required resources exist (query API, create if missing)
- [ ] Access records exist (query API, create if missing)

## Exceptions

NO EXCEPTIONS for rules 1-4. If you cannot create a record, escalate to the user immediately.
Rule 5 exception: User explicitly says "only test as X role".
