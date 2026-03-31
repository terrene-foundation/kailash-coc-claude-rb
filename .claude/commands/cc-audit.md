# /cc-audit - Ruby Project Artifact Audit

Audit Claude Code artifacts for quality, completeness, and template alignment in Ruby projects.

## Checks

1. **Agent quality**: Descriptions under 120 chars, files under 400 lines
2. **Skill coverage**: SKILL.md answers 80% of routine questions
3. **Rule scoping**: Ruby rules have `paths:` frontmatter for `**/*.rb`
4. **Command brevity**: Under 150 lines each
5. **CLAUDE.md lean**: Under 200 lines, navigation-focused
6. **Ruby patterns**: String keys, block cleanup, no mutex wrapping
