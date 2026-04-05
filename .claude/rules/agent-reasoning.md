---
paths:
  - "lib/**/agents/**"
  - "**/kaizen/**"
  - "**/*agent*"
---

# Agent Reasoning Architecture — LLM-First Rule

**The LLM IS the router, the classifier, the extractor, the evaluator.** Tools are dumb data endpoints — they fetch, store, and relay. They do not decide.

## MUST Rules

### 1. LLM-First for All Agent Decisions

Every agent decision MUST go through `agent.run`, NOT code conditionals.

**Why:** Code conditionals for routing create brittle keyword-matching that fails on paraphrased input, while LLMs generalize across natural language variations.

```ruby
# DO: Let the LLM reason
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

# DO NOT: Script pretending to be an agent
def handle(message:)
  lower = message.downcase
  if lower.include?("refund")     # <-- BLOCKED
    process_refund(message)
  elsif lower.include?("cancel")  # <-- BLOCKED
    process_cancellation(message)
  else
    run(user_message: message)    # Why isn't this the ONLY path?
  end
end
```

### 2. Tools Are Dumb Data Endpoints

Tools MUST NOT contain decision logic, routing, or classification.

**Why:** Decision logic in tools is invisible to the LLM's reasoning trace, making agent behavior unexplainable and impossible to improve via prompt engineering.

```ruby
# DO: Tool that fetches data
def get_order(order_id:)
  Order.find_by(id: order_id)&.as_json
end

# DO NOT: Tool that makes decisions
def handle_order_issue(order_id:, message:)
  order = Order.find(order_id)
  if order.status == "delivered"  # <-- Decision in tool! BLOCKED
    process_return(order)
  end
end
```

### 3. Signatures Describe, Code Doesn't Decide

**Why:** Minimal signatures force developers to embed intelligence in Ruby code, producing agents that are glorified `case/when` chains instead of LLM reasoners.

```ruby
# DO: Rich signature
class TriageAgent < Kailash::Kaizen::BaseAgent
  signature do
    input  :ticket,           desc: "Support ticket content"
    input  :customer_history, desc: "Previous interactions"
    output :priority,         desc: "urgent, high, normal, low"
    output :category,         desc: "billing, technical, account, general"
    output :suggested_action, desc: "What to do next"
  end
end
```

### 4. Multi-Step Reasoning Uses Agent Loops, Not Code Loops

**Why:** Hardcoded step sequences cannot adapt when intermediate results change the plan, while ReAct agents re-evaluate strategy after each observation.

```ruby
# DO: ReAct agent reasons about each step
agent = Kailash::Kaizen::ReActAgent.new(config: config, tools: :all)
result = agent.solve("Investigate why revenue dropped last quarter")

# DO NOT: Code loop with hardcoded steps
def investigate(query)
  data = fetch_revenue_data
  if data[:trend] == "declining"  # Hardcoded decision — BLOCKED
    causes = fetch_cause_data
  end
end
```

### 5. Router Agents Use LLM Routing, Not Dispatch Tables

**Why:** Dispatch tables require enumerating every possible intent upfront and break silently on new intents, while LLM routing generalizes to unseen queries.

```ruby
# DO: LLM-based routing
router = Kailash::Kaizen::Pipeline.router(
  agents: [billing_agent, tech_agent, sales_agent]
)
result = router.run(query: user_message)

# DO NOT: Dispatch table
ROUTES = { "billing" => billing_agent }  # <-- BLOCKED
```

## MUST NOT

- No conditionals for agent routing based on input content
  **Why:** Code-based routing silently drops any input that doesn't match a hardcoded branch, creating invisible "agent doesn't respond" bugs.
- No keyword/regex matching (`include?`, `match?`, `=~`, `scan`) on agent inputs for decisions
  **Why:** Regex matching fails on synonyms, typos, and multilingual input -- exactly the cases where users most need the agent to work.
- No decision logic in tools
  **Why:** Splitting reasoning across LLM and Ruby code makes behavior unpredictable and impossible to debug from the prompt alone.
- No pre-filtering input before LLM sees it
  **Why:** Pre-filtering discards context the LLM needs for nuanced reasoning -- a message classified as "simple" by code may contain subtle cues only the LLM would catch.

## Permitted Deterministic Logic

1. **Input validation** — `raise ArgumentError if message.nil?`
2. **Error handling** — `begin/rescue/end` for tool failures
3. **Output formatting** — Transforming LLM output into API shapes
4. **Safety guards** — PII filtering, content policy
5. **Configuration branching** — `if config.async_mode?`
6. **Rate limiting / circuit breaking**
7. **Explicit user opt-in**

**The test**: Is the conditional deciding what the agent should _think_ based on input content? → LLM. Structural plumbing? → fine.
