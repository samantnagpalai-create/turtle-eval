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
