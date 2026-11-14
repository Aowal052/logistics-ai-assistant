# logistics-ai-assistant

An AI assistant for logistics operations: ask questions about orders, shipments and courier (3PL) behavior in natural language, get grounded answers, and let an agent take safe, auditable actions.

This is a hands-on project to practice the full AI engineering stack in one place: **LLM APIs, RAG, agents / tool use, evaluation, observability and guardrails**, built on a backend-first foundation (.NET, PostgreSQL, Redis, Docker).

> Status: Phase 0 (scaffolding). See the [phase table](#phases) for the plan.

---

## What it does (target)

- **Answer questions** like "Why is order #1234 stuck?" or "What does our policy say about returns after a failed delivery?" using RAG over SOPs, courier docs and policy notes.
- **Look up live data** through tools (order status, shipment timeline, rider/courier info) instead of guessing.
- **Take actions with approval**, e.g. propose a cancellation or return, then wait for a human to confirm (maker/checker style).
- **Be measurable**: every change is checked against an eval set, and every request is traced.

> All data is synthetic or anonymized. No real customer or company data goes in this repo.

---

## Architecture sketch

```
                          ┌──────────────────────┐
                          │   Client (CLI / Web)  │
                          └──────────┬───────────┘
                                     │ HTTP / SSE (streaming)
                          ┌──────────▼───────────┐
                          │      Assistant API    │
                          │  (.NET minimal API)   │
                          └──────────┬───────────┘
                                     │
                  ┌──────────────────▼──────────────────┐
                  │            Guardrails (in)           │
                  │  input validation · PII masking ·    │
                  │  prompt-injection checks · rate limit│
                  └──────────────────┬──────────────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Orchestrator/Agent  │◄───────┐
                          │  (LLM loop + routing) │        │ tool results
                          └───┬──────────┬───────┘        │
                              │          │                 │
              ┌───────────────▼──┐   ┌───▼────────────────┴───┐
              │   RAG pipeline   │   │      Tool layer         │
              │ chunk → embed →  │   │ get_order_status        │
              │ retrieve → rerank│   │ get_shipment_timeline   │
              └───────┬──────────┘   │ search_policy           │
                      │              │ propose_action (HITL)   │
              ┌───────▼──────────┐   └───┬────────────────────┘
              │ PostgreSQL +     │       │
              │ pgvector (docs)  │   ┌───▼──────────────┐
              └──────────────────┘   │ Mock logistics    │
                                     │ service / DB      │
                                     └──────────────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Guardrails (out)    │
                          │ schema validation ·   │
                          │ grounding check ·     │
                          │ action allow-list     │
                          └──────────┬───────────┘
                                     │
                              response to client

   Cross-cutting:  Redis (cache, rate limit, conversation state)
                   OpenTelemetry traces + structured logs + token/cost metrics
                   Eval harness (offline, runs in CI) → golden set + LLM-as-judge
```

### Components

| Component | Responsibility |
|---|---|
| Assistant API | Chat endpoint, streaming, auth, request/response contracts |
| Orchestrator | LLM loop: decide between answering, retrieving, or calling a tool |
| RAG pipeline | Ingest docs, chunk, embed, store in pgvector, retrieve and rerank |
| Tool layer | Typed, validated functions the model can call; read tools first, write tools gated |
| Guardrails | Input and output checks, PII handling, injection defense, action allow-list |
| Observability | Traces per request, prompt/response logging (redacted), latency, tokens, cost |
| Eval harness | Golden Q&A set, retrieval metrics, tool-call accuracy, regression gate in CI |

---

## Phases

An 8-week plan. Each phase ends with something runnable and a short write-up in `docs/`.

| Phase | Week | Focus | Deliverable | Done when |
|---|---|---|---|---|
| 0 | 0 | Repo + scaffolding | Repo, README, Docker Compose (API, Postgres+pgvector, Redis), CI skeleton | `docker compose up` starts a health-checked API |
| 1 | 1 | LLM API fundamentals | Chat endpoint with streaming, prompt templates, structured (JSON) output, retries/timeouts | Endpoint answers with streamed, schema-valid output |
| 2 | 2 | RAG: ingestion & retrieval | Doc ingestion, chunking, embeddings, pgvector search | Questions about SOP docs return cited passages |
| 3 | 3 | RAG: quality | Hybrid search, reranking, chunking experiments, citation format | Retrieval hit-rate measured and improved over baseline |
| 4 | 4 | Tool use | Read-only tools (order status, shipment timeline) with typed schemas | Model calls the right tool with valid args on test cases |
| 5 | 5 | Agents & actions | Multi-step agent loop, state in Redis, human-approval flow for write actions | Agent proposes an action and waits for confirmation |
| 6 | 6 | Evaluation | Golden set, LLM-as-judge, tool-call accuracy, regression gate in CI | PR fails if eval score drops below threshold |
| 7 | 7 | Observability | OpenTelemetry traces, token/cost/latency metrics, dashboards | Any request can be traced end to end with its cost |
| 8 | 8 | Guardrails & hardening | Prompt-injection tests, PII masking, output validation, rate limits | Red-team test suite passes; write-ups finished |

---

## Tech stack

- **Backend:** .NET 8 (C#), minimal APIs
- **Data:** PostgreSQL + pgvector, Redis
- **LLM:** provider-agnostic client interface (swap models without touching the orchestrator)
- **Infra:** Docker, Docker Compose, GitHub Actions
- **Observability:** OpenTelemetry, structured logging

---

## Repo layout (planned)

```
logistics-ai-assistant/
├── src/
│   ├── Assistant.Api/          # HTTP layer, streaming, auth
│   ├── Assistant.Core/         # orchestrator, agent loop, abstractions
│   ├── Assistant.Rag/          # ingestion, embeddings, retrieval
│   ├── Assistant.Tools/        # tool definitions + validation
│   ├── Assistant.Guardrails/   # input/output checks
│   └── MockLogistics/          # fake orders/shipments for tools
├── evals/                      # golden sets, judges, runners
├── data/                       # synthetic docs and seed data
├── docs/                       # per-phase write-ups, ADRs
├── docker-compose.yml
└── README.md
```

---

## Getting started

```bash
git clone https://github.com/<your-username>/logistics-ai-assistant.git
cd logistics-ai-assistant

cp .env.example .env        # add your LLM provider API key
docker compose up --build
```

API health check: `http://localhost:8080/health`

---

## Design principles

1. **Ground, don't guess.** Answers come from retrieved docs or tool results, with citations.
2. **Read before write.** Write actions are proposals until a human approves them.
3. **Measure everything.** No prompt or retrieval change merges without an eval run.
4. **Trace everything.** Every request is reproducible from its trace.
5. **Synthetic data only.** Nothing from a real employer or customer.

---

## Progress log

- [ ] Phase 0: scaffolding
- [ ] Phase 1: LLM API fundamentals
- [ ] Phase 2: RAG ingestion & retrieval
- [ ] Phase 3: RAG quality
- [ ] Phase 4: Tool use
- [ ] Phase 5: Agents & actions
- [ ] Phase 6: Evaluation
- [ ] Phase 7: Observability
- [ ] Phase 8: Guardrails & hardening

---

## License

MIT
