---
paths:
  - "lib/**/trust/**"
---

# Trust-Plane Security Rules

### 1. No Bare `File.open` or `File.read` for Record Files

```ruby
# DO:
data = Kailash::Trust::Locking.safe_read_json(path)

# DO NOT:
data = JSON.parse(File.read(path))  # Follows symlinks
```

**Why:** Bare `File.read` follows symlinks, allowing an attacker to redirect trust-plane reads to arbitrary files outside the store directory.

### 2. `validate_id` on Every Externally-Sourced Record ID

```ruby
# DO:
Kailash::Trust::Locking.validate_id(record_id)
path = File.join(store_dir, "#{record_id}.json")

# DO NOT:
path = File.join(store_dir, "#{user_input}.json")  # Path traversal
```

**Why:** Unvalidated record IDs allow `../../../etc/passwd` traversal to read or overwrite files outside the trust store.

### 3. `Float#finite?` on All Numeric Constraint Fields

```ruby
# DO:
raise ArgumentError, "max_cost must be finite" if max_cost && !max_cost.to_f.finite?

# DO NOT:
raise ArgumentError, "negative" if max_cost && max_cost < 0  # NaN passes
```

**Why:** `Float::NAN < 0` evaluates to `false` in Ruby, so NaN silently passes range checks and corrupts constraint budgets.

### 4. Bounded Collections (max size 10_000)

```ruby
# DO:
@entries.shift(@entries.size - (MAX_SIZE * 9 / 10)) if @entries.size >= MAX_SIZE
@entries << entry

# DO NOT:
@call_log = []  # Grows without bound -> OOM
```

**Why:** Ruby's GC cannot reclaim live-referenced arrays, so unbounded collections cause OOM kills in long-running trust-plane processes.

### 5. Parameterized SQL for All Database Queries

```ruby
# DO:
conn.exec_params("SELECT * FROM decisions WHERE id = $1", [record_id])
DB[:decisions].where(id: record_id).first

# DO NOT:
conn.exec("SELECT * FROM decisions WHERE id = '#{record_id}'")
```

**Why:** Ruby string interpolation in SQL turns user input into executable statements, enabling full database compromise via injection.

### 6. Database File Permissions

```ruby
# DO:
FileUtils.touch(db_path)
File.chmod(0o600, db_path)  # Owner read/write only
```

**Why:** World-readable SQLite trust databases expose decision records, HMAC keys, and governance state to any local process.

### 7. All Record Writes Through `atomic_write`

```ruby
# DO:
Kailash::Trust::Locking.atomic_write(path, JSON.generate(record.to_h))

# DO NOT:
File.write(path, JSON.generate(record))  # Partial write on crash = corrupted record
```

**Why:** A crash during bare `File.write` truncates the file mid-write, permanently corrupting the trust record with no recovery path.

## MUST NOT

### 1. No `==` to Compare HMAC Digests

```ruby
# DO:
OpenSSL.secure_compare(stored_hash, computed_hash)

# DO NOT:
stored_hash != computed_hash  # Timing side-channel
```

**Why:** Ruby's `==` on strings short-circuits on the first differing byte, leaking digest contents one character at a time via timing attacks.

### 2. No Trust State Downgrade

Trust state only escalates: `:auto_approved → :flagged → :held → :blocked`. Never relax.

**Why:** Allowing downgrade means a compromised agent can reset its own trust state to `:auto_approved`, bypassing all governance controls.

### 3. No Private Key Material in Memory

```ruby
# DO:
key_mgr.register_key(key_id, private_key)
private_key.replace("\0" * private_key.bytesize)
private_key = nil
```

**Why:** Ruby strings are heap-allocated and GC-managed; without explicit zeroing, private keys persist in memory and appear in core dumps or heap inspections.

### 4. Frozen Constraint/Policy Objects

All constraint and policy classes MUST call `freeze` after initialization. Use `attr_reader`, not `attr_accessor`.

**Why:** Unfrozen policy objects can be mutated after creation, allowing runtime tampering with governance constraints that should be immutable once set.

### 5. No Unvalidated Cost Values

```ruby
# DO:
action_cost = ctx.fetch("cost", 0.0).to_f
return :blocked unless action_cost.finite? && action_cost >= 0

# DO NOT:
return :blocked if action_cost > limit  # NaN > limit is always false
```

**Why:** `Float::NAN > limit` returns `false` in Ruby, so NaN costs silently pass budget checks and allow unlimited spending.

### 6. No Bare `KeyError` Where `RecordNotFoundError` Is Intended

Use `Kailash::Trust::RecordNotFoundError`, not bare `rescue KeyError`.

**Why:** Rescuing `KeyError` catches unrelated Hash/Array key misses, masking real programming errors as "record not found."

### 7. Use `normalize_resource_path` for Constraint Patterns

```ruby
# DO:
norm = Kailash::Trust.normalize_resource_path(user_path)

# DO NOT:
norm = File.expand_path(user_path)  # Platform-dependent
```

**Why:** `File.expand_path` resolves against the OS filesystem, producing platform-dependent paths that break constraint matching across Linux/macOS/Windows.
