---
paths:
  - "lib/**/agents/**"
  - "**/kaizen/**"
  - "**/*agent*"
---

# Agent Reasoning Architecture — LLM-First Rule

## Scope

These rules apply to ALL agent code, ALL Kaizen implementations, ALL AI agent patterns, and ALL codegen that produces agent logic. This includes:

- `lib/**/agents/**`
- Any file creating, extending, or configuring a `Kailash::Kaizen::BaseAgent`
- Any file implementing agent routing, dispatch, classification, or decision-making
- Any code that processes user input to determine agent behavior

## The Principle: LLM Reasons, Tools Fetch

```
+---------------------------+       +---------------------------+
|         LLM AGENT         |       |          TOOLS            |
|                           |       |                           |
|  - Reasons about intent   |  -->  |  - Fetch data             |
|  - Decides what to do     |  <--  |  - Write data             |
|  - Classifies input       |       |  - Call APIs              |
|  - Routes to next step    |       |  - Execute queries        |
|  - Extracts information   |       |  - Return raw results     |
|  - Evaluates outcomes     |       |                           |
|                           |       |  Tools contain ZERO       |
|  ALL intelligence lives   |       |  decision logic.          |
|  in the LLM.              |       |  They are dumb endpoints. |
+---------------------------+       +---------------------------+
```

**The LLM IS the router, the classifier, the extractor, the evaluator.** Tools are dumb data endpoints — they fetch, store, and relay. They do not decide.

## ABSOLUTE RULE: No Deterministic Logic Where LLM Reasoning Belongs

When building an AI agent, the DEFAULT and ONLY mode is:

1. **LLM Reasons** — The agent receives context and thinks about what to do
2. **LLM Decides** — The agent chooses which tool to call, what to respond, or how to route
3. **LLM Acts** — The agent calls a tool (dumb data endpoint) or produces output
4. **LLM Evaluates** — The agent examines the result and decides the next step

Under NO circumstances shall the following be used for agent decision-making:

- `if-elsif` chains for intent routing
- Keyword matching (`if user_input.include?("cancel")`)
- Regex matching (`text.match?(/order.*refund/)`)
- Hash dispatch (`handlers = { "intent_a" => method(:func_a), "intent_b" => method(:func_b) }`)
- Symbol-based routing (`if intent == :billing`)
- Hardcoded classification (`if %w[help support].any? { |w| text.include?(w) }`)
- `case/when` statements on user input content
- Embedding similarity with hardcoded thresholds for routing

**UNLESS the user EXPLICITLY says**: "use deterministic logic here", "use keyword matching", "use regex", "this needs to be rule-based", or equivalent explicit opt-in.

## WARNING TO CODEGEN

```
+================================================================+
|                                                                  |
|   WARNING: DETERMINISTIC AGENT LOGIC DETECTED                   |
|                                                                  |
|   You are writing if-elsif/regex/keyword logic inside an agent.  |
|   This is ALMOST CERTAINLY WRONG.                                |
|                                                                  |
|   An AI agent's entire purpose is to REASON using an LLM.        |
|   If you're writing conditionals to route, classify, extract,    |
|   or decide — you're building a script, not an agent.            |
|                                                                  |
|   The LLM handles: routing, classification, extraction,          |
|                     evaluation, decision-making                  |
|   Tools handle:     fetching data, writing data, calling APIs    |
|                                                                  |
|   STOP. Let the LLM reason. Use agent.run with a clear           |
|   signature that describes what you need.                        |
|                                                                  |
+================================================================+
```

## MUST Rules

### 1. LLM-First for All Agent Decisions

Every agent decision — routing, classification, extraction, evaluation — MUST go through an LLM call (`agent.run`, `agent.run_async`), NOT through code conditionals.

```ruby
# DO: Let the LLM reason about intent
class CustomerServiceAgent < Kailash::Kaizen::BaseAgent
  signature do
    input  :user_message, desc: "Customer message"
    output :action,       desc: "Action to take: refund, escalate, answer, transfer"
    output :reasoning,    desc: "Why this action"
    output :response,     desc: "Response to customer"
  end

  def handle(message:)
    run(user_message: message)  # LLM decides everything
  end
end

# DO NOT: Write a script pretending to be an agent
class CustomerServiceAgent < Kailash::Kaizen::BaseAgent
  def handle(message:)
    lower = message.downcase
    if lower.include?("refund") || lower.include?("money back")  # <-- BLOCKED
      process_refund(message)
    elsif lower.include?("cancel")                                # <-- BLOCKED
      process_cancellation(message)
    elsif lower.match?(/order\s*#?\d+/)                           # <-- BLOCKED
      lookup_order(message)
    else
      run(user_message: message)                                  # Why isn't this the ONLY path?
    end
  end
end
```

### 2. Tools Are Dumb Data Endpoints

Tools MUST be pure data operations. They fetch, store, transform, or relay. They MUST NOT contain decision logic, routing, or classification.

```ruby
# DO: Tool that fetches data — no decisions
def get_order(order_id:)
  # Fetch order details from database.
  Order.find_by(id: order_id)&.as_json
end

# DO: Tool that writes data — no decisions
def issue_refund(order_id:, amount:)
  # Process a refund for the given order.
  PaymentApi.refund(order_id, amount)
end

# DO NOT: Tool that makes decisions
def handle_order_issue(order_id:, message:)
  order = Order.find(order_id)
  if order.status == "delivered"           # <-- Decision in tool! BLOCKED
    process_return(order)
  elsif order.total < 50                   # <-- Decision in tool! BLOCKED
    auto_refund(order)
  else
    { action: "escalate" }                 # <-- Decision in tool! BLOCKED
  end
end
```

### 3. Signatures Describe, Code Doesn't Decide

Agent signatures MUST describe the reasoning the LLM should perform. The code around `run` MUST NOT pre-filter, pre-classify, or pre-route before the LLM sees the input.

```ruby
# DO: Rich signature that tells the LLM what to reason about
class TriageAgent < Kailash::Kaizen::BaseAgent
  signature do
    input  :ticket,           desc: "Support ticket content"
    input  :customer_history, desc: "Previous interactions"
    output :priority,         desc: "urgent, high, normal, low"
    output :category,         desc: "billing, technical, account, general"
    output :suggested_action, desc: "What to do next"
    output :response_draft,   desc: "Draft response to customer"
  end
end

# DO NOT: Minimal signature because code handles the logic
class TriageAgent < Kailash::Kaizen::BaseAgent
  signature do
    input  :ticket,   desc: "Support ticket"
    output :response, desc: "Response"
    # ^ All the interesting decisions happen in if-elsif code
  end
end
```

### 4. Multi-Step Reasoning Uses Agent Loops, Not Code Loops

When an agent needs to perform multi-step reasoning, use ReAct patterns or multi-cycle strategies — NOT imperative code loops with conditionals.

```ruby
# DO: ReAct agent reasons about each step
agent = Kailash::Kaizen::ReActAgent.new(config: config, tools: :all)
result = agent.solve("Investigate why revenue dropped last quarter")
# Agent autonomously: searches data, reads reports, correlates, concludes

# DO NOT: Code loop with hardcoded steps
def investigate(query)
  data = fetch_revenue_data               # Step 1: hardcoded
  if data[:trend] == "declining"           # Step 2: hardcoded decision
    causes = fetch_cause_data              # Step 3: hardcoded
    if causes[:top] == "churn"             # Step 4: hardcoded decision
      generate_churn_report
    end
  end
end
```

### 5. Router Agents Use LLM Routing, Not Dispatch Tables

When routing between multiple agents, the router MUST use LLM reasoning to select the target — NOT a dispatch table or keyword map.

```ruby
# DO: LLM-based routing via Pipeline.router
router = Kailash::Kaizen::Pipeline.router(
  agents: [billing_agent, tech_agent, sales_agent]
)
result = router.run(query: user_message)
# LLM examines A2A capability cards and reasons about best match

# DO NOT: Dispatch table
ROUTES = {                                           # <-- BLOCKED
  "billing"   => billing_agent,
  "technical" => tech_agent,
  "sales"     => sales_agent,
}.freeze
def route(message)
  ROUTES.each do |keyword, agent|
    return agent if message.downcase.include?(keyword)  # <-- BLOCKED
  end
  default_agent
end
```

## MUST NOT Rules

### 1. MUST NOT Use Conditionals for Agent Routing

No `if`, `elsif`, `case/when`, or ternary expressions that route agent behavior based on input content analysis performed in code. The LLM analyzes content. Code routes based on LLM output structure (e.g., calling a tool the LLM selected is fine).

### 2. MUST NOT Use Keyword/Regex Matching on Agent Inputs

No `include?`, `match?`, `=~`, `scan`, `gsub` on user input or message content for the purpose of making agent decisions.

### 3. MUST NOT Put Decision Logic in Tools

Tools are data endpoints. If a tool contains `if-elsif` logic that determines what the agent should do next, that logic belongs in the LLM's signature, not the tool.

### 4. MUST NOT Pre-Filter Input Before LLM Sees It

Do not strip, classify, categorize, or route input before passing it to `run`. The LLM sees the raw input and reasons about it.

```ruby
# BLOCKED: Pre-classification before LLM
def process(message)
  category = classify(message)      # <-- Code classifying! BLOCKED
  if category == "simple"           # <-- Code routing! BLOCKED
    run(message: message, mode: "quick")
  else
    run(message: message, mode: "deep")
  end
end

# CORRECT: LLM does everything
def process(message)
  run(message: message)  # LLM decides depth, approach, everything
end
```

## Permitted Deterministic Logic (Explicit Exceptions)

The following uses of conditionals in agent code are PERMITTED because they are NOT agent reasoning — they are structural, safety, or data-format operations:

1. **Input validation** — `raise ArgumentError, "message required" if message.nil?` (validating presence/type, not content)
2. **Error handling** — `begin/rescue/end` for tool failures, API errors, timeouts
3. **Output formatting** — Transforming LLM output into API response shapes
4. **Safety guards** — Blocking dangerous operations, PII filtering, content policy enforcement
5. **Configuration branching** — `if config.async_mode?` then use async runtime
6. **Tool result parsing** — Extracting structured data from tool responses
7. **Rate limiting / circuit breaking** — Infrastructure-level flow control
8. **Explicit user opt-in** — User said "use keyword matching for this specific case"

**The test**: Is the conditional deciding what the agent should _think_ or _do_ based on input content? If yes, it belongs in the LLM. If it's structural plumbing, it's fine.

## Detection Patterns

Codegen and code review MUST flag these anti-patterns in agent code:

```ruby
# ANTI-PATTERN 1: Keyword routing
if user_input.include?("keyword")           # BLOCKED in agent decision paths
if %w[word1 word2].any? { |w| text.include?(w) }  # BLOCKED in agent decision paths

# ANTI-PATTERN 2: Regex classification
intent = text.match(/pattern/)              # BLOCKED in agent decision paths
entities = text.scan(/pattern/)             # BLOCKED in agent decision paths

# ANTI-PATTERN 3: Dispatch tables
handlers = { a: method(:func_a) }           # BLOCKED in agent decision paths
action_map[classified_intent].call(message) # BLOCKED in agent decision paths

# ANTI-PATTERN 4: Hardcoded classification
if sentiment_score > 0.8                    # BLOCKED — LLM evaluates sentiment
if message.split.length < 5                 # BLOCKED — LLM judges complexity
if message.start_with?("!")                 # BLOCKED — LLM interprets commands

# ANTI-PATTERN 5: Tool-side decisions
def tool_handler(data)
  if data[:type] == "urgent"                # BLOCKED — LLM determines urgency
    escalate(data)
  end
end
```

## Enforcement

- **intermediate-reviewer agent** — MUST flag deterministic agent logic as CRITICAL finding during code review
- **kaizen-specialist agent** — MUST refuse to generate agent code with deterministic routing unless user explicitly opts in
- **validate-workflow.js hook** — Agent files containing keyword/regex routing patterns in decision paths produce WARNING
- **Red-team agents** — During /redteam, MUST check all agent implementations for deterministic reasoning anti-patterns

## Why This Matters

Every `if message.include?("keyword")` you write is:

- **Fragile** — Breaks on synonyms, typos, rephrasing, multilingual input
- **Incomplete** — Misses cases you didn't anticipate (the long tail is infinite)
- **Wasteful** — You're paying for an LLM and not using it
- **Unmaintainable** — Every new case needs a new branch; the code grows forever
- **Defeating the purpose** — You built an agent to reason; then you lobotomized it with a script

The LLM handles ALL of these naturally. It generalizes. It understands intent, not keywords. It handles edge cases you never imagined. **Let it do its job.**

## Cross-References

- `rules/no-stubs.md` — No deferred implementation (related: don't stub reasoning with if-elsif)
- `rules/zero-tolerance.md` — Pre-existing deterministic routing MUST be refactored when found
- `rules/agents.md` Rule 3 — Framework specialist required for Kaizen work
- `.claude/agents/frameworks/kaizen-specialist.md` — Kaizen agent definition (enforces this rule)
- `.claude/skills/04-kaizen/kaizen-baseagent-quick.md` — BaseAgent quick reference
