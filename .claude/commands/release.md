# /release - Ruby Gem Release

## Pre-Release Checklist

1. `bundle exec rspec` — all specs pass
2. Version consistent: `kailash.gemspec` version matches workspace
3. Security review completed
4. Source protection: no `.rs` or `Cargo.toml` in gem

## Release Process

```bash
# 1. Bump version in kailash.gemspec
# 2. Commit and tag
git tag vX.Y.Z
git push --tags

# 3. CI builds platform gems automatically:
#    - arm64-darwin (macOS Apple Silicon)
#    - x86_64-linux
#    - aarch64-linux

# 4. CI publishes to RubyGems via RUBYGEMS_API_KEY secret
```

## Manual Gem Build (local testing)

```bash
cd bindings/kailash-ruby
cargo build -p kailash-ruby --release
cp target/release/libkailash.dylib lib/kailash/kailash.bundle
codesign -fs - lib/kailash/kailash.bundle  # macOS only
bundle exec rspec
```

## Verify Release

```bash
gem install kailash
ruby -e "require 'kailash'; puts Kailash::Registry.new.list_types.length"
```

## Source Protection

Platform gems MUST NOT contain `.rs` files or `Cargo.toml`. The CI `source-protection-audit` step verifies this before every publish.
