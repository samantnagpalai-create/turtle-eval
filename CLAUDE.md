# turtle-eval — Claude Code Instructions

## What This Project Is

turtle-eval is a simulation-first observability and evals platform for AI. It lets teams define skills and workflows, simulate them against scenario packs, score outputs, collect human annotations, and gate promotion to production on passing defined criteria.

The core loop: **simulate → score → annotate → promote → observe → feed failures back to simulate**

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│              SIMULATION ENGINE                       │
│   Skills | Workflows | Chaos | Adversarial inputs   │
├─────────────────────────────────────────────────────┤
│              TRACE STORE                             │
│   Postgres (metadata) + S3 (payloads)               │
├─────────────────────────────────────────────────────┤
│              EVAL PIPELINE                           │
│   Offline evals | Online evals | LLM-as-judge       │
├─────────────────────────────────────────────────────┤
│              ANNOTATION LAYER                        │
│   Review queue | Labeling UI | Dataset builder      │
├─────────────────────────────────────────────────────┤
│              PROMOTION GATES                         │
│   Promotion policies | CI/CD integration | Canary   │
└─────────────────────────────────────────────────────┘
```

---

## Coding Conventions

- **Always add plain English inline comments** to every file — explain what each section does clearly enough for a non-engineer to understand
- Use **TypeScript** for the backend API and frontend UI
- Use **Python** for the simulation engine, eval runners, and SDK
- All data models must include `created_at`, `updated_at`, and `version` fields
- Scorer results must always reference the scorer version, not just the skill version
- Never use the same model for LLM-as-judge scoring that you are evaluating — always use a separate pinned model

---

## Key Data Models

### Skill
```typescript
type Skill = {
  id: string
  name: string
  system_prompt: string
  tools: Tool[]
  context_template: ContextTemplate
  input_schema: JSONSchema
  scoring_criteria: ScoringCriteria[]
  version: string
  created_at: Date
  updated_at: Date
}
```

### Span (Trace Node)
```typescript
type Span = {
  id: string
  parent_id: string | null
  trace_id: string
  skill_id: string
  workflow_id: string | null
  input_context: object       // full context window snapshot — not just the prompt
  model: string
  temperature: number
  tools_available: Tool[]
  tool_calls: ToolCall[]
  output: string
  latency_ms: number
  tokens_in: number
  tokens_out: number
  cost_usd: number
  eval_scores: Record<string, number>
  annotations: Annotation[]
}
```

### Score
```typescript
type Score = {
  trace_id: string
  span_id: string
  scorer_name: string
  scorer_type: 'llm' | 'deterministic' | 'human'
  scorer_version: string
  value: number              // normalized 0-1
  rationale: string
  metadata: object
}
```

### Promotion Policy
```typescript
type PromotionPolicy = {
  skill_id: string
  required_scenario_packs: string[]
  min_pass_rate: number      // e.g. 0.95
  required_human_reviews: number
  regression_check: boolean
  canary_traffic_pct: number
}
```

---

## Storage Rules

- **Postgres**: span metadata, scores, annotations, policies, skill/workflow definitions
- **S3-compatible object store**: full prompt/completion payloads (they can exceed 50KB per span)
- **Parquet/Arrow**: offline evaluation datasets exported for fine-tuning pipelines
- Never store full payloads in Postgres — always reference the S3 key

---

## Eval Pipeline Rules

- Offline evals run against every simulation before promotion — blocking
- Online evals sample live production traces — non-blocking but trigger alerts
- LLM-as-judge scorers must use a **pinned, separate model** — never the model under evaluation
- Every scorer must have a version — score comparisons across versions are invalid
- When online eval scores drop more than the configured threshold in a rolling window, trigger a regression alert

---

## Simulation Engine Rules

- Every skill must have at least one scenario pack before it can be promoted
- Scenario packs must cover three tiers: nominal, edge, and chaos
- Chaos scenarios must include at minimum: hostile input, ambiguous query, tool failure injection, context truncation
- Full context must be stored in traces — including retrieved documents, tool results, and conversation history — so replays are faithful to production conditions

---

## What NOT to Do

- Do not build a traditional APM-style platform (metrics/logs/alerts) — this is a trace/eval/annotation platform
- Do not use the same model for scoring that you are evaluating
- Do not store only the prompt in traces — store the full context window
- Do not use static golden datasets — annotations feed continuously reconciled datasets
- Do not allow promotion without a passing promotion policy check
