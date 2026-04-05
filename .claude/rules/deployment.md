---
paths:
  - "*.gemspec"
  - "Gemfile"
  - "lib/**/version.rb"
  - ".github/**"
  - "CHANGELOG.md"
---

# SDK Release Rules (Ruby Gem)

## Before Any Release

**Why:** Skipping any of these steps risks publishing a broken or insecure gem to RubyGems.org, where it becomes immediately available to all downstream users.

1. Full test suite passes across all supported Ruby versions (`bundle exec rspec`)
2. Security review by **security-reviewer** (mandatory, including `bundle audit`)
3. CHANGELOG.md updated (version, date, Added/Changed/Fixed/Removed, breaking changes marked)
4. Version bumped consistently (`lib/kailash/version.rb` + gemspec)
5. Native extensions compile on all target platforms
6. No uncommitted changes

## Gem Validation Before Production Push

Major/minor releases MUST validate before publishing to RubyGems.org.

**Why:** A gem that builds but fails to load (missing native extension, bad require path) is unusable, and RubyGems does not allow re-publishing the same version after yank.

```bash
gem build kailash.gemspec
gem specification kailash-X.Y.Z.gem
gem install ./kailash-X.Y.Z.gem
ruby -e "require 'kailash'; puts Kailash::VERSION"
```

**Exception**: Patch releases may skip local validation with explicit human approval.

## Native Extension Verification

```bash
bundle exec rake compile
ruby -e "require 'kailash/native'; puts 'Native extension loaded'"
# CI matrix: x86_64-linux, aarch64-linux, x86_64-darwin, arm64-darwin, x64-mingw-ucrt
```

**Why:** A gem with broken native extensions is unusable. The Rust FFI layer must compile and link on every supported platform.

## Publishing Rules

- No publishing when CI is failing
  **Why:** Publishing a gem with failing CI means known-broken code is served to every `bundle install` that matches the version constraint.
- No RubyGems credentials in source — use `~/.gem/credentials` (chmod 0600), CI secrets, or OIDC
  **Why:** Leaked RubyGems API keys allow anyone to publish malicious versions of the gem, compromising all downstream users.
- Research current syntax before running release commands
  **Why:** RubyGems CLI syntax changes between versions; using outdated flags can silently publish to the wrong index or with wrong metadata.

## Version Consistency

```ruby
# lib/kailash/version.rb
module Kailash
  VERSION = "X.Y.Z"
end

# kailash.gemspec
spec.version = Kailash::VERSION
```

Both locations MUST match. Release config: `deploy/deployment-config.md`.

**Why:** A version mismatch between `version.rb` and the gemspec causes `Kailash::VERSION` to report a different version than what Bundler resolves, breaking version-dependent logic downstream.
