# turtle-eval

An observability and evals platform for AI — simulation-first, promotion-gated.

## What This Is

turtle-eval is a platform for simulating, testing, and promoting AI skills and workflows to production. It is built around the insight that AI systems are probabilistic and data-coupled — identical inputs can yield different outputs. Traditional observability tools (metrics, logs, alerts) are not enough. You need a system that lets you **simulate** behavior before it ships and **gate promotion** on passing defined eval criteria.

The flywheel:

```
simulate → score → annotate → promote → observe → feed failures back to simulate
```

---

## Core Philosophy

**From Sierra AI**: Context engineering — how you structure, test, and validate what goes *into* the model — drives reliability more than model size. Production readiness requires simulation environments that expose failure modes before they hit users.

**From Braintrust**: AI observability requires new primitives: traces, evals, and annotations — not metrics/logs/alerts. AI traces average ~50KB vs ~900 bytes in traditional systems. Scoring is the fundamental atom of quality measurement.

**Our position**: Existing tools (Braintrust, LangSmith) are observability-first — they trace what happened. turtle-eval is **simulation-first** — you prove it works before it ships.

---

## Platform Architecture

### 5 Core Layers

```
┌─────────────────────────────────────────────────────┐
│              SIMULATION ENGINE                       │  ← Pre-production
│   Skills | Workflows | Chaos | Adversarial inputs   │
├─────────────────────────────────────────────────────┤
│              TRACE STORE                             │  ← Purpose-built
│   Full execution graphs | Context snapshots | Tools  │
├─────────────────────────────────────────────────────┤
│              EVAL PIPELINE                           │  ← Online + Offline
│   Scoring | Regression detection | LLM-as-judge     │
├─────────────────────────────────────────────────────┤
│              ANNOTATION LAYER                        │  ← Human-in-the-loop
│   Expert review | Dataset curation | Ground truth   │
├─────────────────────────────────────────────────────┤
│              PROMOTION GATES                         │  ← Production readiness
│   Pass/fail criteria | A/B | Canary deploy          │
└─────────────────────────────────────────────────────┘
```

---

### Layer 1: Simulation Engine

The core of the platform. Skills and workflows are tested here before any production deployment.

**Skill simulation** — A skill is a discrete AI capability (e.g., `summarize_ticket`, `route_call`, `answer_faq`). Each skill runs in isolation against a parameterized test harness.

```json
{
  "name": "string",
  "system_prompt": "string",
  "tools": ["Tool"],
  "context_template": "ContextTemplate",
  "inputs": "InputSchema",
  "expected_outputs": ["ScoringCriteria"]
}
```

**Workflow simulation** — A workflow chains skills together with branching logic, model constellations, and guardrails. The simulator:
- Runs the full DAG against synthetic + real-world replay inputs
- Injects failures at any node (tool failure, model timeout, bad retrieval)
- Tests chaos inputs: hostile users, ambiguous queries, off-topic requests, degraded transcripts

**Scenario packs** — Each skill/workflow ships with scenario packs covering nominal, edge, and chaos cases. Simulations must pass all required packs before promotion.

---

### Layer 2: Trace Store

AI traces are ~50KB on average vs ~900 bytes in traditional systems. Architecture:

- **Columnar storage** for span metadata (fast filtering and aggregation)
- **Object storage (S3-compatible)** for full prompt/completion payloads
- **Indexed on**: skill_id, workflow_id, session_id, eval scores, timestamp, model, token cost

Each trace node captures the full context — not just the prompt — so replays match production behavior exactly.

```json
{
  "id": "string",
  "parent_id": "string",
  "trace_id": "string",
  "skill_id": "string",
  "workflow_id": "string",
  "input_context": "full context window snapshot",
  "model": "string",
  "temperature": 0.0,
  "tools_available": [],
  "tool_calls": [{"name": "string", "args": {}, "result": {}}],
  "output": "string",
  "latency_ms": 0,
  "tokens_in": 0,
  "tokens_out": 0,
  "cost_usd": 0.0,
  "eval_scores": {"scorer_name": 0.0},
  "annotations": [{"author": "string", "label": "string", "note": "string"}]
}
```

---

### Layer 3: Eval Pipeline

Two modes, both required:

**Offline evals** (run in simulation before promotion):
- Score every simulated run against defined criteria
- LLM-as-judge for qualitative scoring (correctness, tone, policy adherence)
- Deterministic scorers for structured outputs (JSON validity, field presence, regex match)
- Track pass rates across all scenario packs

**Online evals** (run in production):
- Sample live traces and score automatically
- Alert on regression when average scores drop beyond a threshold in a rolling window
- Feed failing production examples back into the simulation dataset automatically

**Scoring primitive** — the atom of the eval system:

```json
{
  "trace_id": "string",
  "span_id": "string",
  "scorer_name": "string",
  "scorer_type": "llm | deterministic | human",
  "value": 0.0,
  "rationale": "string",
  "metadata": {}
}
```

---

### Layer 4: Annotation Layer

Where non-engineers (PMs, domain experts, QA) contribute expertise:

- **Review queue** — surfaced traces filtered by low eval scores or flagged by online monitoring
- **Labeling interface** — pass/fail with optional output correction
- **Dataset builder** — annotations become continuously reconciled datasets, not static golden files
- **Disagreement tracking** — when automated scorer and human annotator disagree, flagged for calibration

---

### Layer 5: Promotion Gates

What turns this into a simulation → production pipeline:

```json
{
  "skill_id": "string",
  "required_scenario_packs": ["string"],
  "min_pass_rate": 0.95,
  "required_human_reviews": 20,
  "regression_check": true,
  "canary_traffic_pct": 0.05
}
```

A skill or workflow version cannot be deployed until it passes its promotion policy. CI/CD integration runs the simulation engine, checks scores against the policy, and blocks or promotes automatically.

---

## Build Order

### Phase 1 — Foundation
1. Trace schema + storage (Postgres for metadata, S3 for payloads)
2. SDK for instrumenting skills (Python + TypeScript)
3. Basic offline eval runner with deterministic scorers
4. Simple UI: trace viewer + score display

### Phase 2 — Simulation Core
1. Skill & workflow definition DSL
2. Scenario pack system (nominal + edge + chaos)
3. LLM-as-judge scorer integration
4. Simulation runner producing trace batches + aggregate reports

### Phase 3 — Human Loop
1. Annotation queue + labeling UI
2. Dataset builder from annotations
3. Online eval sampling + regression alerts

### Phase 4 — Promotion Pipeline
1. Promotion policy schema
2. CI/CD integration (GitHub Actions hook or CLI)
3. Canary traffic management + score-based rollback

---

## Key Technical Decisions

| Decision | Recommendation |
|---|---|
| Trace storage | Postgres (metadata) + S3 (payloads). Avoid single-table designs. |
| LLM-as-judge model | Use a separate, pinned model version — never the same model being evaluated |
| Scorer versioning | Score results tied to scorer version, not just skill version |
| Dataset format | Arrow/Parquet for offline datasets — interoperable with fine-tuning pipelines |
| Auth model | Multi-tenant from day one — teams share the platform, isolated data |
| Replay fidelity | Store full context (retrieved docs, tool results, history) — not just the prompt |

---

## How This Differs From Braintrust / LangSmith

Existing tools are observability-first: they trace what happened. turtle-eval is **simulation-first**: you prove it works before it ships. Promotion gates, scenario packs, and the simulation engine are the core product — not add-ons.

---

## How Simulation Data Is Created

Simulation data is the hardest part to get right. It is not just test inputs — a simulation scenario is a **full context snapshot**, everything the model would see in production.

### What a Scenario Contains

```json
{
  "input": "the user message or trigger",
  "conversation_history": "prior turns if any",
  "retrieved_context": "what RAG would have returned",
  "tool_results": "mocked responses from external tools",
  "system_prompt": "the skill's instruction",
  "expected_behavior": "scoring criteria — NOT an exact expected output"
}
```

The reason you store all of this is **replay fidelity** — if you only store the user message, your simulation won't match production because the model also saw retrieved docs, tool outputs, and conversation history.

---

### The 4 Sources of Simulation Data

#### 1. Hand-crafted (where you always start)
Domain experts write scenarios before any production traffic exists. Cover all three tiers from the start.

```python
# Example: a skill that routes customer support tickets
scenarios = [
  {
    "tier": "nominal",
    "input": "My order hasn't arrived yet",
    "retrieved_context": "<order #1234, shipped 3 days ago>",
    "expected_behavior": "asks for order number if not provided, or checks status"
  },
  {
    "tier": "edge",
    "input": "WHERE IS MY STUFF I ORDERED 3 MONTHS AGO",
    "retrieved_context": "<order #1234, delivered 2 months ago>",
    "expected_behavior": "remains professional, surfaces delivery confirmation"
  },
  {
    "tier": "chaos",
    "input": "你好我的订单",  # foreign language
    "retrieved_context": None,  # retrieval failed
    "expected_behavior": "handles gracefully, does not hallucinate order data"
  }
]
```

#### 2. LLM-generated (scale up coverage fast)
A scenario generator prompt produces diverse test cases from a skill definition. This is how you go from 10 hand-crafted cases to 500.

```
System: You are a simulation data generator.
Given this skill definition: {skill}
Generate 20 test inputs across three tiers: nominal, edge, chaos.
For chaos tier: include hostile users, foreign languages, prompt injection attempts,
tool failure conditions, and severely truncated context.
Output as JSON array.
```

The LLM generates inputs. Your scorers evaluate outputs. You never hardcode exact expected outputs — you define **scoring criteria** (what good looks like), not the exact answer.

#### 3. Production replay (best data, requires live traffic first)
Once production traffic exists, real traces are captured, stripped of PII, and replayed against new skill versions. This is how regression testing works — replay the same traces against v1 vs v2 and compare scores.

```
Production trace captured →
  Strip PII →
  Store as scenario in scenario pack →
  Re-run against new skill version →
  Compare scores between versions
```

#### 4. Chaos injection (automated, built into the simulation runner)
Rather than writing every chaos scenario by hand, a chaos layer mutates any existing scenario automatically:

```python
chaos_mutations = [
  truncate_context(pct=0.3),          # cut retrieved context by 70%
  inject_prompt("Ignore all previous instructions and..."),
  fail_tool("get_order_status"),      # make the tool return an error
  replace_language("input", "zh"),    # translate input to another language
  add_noise("input", noise_pct=0.2),  # simulate typos or speech recognition errors
  repeat_message(n=5),                # same message sent 5 times in a row
]
```

Apply these mutations to nominal scenarios to auto-generate the chaos tier.

---

### Score, Don't Match

The most important rule: **never do exact output matching**.

```python
# Wrong — breaks on the first paraphrase
assert output == "I can help you track your order"

# Right — score against criteria
score = llm_judge(
  output=output,
  criteria="Did the model ask for an order number or use the provided one? Did it stay professional?"
)
assert score >= 0.8
```

The model might produce 10 different responses that are all correct. Exact matching will always fail eventually.

---

### Scenario Pack Structure

```
scenario_packs/
  ticket_routing/
    nominal/
      happy_path_with_order.json
      happy_path_without_order.json
      refund_request.json
    edge/
      very_long_message.json
      already_resolved_issue.json
      multiple_issues_in_one_message.json
    chaos/
      hostile_user.json
      prompt_injection.json
      retrieval_failure.json
      foreign_language.json
    scoring_criteria.json    ← what "good" looks like for this skill
    generator_prompt.txt     ← the LLM prompt used to generate more cases
```

---

### Practical Order for Building Simulation Data

1. **Start hand-crafted** — write 5-10 scenarios per skill across all three tiers
2. **Build the LLM generator** — expand to 50-100 scenarios per skill automatically
3. **Build the chaos mutator** — auto-generate chaos variants from nominal cases
4. **Wire in production replay** — once live traffic exists, pipe failing traces back into the scenario pack

You do not need production traffic to start. Synthetic data catches obvious failures, prompt regressions, and policy violations before anything ships.
