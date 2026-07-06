# Mini Agent Orchestrator

A lightweight, event-driven order-processing agent. It takes a natural
language request, turns it into a plan, and executes that plan against
async tools — with guardrails so a failed cancellation never triggers a
false "your order is cancelled" email.

```
"Cancel my order #9921 and email me at user@example.com"
        │
        ▼
   Planner (LLM or mock)  →  validated Plan (whitelisted tools, typed args)
        │
        ▼
   Orchestrator  →  cancel_order()  →  send_email()   [only if cancel succeeded]
        │
        ▼
   JobRecord (pending → running → succeeded/failed), pollable at any time
```

## Running it

```bash
pip install -r requirements.txt
uvicorn app.main:app --reload
```

No API key is required — without `OPENAI_API_KEY` set, the planner
automatically uses a deterministic mock parser (see below), so the
service is fully runnable and testable out of the box. To use the real
OpenAI API instead, copy `.env.example` to `.env` and set the key.

Run tests:
```bash
pytest
```

## API

**POST `/process`** — submit a request.
```bash
curl -X POST http://127.0.0.1:8000/process \
  -H "Content-Type: application/json" \
  -d '{"request": "Cancel my order #9921 and email me the confirmation at user@example.com."}'
# -> {"job_id": "...", "status": "pending", "plan_source": "mock_fallback"}
```
Add `?sync=true` to instead wait for the job to finish and get the full
result back in one response (convenient for quick manual testing).

**GET `/jobs/{job_id}`** — poll for live status / final result.
```bash
curl http://127.0.0.1:8000/jobs/<job_id>
```

## Architectural choices

### 1. Planning: real LLM with a safety-netted fallback
`app/planner.py` calls the OpenAI API (JSON-mode, temperature 0) with a
strict system prompt describing the exact JSON schema expected. If no
API key is configured, **or** the call fails, **or** the model returns
something that doesn't parse as JSON, the planner transparently falls
back to a small deterministic regex-based parser (`_plan_with_mock`)
that extracts an order id (`#9921`) and an email address from the text.

This matters because LLMs are unreliable in two ways that matter for an
agent that takes real actions:
- **Availability/format unreliability** — the API can be down, rate
  limited, or occasionally return malformed JSON. The `try/except`
  around the OpenAI call means a single bad LLM response degrades
  gracefully to the mock planner instead of taking the whole request down.
- **Semantic unreliability** — even a well-formed response could
  reference a tool that doesn't exist, or omit a required argument.
  This is why *every* plan (LLM or mock) passes through
  `_validate_plan`, which checks each step's tool against a hard
  whitelist (`TOOL_REGISTRY`) and each tool's required arguments
  (`REQUIRED_ARGS`) before anything is allowed to execute. A plan that
  references an unknown tool or is missing an argument is rejected at
  the API boundary (`422 Unprocessable Entity`), never silently retried
  against a stale/partial understanding of the request.

This is the same reason the assignment says not to reach for LangChain:
the actual "hard part" of agent reliability is this validation boundary
between untrusted model output and code that takes real actions — it's
about 40 lines of plain Python here, not a framework.

### 2. Async execution
Both tools (`cancel_order`, `send_email`) are `async def` functions using
`asyncio.sleep` to simulate I/O latency, matching the assignment spec
(random 20% failure for cancellation, 1-second simulated send for email).

The orchestrator (`app/orchestrator.py`) `await`s each step in plan
order. Steps aren't blindly run in parallel — `send_email` is declared
in the plan as `depends_on` the cancellation step, and the orchestrator
respects that: it will not call `send_email` until `cancel_order` has
returned, and will skip it entirely if cancellation failed.

The API layer is where the "event-driven" part lives: `POST /process`
does not block on the whole job. It plans the request synchronously
(fast, no I/O), then fires the orchestrator off with
`asyncio.create_task(...)` and returns a `job_id` immediately. The job
continues running concurrently on the event loop, writing its progress
into the job store after every single step. A client (or a future
webhook/SSE layer) can observe that progress at any time via
`GET /jobs/{id}` rather than waiting on a single long HTTP call. (A
`?sync=true` flag is provided too, purely for one-shot demo/testing
convenience — it just awaits the same `run_job` coroutine inline.)

### 3. State management
State lives in `app/job_store.py`: an `asyncio.Lock`-guarded in-memory
dict, keyed by `job_id`, storing a `JobRecord` (request text, plan,
per-step results, overall status, final message). It's intentionally
the simplest thing that could work for a take-home-scale service:
- Reads/writes are async-safe via the lock, so concurrent requests and
  the background orchestrator task can't interleave a torn write.
- `get()` returns a deep copy, so callers never accidentally mutate
  shared state out from under a running job.
- Swapping this for Redis or Postgres later is a one-file change —
  nothing in `main.py` or `orchestrator.py` knows or cares that the
  store is in-memory.

### 4. Guardrails
Two independent guardrail layers, both exercised by the automated tests
in `tests/test_orchestrator.py`:
- **Plan validation** (before execution): unknown tools and missing
  arguments are rejected up front (`_validate_plan`).
- **Execution-time dependency guarding** (during execution): the
  orchestrator tracks which step ids have failed. Any subsequent step
  whose `depends_on` points at a failed step is **skipped**, not run,
  and recorded with a `skipped_reason`. This is what guarantees
  `send_email` never fires after a failed `cancel_order` — per the
  assignment's core requirement — and the job's `final_message` clearly
  states which step failed and why.

## Project layout
```
app/
  main.py          FastAPI routes
  models.py         Pydantic models: request, Plan/PlanStep, JobRecord/StepResult
  planner.py        LLM + mock-fallback planning, plan validation guardrail
  tools.py          async cancel_order / send_email + the tool whitelist
  orchestrator.py    executes a Plan, dependency-aware, updates job store live
  job_store.py       tiny async-safe in-memory store
tests/
  test_orchestrator.py   planner extraction + guardrail behavior (pass/fail paths)
```

## Known limitations (by design, for this scope)
- State is in-memory and process-local — restarting the server loses
  jobs. A production version would move `job_store` to Redis/Postgres.
- No auth on the endpoints.
- Only two tools are implemented, per the assignment; adding a new one
  is: write the async function, add it to `TOOL_REGISTRY` and
  `REQUIRED_ARGS`, mention it in the planner's system prompt.
