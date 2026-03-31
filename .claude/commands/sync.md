# /sync - COC Artifact Sync (Ruby)

Pulls latest COC artifacts from the upstream template (`kailash-coc-claude-rb`) into this Ruby project.

## Usage

```
/sync
```

## What It Does

1. Reads `.claude/VERSION` to check artifact currency
2. Pulls global + Ruby-variant artifacts from the COC template
3. Merges without overwriting project-specific customizations
4. Updates `.claude/VERSION`

## Artifacts Synced

- `agents/` — Specialized agents for Ruby development
- `skills/` — Ruby SDK patterns, quick references
- `rules/` — Ruby coding standards, testing rules
- `commands/` — Slash commands with Ruby examples

## When to Sync

- After a new Kailash gem version is released
- When the COC template is updated with new agents/skills
- At the start of a new project phase
