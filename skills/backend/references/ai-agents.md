# Implementing AI agents and LLM features

The architect decides whether a feature needs an LLM, which provider and models, and the budget (ADR). The backend implements it behind ports so the product logic doesn't depend on a vendor SDK. Check the provider's current docs, SDK versions and model ids before building — they change often; never hard-code model ids (config / env).

## 1. Choose the simplest shape that works

| Shape | When |
|---|---|
| Single LLM call with structured output | classification, extraction, summarization, drafting |
| Fixed workflow (prompt chain, routing, parallel calls, evaluator-optimizer) | known steps, predictable cost and latency |
| Agent (LLM chooses tools in a loop) | open-ended tasks where the steps can't be known in advance |
| Multi-agent (orchestrator + workers) | genuinely separable subtasks that benefit from parallel or isolated context; costs more |

Start with a workflow; move to an agent only when evals show the workflow can't do the job.

## 2. Architecture (hexagonal)

```
application/
  ports/llm.py          # LlmClient: complete(messages, tools, schema, ...) -> LlmResult
  ports/tools.py        # Tool protocol: name, description, input schema, execute(ctx, input)
  agents/<name>.py      # agent loop / workflow as a use case: prompt, tools, limits, policies
adapters/outbound/llm/  # Anthropic / other provider adapters, retries, tracing, cost accounting
adapters/inbound/mcp/   # optional: expose domain use cases as an MCP server
adapters/outbound/mcp/  # optional: consume external MCP servers as tools
```

- **Tools are use cases**: each tool calls an application use case (never the DB directly), with the **end user's identity and permissions** — the agent can never do more than the user could.
- Tool definitions: clear names and descriptions written for the model, JSON Schema inputs (generated from Pydantic / records), strict validation of tool inputs, concise results (ids, summaries; paginate large results), errors returned as tool results the model can act on.
- Prompts versioned in the repo (templates with variables), not scattered strings; system prompt states role, constraints, output format and tool-use guidance.
- Frameworks: the provider SDK directly (Anthropic Python / Java SDK, tool runner), the Claude Agent SDK for agents that need a full harness (files, shell, subagents, MCP), MCP Python / Java SDKs for servers and clients; LangGraph, Pydantic AI or Spring AI when the project adopts them (ADR). Keep the port so swapping is possible.

## 3. The agent loop

- Loop: send messages + tools → if the model requests tools, validate and execute them (in parallel when independent), append results → repeat until a final answer or a limit.
- Limits: max iterations, max tokens per run, wall-clock timeout, max cost per run and per user / tenant per day; stop with a clear partial result when hit.
- **Human in the loop** for tools that change data, send messages or spend money (per the requirements): the agent proposes, the user confirms (or a policy auto-approves within limits). Model this as a pending action resource with its own API.
- State: conversation and run state persisted (Postgres: `agent_runs`, `agent_messages`, `agent_tool_calls`) for resumption, audit and evals; large context trimmed or summarized deliberately (context editing / compaction where the provider supports it).
- Streaming to the frontend via SSE (token and tool-progress events) when the UX needs it; the architect and frontend agree the event schema (AsyncAPI or an SSE section in the contract).
- Long runs as background jobs with an operation resource (`202` + status), not inside a request.

## 4. Structured output and determinism

- Use the provider's structured output / JSON schema features or a forced tool call; validate the result with Pydantic / Bean Validation; on failure retry once with the validation error, then fail clearly.
- Low temperature for extraction and decisions; keep business rules (prices, availability, permissions) in code — the model never computes what the domain can compute.

## 5. Retrieval (RAG)

- pgvector in the service's Postgres by default (HNSW index, cosine), a dedicated vector store only by ADR.
- Chunking by document structure (headings, paragraphs) with overlap; store source, version, tenant and permissions per chunk; filter by permissions **in the query**.
- Hybrid retrieval (vector + full-text `tsvector` / BM25) and reranking when quality needs it; cite sources in answers.
- Embedding model and dimension recorded per row so re-embedding is possible; ingestion as idempotent jobs.

## 6. Safety and security

- **Prompt injection**: treat all retrieved content, tool results, user files and web pages as untrusted data; never let it change tools, permissions or instructions; delimit it clearly; least-privilege tools; confirmation for consequential actions; no secrets in prompts.
- Output handling: never execute model output as code / SQL / shell without a sandbox and allowlist; sanitize before rendering (the frontend renders as text or sanitized markdown).
- PII: minimize what's sent to the provider, respect data-processing agreements and regions (GDPR), redact in logs and traces, retention for stored conversations per the requirements.
- Abuse and cost: per-user rate limits and quotas, input size limits, moderation where the product needs it.

## 7. Reliability, cost and latency

- Timeouts, retries with backoff on 429 / 5xx / overloaded (respect `retry-after`), fallback model or graceful "try later" message.
- Prompt caching for stable prefixes (system prompt, tool definitions, documents); batch APIs for offline bulk work; the smallest model that passes evals per step (route easy steps to fast models).
- Track tokens and cost per run, feature, tenant; alert on budget burn.

## 8. Evals and observability

- Eval set per feature in the repo (`evals/<feature>/`): representative inputs with expected outputs or grading rubrics, including adversarial and injection cases; run in CI on prompt / model / tool changes with thresholds agreed with the architect and qa.
- Graders: exact / schema checks first, code-based checks, LLM-as-judge with a rubric only where needed (calibrated against human labels).
- Tracing with OpenTelemetry GenAI semantic conventions (model, tokens, latency, tool calls, finish reason) linked to the request's trace; store run transcripts for debugging with PII redaction.
- Online metrics: task success, user feedback, escalation / fallback rate, cost per successful task.
- Tests: unit-test tools and the loop with a fake `LlmClient` (scripted responses), contract-test the adapter against recorded responses, evals for quality.

## 9. Review checklist

- Workflow vs agent justified; limits (iterations, tokens, time, cost) set.
- Tools call use cases with the user's permissions; inputs validated; consequential actions confirmed.
- Untrusted content handled as data; no secrets or unnecessary PII in prompts; outputs sanitized.
- Structured outputs validated; business rules in code.
- Model ids and prompts from config / versioned files; prompt caching where it pays.
- Timeouts, retries, fallbacks; cost tracked; tracing on.
- Evals exist, run in CI and pass thresholds.
