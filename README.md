# Kailash COC Claude (Ruby)

<p align="center">
  <img src="https://img.shields.io/badge/platform-Claude%20Code-7C3AED.svg" alt="Claude Code">
  <img src="https://img.shields.io/badge/architecture-COC%205--Layer-blue.svg" alt="COC 5-Layer">
  <img src="https://img.shields.io/badge/language-Ruby-CC342D.svg" alt="Ruby">
  <img src="https://img.shields.io/badge/binding-Magnus-orange.svg" alt="Magnus FFI">
  <img src="https://img.shields.io/badge/license-Apache%202.0-blue.svg" alt="Apache 2.0">
</p>

<p align="center">
  <strong>Cognitive Orchestration for Codegen (COC)</strong><br>
  A five-layer cognitive architecture for <a href="https://docs.anthropic.com/en/docs/claude-code">Claude Code</a> that replaces unstructured vibe coding with institutionally aware, self-enforcing AI code generation.
</p>

---

> "The problem with vibe coding is not the AI model. It's the absence of institutional knowledge in the coding loop."

Vibe coding fails because AI forgets your conventions (amnesia), drifts across patterns (convention drift), has never seen your frameworks (framework ignorance), degrades over sessions (quality erosion), and generates security vulnerabilities faster than humans can review (security blindness). COC solves all five by encoding institutional knowledge directly into the AI's operating environment.

---

## What Is This?

This COC template is for **Ruby developers** who use the **Magnus-bound Kailash gem** (`gem install kailash`). You write Ruby — you never touch Rust. The Rust runtime accelerates workflow execution under the hood; you interact only with the Ruby API surface.

The Magnus binding is **free to install**, requires **no registration**, and has **no paid tier**. The gem is published to RubyGems like any other Ruby native extension.

## What's In This Template

```
.claude/
├── agents/      # Specialist agent definitions
├── skills/      # Numbered knowledge directories (01-29)
├── rules/       # Codified discipline rules (priority + path-scope)
├── commands/    # Slash commands (/analyze, /implement, /redteam, etc.)
├── hooks/       # SessionStart, PreToolUse, PostToolUse JS hooks
├── guides/      # Extended reference (rule-extracts, deterministic-quality)
└── settings.json
CLAUDE.md         # Project-level directives (loaded every session)
```

## Use This Template

1. Create a new repo from this template (GitHub: "Use this template")
2. Open the new repo in [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
3. The five COC layers (CLAUDE.md + agents + skills + rules + hooks) load automatically
4. Run `/sync` periodically to pull artifact updates from this template

## Variant Identity

This is the **Ruby variant** in the Kailash COC family. Sister templates:

- [`kailash-coc-claude-py`](https://github.com/terrene-foundation/kailash-coc-claude-py) — Python developers (pure-Python Kailash SDK)
- [`kailash-coc-claude-rs`](https://github.com/terrene-foundation/kailash-coc-claude-rs) — Python and Ruby developers using the Rust-backed binding

All three share the same global COC artifact set (rules, agents, skills, commands) with language-specific overlays applied at sync time.

## Architecture

The five-layer cognitive architecture:

1. **CLAUDE.md** — repo-level absolute directives (always loaded)
2. **Agents** — specialist sub-agents for delegation (analyst, dataflow-specialist, kaizen-specialist, etc.)
3. **Skills** — numbered knowledge directories with progressive disclosure
4. **Rules** — codified discipline as MUST/MUST NOT clauses with BLOCKED rationalizations
5. **Hooks** — session-start, user-prompt, tool-use enforcement scripts

For deep reference on the architecture, see `.claude/guides/co-setup/`.

## Maintained By

This template is maintained by the [Terrene Foundation](https://terrene.foundation) and licensed under Apache 2.0. Issues and contributions welcome — see `CONTRIBUTING.md`.

## License

Apache 2.0 — see `LICENSE`.
