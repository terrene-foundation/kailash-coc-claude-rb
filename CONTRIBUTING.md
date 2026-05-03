# Contributing to Kailash COC Claude (Ruby)

We welcome contributions to the Kailash COC template for Ruby projects. This document provides guidelines for contributing.

## About This Repository

This is a **COC (Cognitive Orchestration for Codegen) template** for Claude Code. It provides agents, skills, rules, commands, and hooks that Ruby projects inherit through the `.claude/` directory.

This project is maintained by the [Terrene Foundation](https://terrene.foundation) and licensed under Apache 2.0.

## Getting Started

1. Fork the repository on GitHub
2. Clone your fork locally
3. Create a new branch for your changes:
   ```bash
   git checkout -b feat/your-feature
   ```

## What to Contribute

- **Agents**: New or improved agent definitions in `.claude/agents/`
- **Skills**: Knowledge documents in `.claude/skills/` (numbered directories)
- **Rules**: Discipline rules in `.claude/rules/` (must include `priority:` + `scope:` frontmatter)
- **Commands**: Slash commands in `.claude/commands/`
- **Hooks**: Automation hooks in `.claude/hooks/`

## Guidelines

### Template Independence

This template is one of three sibling COC templates (`-py`, `-rs`, `-rb`) that share a global artifact set with language-specific overlays. Most contributions should land in the upstream `loom/` source-of-truth repo (which feeds all three templates via `/sync`); direct edits to this template will be overwritten on the next sync.

If your change is Ruby-specific, place it under `variants/rb/` in `loom/`. If it's universal (applies to py + rs + rb consumers), place it at the global tier in `loom/`.

### Rule Authoring

Every rule MUST follow the [Loud, Linguistic, Layered](https://github.com/terrene-foundation/loom/blob/main/.claude/rules/rule-authoring.md) test:

1. **Loud** — phrased as `MUST`/`MUST NOT`, not `should`/`prefer`
2. **Linguistic** — enumerates verbatim BLOCKED rationalizations the agent might use to skip
3. **Layered** — declares `priority:` + `scope:` in frontmatter so the emitter classifies it correctly

### Code of Conduct

Be respectful, constructive, and aligned with the Foundation's anti-capture provisions. No commercial coupling, no proprietary references — see `.claude/rules/independence.md`.

## Pull Request Process

1. Open a PR with a clear title and description
2. Reference any related issues
3. Wait for CI checks to pass (`Validate COC Structure` workflow)
4. A maintainer will review and merge

## Reporting Issues

For security vulnerabilities, see `SECURITY.md`. For other issues, open a GitHub issue with:
- A clear description of the problem
- Steps to reproduce (if applicable)
- Expected vs actual behavior

## License

By contributing, you agree that your contributions will be licensed under Apache 2.0.
