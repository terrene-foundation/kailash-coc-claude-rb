---
paths:
  - "**/*.rb"
  - "**/config/**"
  - "**/app/**"
---

# Ruby Observability Rules

This variant replaces the global `observability.md` for the kailash-coc-claude-rb USE template. The same mandatory log points and log-triage gate apply; this file provides Ruby-specific bindings.

## Framework Logger — SemanticLogger or Rails.logger

MUST use `SemanticLogger` (recommended) or `Rails.logger` (if already in a Rails app). Never `puts`, `p`, `pp`, or `$stderr.puts` in production code.

```ruby
# DO — SemanticLogger (preferred for structured JSON output)
require 'semantic_logger'
SemanticLogger.default_level = :info
SemanticLogger.add_appender(io: $stdout, formatter: :json)

logger = SemanticLogger['UserService']
logger.info("user.create.start", user_id: uid, request_id: req_id)

# DO — Rails.logger with tagged logging
Rails.logger.tagged(request_id: request.request_id) do
  Rails.logger.info("user.create.start")
end

# DO NOT
puts "Creating user #{uid}"  # unstructured, no level, no fields, lost on restart
```

**Why:** `puts` writes unstructured strings to stdout with no level, no tagging, and no routing. It cannot be filtered, aggregated, or shipped to a log aggregator.

## Correlation ID — ActionDispatch::RequestId or Manual

Every log line MUST carry a correlation ID (`request_id`, `trace_id`, `run_id`) for the scope of the execution.

```ruby
# DO — Rails (ActionDispatch::RequestId middleware adds X-Request-Id automatically)
# config/application.rb
config.middleware.use ActionDispatch::RequestId

# In controllers/services:
Rails.logger.tagged(request_id: request.request_id) do
  Rails.logger.info("payment.charge.start", amount: amount)
end

# DO — Non-Rails (manual thread-local)
Thread.current[:request_id] = SecureRandom.uuid
logger.tagged(request_id: Thread.current[:request_id]) do
  logger.info("step.start")
end

# DO NOT
logger.info("step.start")  # no request_id — cannot trace across service calls
```

**Why:** Without correlation IDs, multi-step requests interleave in logs and become impossible to trace during incident response.

## Mandatory Log Points — Ruby Examples

### 1. Endpoints (Rails controllers, Sinatra routes, Grape endpoints)

```ruby
# DO
class UsersController < ApplicationController
  def create
    logger.info("users.create.start", params: filtered_params)
    t0 = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    user = UserService.create!(user_params)
    logger.info("users.create.ok", user_id: user.id,
                latency_ms: ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - t0) * 1000).round)
    render json: user, status: :created
  rescue StandardError => e
    logger.error("users.create.error", error: e.message, backtrace: e.backtrace.first(5))
    raise
  end
end

# DO NOT
class UsersController < ApplicationController
  def create
    render json: UserService.create!(user_params), status: :created  # zero observability
  end
end
```

### 2. Integration Points (HTTP clients, DB, queues, file IO)

```ruby
# DO
logger.info("stripe.charge.start", customer_id: cid, amount_cents: amount)
response = Stripe::Charge.create(customer: cid, amount: amount)
logger.info("stripe.charge.ok", charge_id: response.id)

# DO NOT
response = Stripe::Charge.create(customer: cid, amount: amount)  # silent integration
```

### 3. Data Calls — mode field required

```ruby
# DO — real
logger.info("user.fetch", user_id: uid, source: "postgres", mode: "real")

# DO — fake (presence in prod logs is a violation)
logger.warn("user.fetch", user_id: uid, source: "fixture", mode: "fake")
```

### 4. State Transitions, Auth Events, Config Loads

`info` level, once per transition. Auth events (sign-in, sign-out, role change) MUST log subject, action, and outcome. Config loads MUST log which file/env was used.

```ruby
# DO
logger.info("auth.sign_in.ok", user_id: uid, method: "password")
logger.info("config.loaded", source: "config/application.yml", env: Rails.env)

# DO NOT
# (no log at all — auth state changes invisible to ops)
```

**Why:** Auth failures and config drifts are the second-most-common production incidents and are nearly impossible to diagnose without their own log lines.

## Log Levels

| Level   | When                                                          |
| ------- | ------------------------------------------------------------- |
| `error` | Operation failed and caller will see the failure              |
| `warn`  | Succeeded but used a fallback, retry, or degraded path        |
| `info`  | Normal state transition the operator should see in production |
| `debug` | Step-by-step detail for investigation; off by default         |

## MUST NOT

- **No `rescue => e; end` (bare rescue with no logging or re-raise)** — same as `except: pass`, BLOCKED per `zero-tolerance.md` Rule 3.

**Why:** Silent rescue swallows the failure and continues with corrupt or partial state, hiding bugs until they cascade into data corruption.

- **No log-and-continue on rescued exceptions without action.** If you rescue an exception, log it AND either retry, fall back, or re-raise. `rescue => e; logger.error(e); end` (swallow), `rescue => e; logger.error(e); nil` (return nil), and `rescue => e; logger.error(e); next` (skip iteration in a loop) are all BLOCKED. **Exception:** cleanup/finalizer paths where failure is expected — log at `warn` and continue.

**Why:** Logging an exception and discarding it produces a paper trail of failures that nothing acts on, creating noisy logs and broken behavior simultaneously.

- **No `puts`/`p`/`pp`/`$stderr.puts` in production code** — use the structured logger.

**Why:** `puts` writes unstructured strings with no level, no field tagging, and no routing. It cannot be filtered or shipped to a log aggregator.

- **No unstructured string interpolation in log messages** — pass fields as keyword arguments.

```ruby
# DO
logger.info("user.created", user_id: uid, plan: plan)

# DO NOT
logger.info("User created: #{uid} on #{plan}")  # cannot filter or aggregate
```

**Why:** Interpolated log messages cannot be queried by field, defeating the entire purpose of structured logging.

- **No log-spam in hot loops.** Per-iteration `info` inside a tight loop floods aggregators and increases bills. Use sample-rate logging or aggregate to one summary line per N items.

**Why:** A 1M-row processing loop emitting one info per row produces 1M log lines per run, which costs money in the aggregator and crowds out the real signal.

- **No silent log-level downgrades.** MUST NOT change a `warn`/`error` to `info` to "clean up" CI output. Fix the root cause or document the suppression in code.

**Why:** Downgrading log levels to silence noise is a Zero-Tolerance Rule 1 violation in disguise — the failure is still happening, the operator just stops seeing it.

## Log Triage Gate — Read Before Reporting Done

Before any of `/implement`, `/redteam`, `/deploy`, `/wrapup` reports complete, MUST scan for `warn`+ entries and acknowledge each one.

**Concrete scan commands** (run all that apply):

```bash
# RSpec output (most recent run)
bundle exec rspec 2>&1 | grep -iE 'warn|error|deprecat|fail' | sort -u

# Project log directory (modified in the last 2 hours — proxy for "this session")
find . -name "*.log" -mmin -120 -exec grep -HnE 'WARN|ERROR' {} +

# Rails server output (development.log / test.log)
grep -E 'WARN|ERROR|DEPRECATION' log/*.log | sort -u

# Bundle dependency check
bundle check 2>&1
```

**Disposition protocol** (per unique entry, not per occurrence):

1. Group identical `warn`+ entries by source file + message pattern. Count occurrences but state disposition once.
2. For each unique entry, state one of:
   - **Fixed** — root cause addressed in this session, with the commit SHA
   - **Deferred** — explicit reason + tracked todo + human acknowledgment
   - **Upstream** — third-party deprecation requiring a gem update; opened upstream issue or pinned version with reason
   - **False positive** — explanation of why it does not apply
3. Unacknowledged `warn`+ entries BLOCK the gate.

**Exception:** Cleanup paths where failure is expected may emit `warn` entries that are pre-acknowledged in code via a comment marker.

**Why:** Logs that no one reads create the illusion of observability while letting underlying problems fester. Without dedup, a Rails test suite producing 200 deprecation warnings becomes 200 disposition lines and the agent rationally skips the gate; with dedup, it becomes 5–10 unique entries that are tractable.
