# Kailash COC Claude (Ruby)

This repository is the **COC (Cognitive Orchestration for Codegen) setup** for building with the Kailash Ruby SDK — providing agents, skills, rules, and hooks for Kailash SDK development via Magnus (Rust FFI) bindings.

## Absolute Directives

These override ALL other instructions.

### 0. Foundation Independence — No Commercial Coupling

Kailash Ruby SDK is a **Terrene Foundation project**. Fully independent. NO relationship to any commercial product. See `rules/independence.md`.

### 1. Framework-First

Never write code from scratch before checking whether Kailash frameworks handle it.

- Instead of direct SQL/ActiveRecord → check with **dataflow-specialist**
- Instead of custom API/Sinatra/Rails → check with **nexus-specialist**
- Instead of custom MCP server → check with **mcp-specialist**
- Instead of custom agent framework → check with **kaizen-specialist**
- Instead of custom governance → check with **pact-specialist**

### 2. .env Is the Single Source of Truth

All API keys and model names MUST come from `.env`. Never hardcode model strings.

### 3. Implement, Don't Document

When you discover a missing feature — **implement it**. Do not note it as a gap. See `rules/zero-tolerance.md`.

### 4. Zero Tolerance

Pre-existing failures MUST be fixed, not reported. Stubs BLOCKED. See `rules/zero-tolerance.md`.

### 5. Recommended Reviews

- **Code review** (reviewer) after file changes
- **Security review** (security-reviewer) before commits
- **Real infrastructure recommended** in Tier 2/3 tests

### 6. LLM-First Agent Reasoning

When building AI agents: **the LLM does ALL reasoning. Tools are dumb data endpoints.** No if-else routing, no keyword matching. See `rules/agent-reasoning.md`.

## Ruby SDK Essentials

```ruby
# Block-based runtime (MUST — ensures native cleanup)
Kailash::Runtime.open(registry) do |rt|
  result = rt.execute(wf, params)
end

# String-based node IDs, string keys in hashes (Magnus FFI requirement)
builder.add_node("LogNode", "log", { "message" => "hello" })

# Close native objects (GC does not call Rust destructors)
begin
  registry = Kailash::Registry.new
  wf = builder.build(registry)
ensure
  wf&.close
  registry&.close
end

# Express API for single-record CRUD
result = db.express.create("User", { "name" => "Alice" })
```

See `rules/patterns.md` for complete Ruby patterns and MUST NOT rules (no Mutex wrapping, no fork after Registry).

## Workspace Commands

| Command      | Phase | Purpose                                            |
| ------------ | ----- | -------------------------------------------------- |
| `/analyze`   | 01    | Load analysis phase for current workspace          |
| `/todos`     | 02    | Load todos phase; stops for human approval         |
| `/implement` | 03    | Load implementation phase; repeat until todos done |
| `/redteam`   | 04    | Load validation phase; red team with MCP tools     |
| `/codify`    | 05    | Load codification phase; create agents & skills    |
| `/release`   | --    | SDK release: RubyGems publishing, CI (standalone)  |
| `/ws`        | --    | Read-only workspace status dashboard               |
| `/wrapup`    | --    | Write session notes before ending                  |
| `/journal`   | --    | View, create, or search project journal entries    |

## Rules Index

| Concern                    | Rule File                       | Scope                                                       |
| -------------------------- | ------------------------------- | ----------------------------------------------------------- |
| **Foundation independence**| `rules/independence.md`         | **Global**                                                  |
| **Autonomous execution**   | `rules/autonomous-execution.md` | **Global**                                                  |
| **LLM-first reasoning**    | `rules/agent-reasoning.md`      | `**/kaizen/**`, `**/*agent*`                                |
| **Zero tolerance**         | `rules/zero-tolerance.md`       | **Global**                                                  |
| Agent orchestration        | `rules/agents.md`               | Global                                                      |
| Ruby SDK patterns          | `rules/patterns.md`             | `**/*.rb`, `**/Gemfile`, `**/Rakefile`, `**/*.gemspec`      |
| CC artifact quality        | `rules/cc-artifacts.md`         | `.claude/**`, `scripts/hooks/**`                            |
| Communication style        | `rules/communication.md`        | Global                                                      |
| Connection pool safety     | `rules/connection-pool.md`      | Database connection code                                    |
| DataFlow pool config       | `rules/dataflow-pool.md`        | `**/dataflow/**`                                            |
| Gem release & publishing   | `rules/deployment.md`           | `*.gemspec`, `Gemfile`, `lib/**/version.rb`, `.github/**`   |
| Documentation standards    | `rules/documentation.md`        | `README.md`, `docs/**`, `CHANGELOG.md`                      |
| E2E testing                | `rules/e2e-god-mode.md`         | `spec/e2e/**`, `**/*e2e*`                                   |
| EATP SDK conventions       | `rules/eatp.md`                 | `**/trust/**`, `**/eatp/**`                                 |
| API keys & model names     | `rules/env-models.md`           | `**/*.rb`, `.env*`                                          |
| Git workflow               | `rules/git.md`                  | Global                                                      |
| Infrastructure SQL safety  | `rules/infrastructure-sql.md`   | `**/db/**`, `**/infrastructure/**`                          |
| PACT governance            | `rules/pact-governance.md`      | `**/pact/**`, `**/governance/**`                            |
| Security                   | `rules/security.md`             | Global                                                      |
| Terrene naming             | `rules/terrene-naming.md`       | Global                                                      |
| 3-tier testing             | `rules/testing.md`              | `spec/**`, `**/*test*`, `**/*spec*`                         |
| Trust-plane security       | `rules/trust-plane-security.md` | `**/trust/**`                                               |

## Agents

### Analysis (`agents/analysis/`)

- **analyst** — Failure point analysis, risk assessment, requirements breakdown, ADRs

### Framework Specialists (`agents/frameworks/`)

- **dataflow-specialist** — Database operations, auto-generated nodes
- **nexus-specialist** — Multi-channel platform (API/CLI/MCP)
- **kaizen-specialist** — AI agents, signatures, multi-agent coordination
- **mcp-specialist** — MCP server implementation
- **mcp-platform-specialist** — FastMCP platform server, contributor plugins, security tiers
- **pact-specialist** — Organizational governance (D/T/R, envelopes, clearance)
- **ml-specialist** — ML lifecycle, feature stores, training, drift monitoring
- **align-specialist** — LLM fine-tuning, LoRA adapters, alignment, model serving

### Implementation (`agents/implementation/`)

- **pattern-expert** — Workflow patterns, nodes, parameters
- **tdd-implementer** — Test-first development
- **build-fix** — Fix build/type errors with minimal changes

### Quality (`agents/quality/`)

- **reviewer** — Code review, doc validation, cross-reference accuracy
- **gold-standards-validator** — Compliance checking
- **security-reviewer** — Security audit before commits

### Frontend (`agents/frontend/`)

- **react-specialist** — React/Next.js frontends
- **flutter-specialist** — Flutter mobile/desktop apps
- **uiux-designer** — Enterprise UI/UX design

### Testing (`agents/testing/`)

- **testing-specialist** — 3-tier strategy with real infrastructure, E2E

### Release (`agents/release/`)

- **release-specialist** — CI/CD, RubyGems publishing, deployment

### Management (`agents/management/`)

- **todo-manager** — Project task tracking
- **gh-manager** — GitHub issue/project management

### Other

- **claude-code-architect** — CC artifact quality auditing
- **open-source-strategist** — Licensing, community building
- **value-auditor** — Enterprise demo QA from buyer perspective

## Skills Navigation

SDK implementation patterns: `.claude/skills/` — organized by framework (`01-core-sdk` through `05-kailash-mcp`), enterprise infrastructure (`15-enterprise-infrastructure`), and topic (`06-cheatsheets` through `28-coc-reference`).

## Kailash Platform

| Framework    | Purpose                                | Install                       |
| ------------ | -------------------------------------- | ----------------------------- |
| **Core SDK** | Workflow orchestration, 140+ nodes     | `gem install kailash`         |
| **DataFlow** | Zero-config database operations        | `gem install kailash-dataflow`|
| **Nexus**    | Multi-channel deployment (API+CLI+MCP) | `gem install kailash-nexus`   |
| **Kaizen**   | AI agent framework                     | `gem install kailash-kaizen`  |
| **PACT**     | Organizational governance (D/T/R)      | `gem install kailash-pact`    |

All frameworks are built ON Core SDK via Magnus (Rust FFI) bindings.
