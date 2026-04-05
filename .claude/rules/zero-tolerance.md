---
paths:
  - "**/*.rb"
  - "**/*.gemspec"
  - "**/Gemfile"
---

# Zero-Tolerance Rules (Ruby)

## Scope

ALL sessions, ALL agents, ALL code, ALL phases. ABSOLUTE and NON-NEGOTIABLE.

## Rule 1: Pre-Existing Failures MUST Be Fixed

If you found it, you own it. Period.

**Why:** Deferring a known failure means it compounds silently across sessions until it blocks a critical path, at which point the original context is lost.

1. Diagnose root cause
2. Implement the fix
3. Write a regression spec
4. Verify with `bundle exec rspec`

**Exception:** User explicitly says "skip this issue."

## Rule 2: No Stubs or Placeholders

**Why:** Ruby silently returns `nil` from empty methods, so a stub appears to work until a caller chains on the return value and gets a `NoMethodError` on `nil`.

```ruby
# BLOCKED:
raise NotImplementedError
# TODO: implement this
def method_name; end  # empty body
return nil  # placeholder
```

## Rule 3: No Silent Error Swallowing

**Why:** Ruby's `rescue => e` catches all `StandardError` subclasses; silently ignoring them hides real failures (DB errors, auth failures) behind `nil` returns that propagate far from the source.

```ruby
# BLOCKED:
rescue => e
  # silently ignored
end

# ACCEPTABLE:
rescue Errno::ENOENT
  logger.warn("File not found, using default")
  default_value
end
```

## Rule 4: No SDK Workarounds

**Why:** Ruby monkey-patches are globally visible and persist for the process lifetime, so a workaround in one gem silently alters behavior for every other gem that uses the patched class.

If you encounter a bug in the Kailash gem, report it upstream. Do NOT work around it with monkey-patches or reimplementations.
