---
paths:
  - "lib/**/trust/**"
---

# Trust-Plane Security Rules

## Scope

These rules apply when editing trust-plane code in the Ruby SDK.

These rules supplement `.claude/rules/security.md`. Both apply to trust-plane files.
Violations during code review by intermediate-reviewer are BLOCK-level findings.

## MUST Rules

### 1. No Bare `File.open` or `File.read` for Record Files

```ruby
# DO:
data = Kailash::Trust::Locking.safe_read_json(path)

# DO NOT:
data = JSON.parse(File.read(path))       # Follows symlinks -- attacker redirects to arbitrary file
File.open(path) { |f| JSON.parse(f.read) }  # No symlink protection, no fd safety
```

**Why**: `safe_read_json` uses `File.realpath` and `O_NOFOLLOW` (where supported) to prevent symlink attacks. Bare `File.read` and `File.open` bypass all protections.

### 2. `validate_id` on Every Externally-Sourced Record ID

```ruby
# DO:
Kailash::Trust::Locking.validate_id(record_id)  # Raises ArgumentError on "../", "/", null bytes, etc.
path = File.join(store_dir, "#{record_id}.json")

# DO NOT:
path = File.join(store_dir, "#{user_input}.json")  # Path traversal: "../../../etc/passwd"
conn.exec("SELECT * FROM records WHERE id = '#{record_id}'")  # SQL injection
```

**Why**: The regex `/\A[a-zA-Z0-9_-]+\z/` prevents directory traversal and SQL injection via IDs. Every method that accepts a record ID or query parameter MUST validate before use.

### 3. `Float#finite?` on All Numeric Constraint Fields

```ruby
# DO (in initialize or from_hash):
raise ArgumentError, "max_cost must be finite" if max_cost && !max_cost.to_f.finite?

# DO NOT:
raise ArgumentError, "negative" if max_cost && max_cost < 0  # NaN passes, Inf passes
```

**Why**: `Float::NAN` and `Float::INFINITY` bypass numeric comparisons (`Float::NAN < 0` is `false`, `Float::INFINITY < 0` is `false`). Constraints set to `NaN` make all checks pass silently.

### 4. Bounded Collections (max size 10_000)

```ruby
# DO:
class BoundedLog
  MAX_SIZE = 10_000

  def initialize
    @entries = []
  end

  def push(entry)
    @entries.shift(@entries.size - (MAX_SIZE * 9 / 10)) if @entries.size >= MAX_SIZE
    @entries << entry
  end
end

# DO NOT:
@call_log = []  # Grows without bound -> OOM
```

**Why**: Unbounded collections in long-running processes lead to memory exhaustion. Trim oldest 10% when at capacity.

### 5. Parameterized SQL for All Database Queries

```ruby
# DO (PG gem):
conn.exec_params("SELECT * FROM decisions WHERE id = $1", [record_id])
conn.exec_params("INSERT INTO decisions (id, data) VALUES ($1, $2)", [id, data])

# DO (Sequel):
DB[:decisions].where(id: record_id).first
DB[:decisions].insert(id: id, data: data)

# DO NOT:
conn.exec("SELECT * FROM decisions WHERE id = '#{record_id}'")
conn.exec("INSERT INTO decisions VALUES (#{id}, #{data})")
```

**Why**: String interpolation into SQL enables injection. Even validated IDs should use parameterized queries as defense-in-depth.

### 6. Database File Permissions

```ruby
# DO (on POSIX):
FileUtils.touch(db_path)
File.chmod(0o600, db_path)  # Owner read/write only

# DO NOT:
FileUtils.touch(db_path)  # Default permissions may be world-readable
```

**Why**: Database files (`.db`, `-wal`, `-shm`) contain all trust records. Default permissions may expose them to other users on shared systems.

### 7. All Record Writes Through `atomic_write`

```ruby
# DO:
Kailash::Trust::Locking.atomic_write(path, JSON.generate(record.to_h))

# DO NOT:
File.write(path, JSON.generate(record))  # Partial write on crash = corrupted record
File.open(path, "w") { |f| f.write(JSON.generate(record)) }
```

**Why**: `atomic_write` uses temp file + `fsync` + `File.rename` for crash safety. It also checks `File.realpath` to prevent symlink attacks during writes.

## MUST NOT Rules

### 1. MUST NOT Use `==` to Compare HMAC Digests

```ruby
# DO:
require "openssl"
unless OpenSSL.secure_compare(stored_hash, computed_hash)
  raise Kailash::Trust::TamperDetectedError, "..."
end

# Or using Rack::Utils if available:
unless Rack::Utils.secure_compare(stored_hash, computed_hash)
  raise Kailash::Trust::TamperDetectedError, "..."
end

# DO NOT:
if stored_hash != computed_hash  # Timing side-channel for byte-by-byte forgery
  raise Kailash::Trust::TamperDetectedError, "..."
end
```

**Why**: String equality (`==`) leaks timing information. An attacker can measure comparison time to determine how many bytes match. Use `OpenSSL.secure_compare` or `Rack::Utils.secure_compare` for constant-time comparison.

### 2. MUST NOT Downgrade Trust State

```ruby
# CORRECT: Monotonic escalation only
# :auto_approved -> :flagged -> :held -> :blocked (only forward)

# FORBIDDEN:
verdict = :auto_approved if some_condition  # Downgrading from :held is forbidden
```

**Why**: Trust state can only escalate, never relax. A HELD action cannot become AUTO_APPROVED -- it must be explicitly resolved through the hold workflow.

### 3. MUST NOT Write Records Without `atomic_write`

Any filesystem record write that bypasses `atomic_write` is a security defect. See MUST Rule 7.

### 4. MUST NOT Leave Private Key Material in Memory

```ruby
# DO:
key_mgr.register_key(key_id, private_key)
private_key.replace("\0" * private_key.bytesize)  # Overwrite contents
private_key = nil

# On revocation:
@keys[key_id] = ""  # Clear material, keep tombstone

# DO NOT:
key_mgr.register_key(key_id, private_key)
# private_key persists in scope -- visible to memory dumps
```

**Why**: Private key material in memory is vulnerable to debugger inspection and memory dumps. Ruby strings are mutable, so overwrite with null bytes before discarding.

### 5. MUST NOT Construct Policy Objects as Mutable After Validation

```ruby
# DO:
class MultiSigPolicy
  attr_reader :required_signatures

  def initialize(required_signatures:)
    raise ArgumentError, "must be positive" unless required_signatures.positive?
    @required_signatures = required_signatures
    freeze
  end
end

# DO NOT:
class MultiSigPolicy
  attr_accessor :required_signatures  # Mutable -- fields can be changed after validation
end
```

**Why**: Without `freeze`, an attacker with object reference can bypass validation by directly setting fields. This applies to ALL five constraint classes (`OperationalConstraints`, `DataAccessConstraints`, `FinancialConstraints`, `TemporalConstraints`, `CommunicationConstraints`) -- all must call `freeze` after initialization.

### 6. MUST NOT Pass Unvalidated Cost Values to Budget Checks

```ruby
# DO:
action_cost = ctx.fetch("cost", 0.0).to_f
unless action_cost.finite? && action_cost >= 0
  return :blocked  # Fail-closed on NaN/Inf/negative
end

# DO NOT:
action_cost = ctx.fetch("cost", 0.0).to_f
return :blocked if action_cost > limit  # NaN > limit is always false -- budget bypassed!
```

**Why**: `Float::NAN` bypasses all numeric comparisons (`Float::NAN > x` is always `false`). If `NaN` enters `session_cost` via `+=`, it permanently poisons the accumulator -- all future budget checks pass. Every path that accepts a cost value MUST validate with `finite?`.

### 7. MUST NOT Rescue Bare `KeyError` Where `RecordNotFoundError` Is Intended

```ruby
# DO:
begin
  delegate = store.get_delegate(did)
rescue Kailash::Trust::RecordNotFoundError
  # Already gone
end

# DO NOT:
begin
  delegate = store.get_delegate(did)
rescue KeyError  # Too broad -- catches unrelated hash lookup failures
  # ...
end
```

**Why**: `RecordNotFoundError` may inherit from both `Kailash::Trust::StoreError` and `KeyError`. Bare `rescue KeyError` catches store errors and potentially swallows unrelated hash lookup failures or corrupted-record exceptions.

### 8. MUST: Use `normalize_resource_path` for All Constraint Pattern Storage and Comparison

All constraint patterns and resource paths MUST be normalized via `Kailash::Trust.normalize_resource_path` before storage or comparison. Direct use of `File.expand_path`, `Pathname#cleanpath`, or manual path manipulation for constraint patterns is FORBIDDEN.

```ruby
# DO:
norm = Kailash::Trust.normalize_resource_path(user_path)

# DO NOT:
norm = File.expand_path(user_path)      # Platform-dependent, resolves against cwd
norm = Pathname.new(user_path).cleanpath.to_s  # Doesn't normalize separators consistently
```

**Why**: `File.expand_path` resolves against the current working directory and behaves differently across platforms. `Pathname#cleanpath` doesn't collapse double slashes consistently. `normalize_resource_path` provides consistent forward-slash normalization on all platforms.

## Cross-References

- `.claude/rules/security.md` -- Global security rules (secrets, injection, input validation)
- `.claude/rules/eatp.md` -- EATP SDK conventions (error hierarchy, cryptography)
