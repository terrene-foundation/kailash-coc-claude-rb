---
paths:
  - "**/*.rb"
  - "**/*.gemspec"
  - "**/Gemfile"
---

# Zero-Tolerance Enforcement Rules (Ruby)

## Scope

These rules apply to ALL Ruby code in this repository.

## ABSOLUTE RULE 1: Pre-Existing Failures MUST Be Resolved

When tests or analysis reveals a pre-existing failure: FIX IT.

**Required response:**
1. Diagnose the root cause
2. Implement the fix
3. Write a regression spec
4. Verify with `bundle exec rspec`

## ABSOLUTE RULE 2: No Stubs or Placeholders

Production code MUST NOT contain:

```ruby
# BLOCKED:
raise NotImplementedError
# TODO: implement this
def method_name; end  # empty body
return nil  # placeholder
```

## ABSOLUTE RULE 3: No Silent Error Swallowing

```ruby
# BLOCKED:
rescue => e
  # silently ignored
end

rescue StandardError
  nil
end

# ACCEPTABLE:
rescue Errno::ENOENT
  logger.warn("File not found, using default")
  default_value
end
```

## ABSOLUTE RULE 4: No SDK Workarounds

If you encounter a bug in the Kailash gem, report it upstream. Do NOT work around it with monkey-patches or reimplementations.
