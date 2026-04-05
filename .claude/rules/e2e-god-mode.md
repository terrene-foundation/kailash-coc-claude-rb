---
paths:
  - "spec/e2e/**"
  - "test/e2e/**"
  - "**/*e2e*"
---

# E2E God-Mode Testing Rules

### 1. Create ALL Missing Records

When a required record is missing (404, 403, empty response): create it immediately via API or direct DB. MUST NOT skip, document as "gap", or report as "expected behavior."

**Why:** Skipping missing records produces hollow test runs that pass on paper but never exercise actual feature paths, hiding real bugs behind green specs.

```ruby
uri = URI("#{base_url}/api/users/#{user_id}")
resp = Net::HTTP.get_response(uri)

if resp.code == "404"
  # God-mode: create the missing user
  Net::HTTP.post(URI("#{base_url}/api/users"),
    { name: "test_user", role: "admin" }.to_json,
    "Content-Type" => "application/json")
  resp = Net::HTTP.get_response(uri)
  expect(resp.code).to eq("200")
end
```

### 2. Adapt to Data Changes

Test data changes between runs. Query the API to discover actual records before testing. MUST NOT hardcode user emails, IDs, or other test data.

**Why:** Hardcoded IDs cause intermittent failures whenever the database is reset or `rake db:seed` runs with different data, making specs appear flaky when they are actually brittle.

### 3. Implement Missing Endpoints

If an API endpoint doesn't exist and testing needs it: implement it immediately. MUST NOT document as "limitation."

**Why:** Documenting a missing endpoint as a "limitation" halts all dependent test coverage and defers the gap to a future session that may never come.

### 4. Follow Up on Failures

When an operation fails gracefully (error displayed, no crash): investigate root cause and fix. MUST NOT report "graceful failure" and move on.

**Why:** "Graceful failure" often masks real bugs behind Rails error-handling middleware -- the user sees a polished error page while the underlying feature remains broken.

### 5. Assume Correct Role

During multi-persona testing, log in as the role needed for each operation.

**Why:** Testing admin-only features as a superuser misses permission-denied paths, leaving authorization bugs undetected until a real restricted user hits them.

## Pre-E2E Checklist

**Why:** Missing any checklist item causes specs to fail for environmental reasons rather than code bugs, wasting an entire test cycle on setup issues.

- Application server running (`bundle exec rails s` or equivalent)
- Frontend dev server running (if applicable)
- .env loaded and verified
- Database migrations applied (`bundle exec rake db:migrate`)
- Required users, resources, and access records exist (query API, create if missing)
