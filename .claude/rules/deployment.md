---
paths:
  - "*.gemspec"
  - "Gemfile"
  - "lib/**/version.rb"
  - ".github/**"
  - "CHANGELOG.md"
---

# SDK Release Rules

## Scope

These rules apply to all SDK release operations and release-related files (`deploy/**`, `.github/workflows/**`, `*.gemspec`, `CHANGELOG.md`).

## MUST Rules

### 1. Full Test Suite Before Release

All releases MUST pass the full test suite across all supported Ruby versions before publishing.

```bash
# Run full test suite
bundle exec rspec

# Or via Rake if configured
bundle exec rake spec

# Run across Ruby version matrix (CI)
# Ruby 3.1, 3.2, 3.3, 3.4
```

**Enforced by**: deployment-specialist agent, CI pipeline
**Violation**: BLOCK release

### 2. RubyGems.org Validation Before Production Push

Major and minor releases MUST be validated before publishing to RubyGems.org.

```bash
# Build the gem
gem build kailash.gemspec

# Verify the gem contents
gem specification kailash-X.Y.Z.gem
tar tf kailash-X.Y.Z.gem | head -20

# Install locally and verify
gem install ./kailash-X.Y.Z.gem
ruby -e "require 'kailash'; puts Kailash::VERSION"
```

**Exception**: Patch releases (bug fixes only) may skip local validation with explicit human approval.

### 3. Version Consistency Across All Packages

All packages in the SDK MUST have consistent, compatible versions before release.

**Check**:

- `*.gemspec` version field (or dependency on `Kailash::VERSION`)
- `lib/kailash/version.rb` — `Kailash::VERSION` constant
- Cross-gem dependency version pins in `Gemfile` and gemspecs

```ruby
# lib/kailash/version.rb
module Kailash
  VERSION = "X.Y.Z"
end

# kailash.gemspec
Gem::Specification.new do |spec|
  spec.version = Kailash::VERSION
end
```

Both locations MUST match. The session-start hook checks this automatically. **A release with mismatched versions is BLOCKED.**

### 4. CHANGELOG.md Updated for Every Release

Every release MUST have a corresponding entry in `CHANGELOG.md` with:

- Version number and date
- Added, Changed, Fixed, Removed sections as applicable
- Breaking changes clearly marked

### 5. Security Review Before Publishing

Security review by **security-reviewer** is MANDATORY before any gem push.

**Check for**:

- No hardcoded secrets in gem source
- No sensitive data in gem contents
- Dependencies are pinned and audited (`bundle audit`)
- No known vulnerabilities in dependencies

**Enforced by**: agents.md Rule 2, validate-deployment.js hook
**Violation**: BLOCK release

### 6. Native Extension Build Verification

Because the Kailash Ruby SDK wraps the Rust SDK via native extensions, releases MUST verify native extension compilation on all target platforms.

```bash
# Verify native extension builds
bundle exec rake compile

# Cross-platform verification (CI matrix)
# - x86_64-linux, aarch64-linux
# - x86_64-darwin, arm64-darwin
# - x64-mingw-ucrt (Windows)

# Verify the native extension loads
ruby -e "require 'kailash/native'; puts 'Native extension loaded'"
```

**Why**: A gem with broken native extensions is unusable. The Rust FFI layer must compile and link correctly on every supported platform.

### 7. Release Config Documented

Every SDK that publishes releases MUST have `deploy/deployment-config.md` at the project root. Run `/deploy` to create it via the onboarding process.

### 8. Research Before Executing

RubyGems tooling and CI patterns change frequently. MUST verify current syntax via web search or `--help` before running release commands. Do NOT rely on memorized commands that may be outdated.

## MUST NOT Rules

### 1. No Publishing Without CI Green

MUST NOT publish to RubyGems.org when CI is failing. All checks must pass first.

### 2. No Skipping Validation for Major/Minor Releases

MUST NOT skip local install validation for major (X.0.0) or minor (X.Y.0) releases. These carry higher risk of breaking changes.

### 3. No RubyGems Credentials in Source

MUST NOT commit RubyGems API keys or credentials to source control.

**Correct**:

- `~/.gem/credentials` (local, gitignored, `chmod 0600`)
- CI secrets (GitHub Actions secrets)
- Trusted publisher (OIDC — no tokens needed)

**Incorrect**:

```
# BLOCKED: credentials in repo
GEM_HOST_API_KEY=rubygems-... in .env
~/.gem/credentials committed to repo
API key in CI config files
```

**Enforced by**: validate-deployment.js hook, security-reviewer agent
**Violation**: BLOCK commit

### 4. No Publishing Without Version Bump

MUST NOT publish a release without bumping the version number. RubyGems.org does not allow overwriting existing versions.

### 5. No Uncommitted Changes During Release

MUST NOT publish from a dirty working tree. All changes must be committed and pushed before building release artifacts.

## Release Checklist

Before any SDK release:

- [ ] All tests pass (full matrix: Ruby versions x OS x native extension platforms)
- [ ] Security review completed (`bundle audit`, security-reviewer agent)
- [ ] CHANGELOG.md updated with release entry
- [ ] Version bumped consistently (`lib/kailash/version.rb` + gemspec)
- [ ] RuboCop and formatting checks pass
- [ ] Native extensions compile on all target platforms
- [ ] Local gem install verification passed
- [ ] `gem push` to RubyGems.org successful
- [ ] Clean install verification passed (`gem install kailash -v X.Y.Z`)
- [ ] GitHub Release created with release notes
- [ ] Documentation deployed and verified
- [ ] Release logged in `deploy/deployments/`

## Exceptions

Release rule exceptions require:

1. Explicit human approval
2. Documentation in deployment-config.md
3. Justification (e.g., critical hotfix skipping full validation)
