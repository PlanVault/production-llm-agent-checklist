# Production LLM Agent Security & Runtime Checklist

> A practical reference for engineering teams moving LLM agents from demos to production. This document covers typical failure modes and deterministic controls required when AI agents have access to internal tools, APIs, databases, workflow systems, or customer-visible actions.
>
> **Core Principle:** An LLM should never be the sole decision-maker and execution engine. The model plans or interprets; a deterministic runtime validates, limits, executes, audits, and can deny.
>
> This checklist covers runtime security for production agents. If you are looking for IDE and team development guidelines, check out our [AI-Assisted Engineering Playbook](https://github.com/PlanVault/ai-engineering-playbook).

---

## TL;DR: The 3 Pillars of Agent Security

**1. Data & Context Boundaries**
* **Minimize & Redact:** Never send raw secrets, PII, or internal UUIDs to the LLM. Map them to safe aliases before the boundary.
* **Flatten Structures:** Deep JSON trees confuse models. Pass simple, flattened DTOs.
* **Treat Data as Untrusted:** RAG outputs and web searches are untrusted inputs, not system instructions.

**2. Deterministic Execution**
* **The LLM Proposes, the Runtime Disposes:** The model plans an action; a deterministic layer validates policy, tenant scope, and limits *before* execution.
* **Tool Risk Classes:** Not all tools are equal. Read-only tools pass; high-risk mutations require Human-in-the-Loop (HITL) approval.
* **Idempotency is Mandatory:** Never retry a high-risk mutation without an idempotency key.

**3. Budget & Loop Guardrails**
* **Cap Autonomous Behavior:** Set hard limits on RAG loops, tool retries, and budget exhaustion. Prevent infinite API loops.
* **Audit Everything:** Replayability requires logging prompt versions, tool choices, and parameter hashes (not just the raw text).

---

## Phase 1: Foundation — Build Before You Wire

> Set up the infrastructure, lock your model stack, and establish the core architectural principle before any AI logic is written.

## 1. AI Infrastructure First (Architecture Philosophy)

Before writing a single line of AI logic, the underlying infrastructure must be production-ready. Agents built on fragile foundations fail unpredictably and are impossible to debug.

### Failure Modes
* Teams build AI agent logic on top of fragile, undocumented internal tooling.
* Knowledge base is maintained by engineers only and is unreadable to the model.
* Agents query the database directly, bypassing all business-logic validation layers.
* Tools are built "for AI" from scratch instead of wrapping proven, tested APIs.

### Controls
* Treat the knowledge base as a first-class artifact: it must be readable and maintainable by both humans and AI. If a rule is ambiguous to a human reader, it will be unreliable for the model.
* Prefer injecting rules and context directly into the prompt alongside the user request rather than relying solely on MCP calls for context retrieval. MCP is best suited for tool invocation, not for delivering structured policy context.
* Build tools as standard REST APIs first, then wrap them as AI tools or expose via MCP. This gives you versioning, testability, rate limiting, and observability for free.
* Never give the agent direct database access. All data mutations and queries must go through existing service layers that enforce business rules, authorization, and audit hooks.
* Establish monitoring, integration tests, and e2e coverage before wiring up the AI layer.

## 2. Model Provider Risk Management

Once the infrastructure is ready, lock down your model dependencies before any product logic is written. A silent model update or provider outage can break everything downstream.

### Failure Modes
* The LLM provider updates the base model version silently, breaking established behavior.
* A provider outage causes the agent to fail with no deterministic fallback.
* Prompts sent to the LLM provider are logged and retained under unclear data policies.

### Controls
* Pin to specific model versions in production. Treat a model upgrade as a code change: test before promoting.
* Define and document a fallback strategy for provider unavailability (deterministic fallback, graceful degradation to human handoff, or secondary provider).
* Review and contractually confirm the provider's data retention and zero-data-retention (ZDR) policies before sending customer data to any LLM API.
* Run automated behavioral regression tests whenever you intentionally upgrade the pinned model version.

## 3. Planner / Executor Separation

The single most important architectural decision: the LLM must never directly execute actions. It proposes; a deterministic runtime decides, validates, and acts.

### Failure Modes
* The model directly mutates state or forms HTTP requests without a deterministic validation layer in between.
* The final answer claims an action was completed, even if the API call failed.

### Controls
* **The LLM proposes; the runtime executes.**
* The executor validates, authorizes, applies budgets, requests HITL if needed, and applies the state transition.
* The final user response must be based on the committed runtime state, not on the LLM's "plan."

---

## Phase 2: What Goes Into the Model

> Control volume, sensitivity, structure, and source of every piece of context before it crosses the LLM boundary.

## 4. Context Minimization

With the architecture in place, control what enters the model. Noise in the context window costs money and degrades reasoning quality.

### Failure Modes
* The model receives a massive HTTP/DB/tool response and loses focus in the noise.
* A critical instruction is buried in the middle of a long prompt and loses attention.
* The full chat history bloats the context window, increasing costs and degrading reasoning.
* A single, bloated prompt forces the model to simultaneously plan, extract data, decide policy, and draft a final response.

### Controls
* Pass only the minimum viable context required for the specific step.
* Use schema-bound extraction or concise summaries instead of raw responses.
* Truncate, aggregate, or paginate large arrays; explicitly indicate to the model that the data is partial.
* Place the primary task or question at the beginning or very end of the prompt.
* Use rolling summaries for conversation history; keep the full transcript in deterministic storage only.

## 5. Data Minimization, Redaction, and Alias Mapping

Beyond reducing volume, scrub everything sensitive before it reaches the model boundary.

### Failure Modes
* The LLM sees secrets, access tokens, PII, or customer-sensitive fields.
* The model leaks internal UUIDs, synthetic IDs, or implementation details into user-facing responses.
* The agent makes a tool call using an ID that belongs to a different tenant/project.

### Controls
* **Never** pass secrets into the prompt.
* Redact PII and sensitive fields at the LLM boundary.
* Replace UUIDs/internal IDs with short, stable aliases (e.g., `item_1`, `account_2`).
* Perform reverse mapping (`alias -> real ID`) strictly within the deterministic executor *after* the LLM response.
* Validate that the mapped entity actually belongs to the current `org_id` / `project_id` during execution.

## 6. Structure Simplification

Flat, task-specific data structures consistently outperform raw domain objects for tool-choice accuracy and parameter correctness.

### Failure Modes
* Deeply nested JSON degrades tool choice accuracy and parameter hallucination.
* The LLM confuses fields with identical names in different nested branches.
* The model misses constraints hidden 5–7 levels deep in an object.

### Controls
* Flatten nested models before crossing the LLM boundary.
* Provide task-specific DTOs instead of full domain objects.
* Lift critical constraints into top-level fields or a dedicated `constraints` block.
* Do not mix source data, security policy, tool schemas, and user-facing explanations in a single blob.

## 7. Context Provenance

Label every piece of context by its source. Without provenance, debugging model behavior and tracing decisions back to their origin is impossible.

### Failure Modes
* The model cannot distinguish between user input, retrieved docs, tool results, and system policy.
* Debugging is impossible because it's unclear where the LLM got a specific "fact".

### Controls
* Mark context blocks by source (e.g., user message, runtime policy, retrieved document, tool result).
* Retain document IDs, version/freshness, and confidence scores for retrieved data.
* The audit trail must be able to answer: *"Which specific context block influenced this decision?"*

---

## Phase 3: Tools and Execution

> Classify every tool by risk, select only what's needed, route to the right model, and validate before anything is executed.

## 8. Tool Risk Classification

Before the agent can call anything, every tool must carry a risk class. This is the foundation for all downstream enforcement — routing, validation, HITL, and audit.

### Failure Modes
* The agent treats a destructive database mutation the same as a read-only lookup.
* Security reviews block AI releases because there is no formal boundary between safe and risky actions.

### Controls
Every tool in the catalog must have an assigned risk class:

| Class | Example | Runtime Policy |
|---|---|---|
| **Read-only** | Search, list, get status | Allow (with tenant-scope validation). |
| **Low-risk mutation** | Create draft, save preference | Allow, or require lightweight confirmation. |
| **High-risk mutation** | Update production record | Require policy validation, possible HITL. |
| **Destructive** | Delete, revoke, wipe | Require HITL, dry-run/preview, strong audit. |
| **External effect** | Send email, webhook, API call | Require idempotency and visibility controls. |
| **Money / Legal** | Payment, contract execution | Default HITL, strict policy, explicit confirmation. |

## 9. Dynamic Tool Routing

Never expose the full tool catalog to the model. First select the right tools for the intent, then plan — not the other way around.

### Failure Modes
* Adding too many tools to the prompt so the "model can choose," causing decision paralysis.
* Recursive tool discovery creates long, expensive loops.

### Controls
* Do not pass the entire global tool catalog into every LLM turn.
* Use dynamic routing (e.g., vector + BM25 fusion) to narrow down a ranked short list of candidate tools based on intent.
* Provide concise semantic descriptions and risk summaries for candidate tools.
* In practice, more than ~50 tools in a single context causes measurable degradation in tool-choice accuracy, even in large models. The effective limit depends on description length, parameter count, and model family.
* Preferred approach: first select only the tools relevant to the user's intent, then generate a plan using that constrained tool set (two-pass: tool selection → plan).
* Tool description quality matters more than count. Five well-described tools outperform twenty ambiguous ones.

## 10. Model Routing and Task Splitting

Route each subtask to the right model tier. Overusing a heavyweight model for trivial steps is the most common source of unnecessary cost.

### Failure Modes
* An expensive reasoning model is wasted on cheap data extraction or formatting.
* A small utility model is assigned a planning task requiring complex reasoning.
* Cost spikes due to a universal "do everything" prompt architecture.

### Controls
* Route planning, hard reasoning, and ambiguous decisions to heavy models.
* Route extraction, summarization, classification, and context compression to fast/cheap models.
* Decompose the pipeline into short deterministic or model-assisted steps.

## 11. Pre-Execution Validation

The model has proposed an action and the right tools are selected. Now the runtime must independently verify the action is safe before anything is executed.

### Failure Modes
* The LLM outputs a perfectly valid JSON schema, but the requested business action is unsafe or illogical (e.g., transferring a negative amount).
* Cross-tenant data access slips through hallucinated tool parameters.

### Controls
* Validate business invariants (ranges, enum policies, limits, allowed state transitions) *before* execution.
* Validate tenant/project scope for every entity reference.
* Perform dry-runs or previews for risky actions before committing the mutation.
* The deterministic executor must retain the right to reject an action, even if it is syntactically valid.
* **Guard against concurrent mutations.** When multiple agents or parallel user sessions can mutate the same record simultaneously, use optimistic locking: read the current version/etag of the entity before proposing an action and reject the execution if the version has changed by the time the write is committed. This prevents lost-update bugs that are invisible to schema validation but produce corrupt state.

---

## Phase 4: Defence Against Attacks

> External content — tool outputs, documents, user input — is an attack surface. Treat all of it as untrusted.

## 12. Prompt Injection Boundaries

External content — tool outputs, retrieved documents, user messages — must never be treated as instructions by the runtime.

### Failure Modes
* A tool output says "ignore previous instructions" and the model obeys.
* A retrieved webpage attempts to exfiltrate secrets via an API call.

### Controls
* System, developer, and runtime policies must never be presented as part of the editable context.
* Box or quote documents containing instructions as raw data.
* Because the LLM never has access to plaintext secrets (see Section 5), injections cannot exfiltrate them.
* Execution must not trust LLM claims; it must verify policy independently.

## 13. RAG and Knowledge Base Controls

Retrieved knowledge is an attack surface. Apply the same distrust to KB content as to any user input.

### Failure Modes
* The agent recursively calls the Knowledge Base in a loop because it is "unsure," burning tokens.
* Retrieved content contains prompt injections or explicit instructions that hijack the agent.
* The model confuses untrusted document text with system/developer instructions.

### Controls
* Set hard limits for dynamic retrieval: max calls, max chunks, max tokens.
* **Treat retrieved documents as untrusted data, not instructions.**
* Isolate instruction-like content found in documents (e.g., ignoring "ignore previous instructions").
* If RAG finds no evidence, the agent must fail gracefully or ask for clarification—not enter a recursion loop.
* RAG recursion is the most common cause of uncontrolled budget consumption in production. Hard per-session and per-project token budgets (not just per-run) are required.
* For large knowledge bases, use a two-stage retrieval: retrieve candidates, compress/rank, then pass only the top-k chunks to the model. Never pass the full KB.
* Track RAG token costs separately in observability to identify which retrieval patterns are most expensive.
* **Enforce tenant isolation at the vector DB level.** Apply a hard `tenant_id` metadata filter on every retrieval query before results are returned. Do not rely solely on post-retrieval filtering in application code — a missing or incorrect filter at the DB level is sufficient for cross-tenant data leakage, even before results reach the LLM context.

---

## Phase 5: Human Oversight and Accountability

> High-risk actions require human sign-off. That sign-off is only meaningful if the reasoning behind it is visible and auditable.

## 14. HITL (Human-in-the-Loop) Approval Gates

High-risk actions must pause and require explicit human sign-off. The approval must be cryptographically bound to the exact payload — not a general "yes."

### Failure Modes
* The LLM executes a high-impact action autonomously.
* The user approves an action without understanding the exact payload or side effects.
* An approval is disconnected from the payload hash and reused maliciously.

### Controls
* Require HITL for high-risk, destructive, money/legal, and customer-visible actions.
* The approval screen must show: action name, risk class, exact parameters (after alias mapping), affected entities, and policy reason.
* Bind the approval cryptographically to the action hash, actor, session, and timestamp.
* Any parameter change by the LLM invalidates the previous approval.

## 15. Decision Explainability for Variable Business Logic

When the model applies fuzzy or variable rules — pricing, scoring, routing — it must articulate its reasoning. A human approving an action they cannot explain is not meaningful oversight.

### Failure Modes
* The agent applies a discount, penalty, or priority score with no auditable reasoning.
* A human reviewer approves a HITL action without understanding why the model chose that outcome.
* Fuzzy business rules (pricing, scoring, routing) produce inconsistent outputs with no traceability.

### Controls
* For any functionality with variable or fuzzy logic (discounts, recommendations, scoring, priority), require the model to provide explicit chain-of-thought reasoning: which rule was triggered, which limit was checked, and why this specific outcome was selected.
* Store this reasoning in the audit log alongside the decision and the parameters used.
* At HITL approval screens (see Section 14), surface the model's stated reasoning — not just the proposed action — so the human reviewer can validate the logic, not only the output.
* If the model cannot provide a clear, rule-referenced justification for a critical decision, block execution.
* Define for each variable-logic tool: the enumerated list of valid reasons, the applicable limits, and the required output fields that capture the rationale.

---

## Phase 6: Resilience and Fault Tolerance

> Prevent runaway loops, guarantee mutations happen exactly once, survive crashes, handle async long-running tasks, and enforce trust isolation in multi-agent systems.

## 16. Retry, Recursion, and Budget Guardrails

The agent must not be able to run itself into infinite loops. Hard caps protect both reliability and cost.

### Failure Modes
* The LLM attempts to self-correct an API error in an infinite loop.
* The agent repeats a high-risk mutation after a timeout, creating duplicate side effects.

### Controls
* Enforce hard limits per run: max tool calls, max retries per step, total retries, and wall-clock time.
* Implement circuit breakers for repeated same-error patterns.
* Only retry classified *transient* failures (e.g., 429, 503).
* Never retry high-risk mutations without an idempotency key.
* Circuit breakers must apply at the LLM provider level, not only at the tool level. If the model provider returns repeated errors, fall back to a degraded mode rather than looping.
* **Leverage prompt caching for multi-step pipelines.** In pipelines where the system prompt, large tool schemas, or a static knowledge block are repeated across many turns, enable provider-level prompt caching (e.g., Anthropic cache-control breakpoints, OpenAI cached prefix). This is the primary lever for reducing both latency and token cost in long-horizon agentic runs — budget savings compound with every additional step.

## 17. Idempotency and Replay Safety

Network failures are inevitable. Every high-risk mutation must be safe to observe more than once without producing duplicate effects.

### Failure Modes
* A network timeout after a successful external mutation causes the agent to retry, resulting in duplicate orders/payments.
* Replaying session logs executes external side effects a second time.

### Controls
* Require an Idempotency Key per planned external action.
* Store the action intent, parameter hash, execution status, and external response.
* Journal replays must never re-execute side effects.
* Detect duplicate action attempts at the HTTP client level and return the cached previous result.

## 18. State Management and Fault Tolerance

Long-running agents will fail mid-run. Checkpointing and safe stopping points ensure failures are recoverable, not catastrophic.

### Failure Modes
* A long-running agent (20+ steps) crashes at step 15 with no recovery point.
* Agent state is lost between sessions, causing the agent to restart from scratch on reconnect.
* The system leaves partially applied mutations when the executor fails mid-run.

### Controls
* Checkpoint agent state at each completed step. On failure, resume from the last committed checkpoint, not from the beginning.
* Define explicit "safe stopping points" in multi-step workflows. The agent must be able to halt at a safe state, not in the middle of a mutation sequence.
* Persist intent and execution status across sessions (see Section 17 on Idempotency) so restarts are replay-safe.
* Distinguish between executor failures and model failures; recovery strategy may differ between them.

## 19. Async and Long-Running Task Patterns

Agents frequently trigger tools or workflows — external jobs, data pipelines, third-party APIs — that take minutes or hours to complete. Synchronous blocking is unreliable at this timescale; the architecture must support durable suspension and async completion.

### Failure Modes
* The agent blocks synchronously waiting for a tool that takes minutes, burning connection resources and eventually timing out.
* A long-running job completes but the result is never delivered because the original request context was lost on restart.
* The agent polls a status endpoint in a tight loop, burning tokens and API quota unnecessarily.
* A webhook fires but there is no mechanism to correlate it back to the paused agent run.

### Controls
* For any tool that may take longer than a few seconds, return a **job handle** immediately and suspend the agent run — do not block.
* Support two async completion strategies and choose based on the tool:
  * **Polling:** the runtime checks job status at configurable intervals with exponential backoff and jitter.
  * **Webhooks/callbacks:** the external system pushes the result to a durable agent endpoint that resumes the suspended run.
* Persist the suspended agent state and job handle at a checkpoint (see Section 18) so the run survives process restarts between submission and completion.
* Enforce a **maximum wait budget per async step**: if a job does not complete within the configured deadline, escalate to a human or fail gracefully — never wait indefinitely.
* Treat excessive polling as a budget violation: apply the same circuit-breaker and retry limits as tool calls (see Section 16).
* On webhook receipt, verify the payload signature and correlate it to the exact suspended run and idempotency key (see Section 17) before resuming execution.

## 20. Multi-Agent Trust Boundaries

When agents orchestrate other agents, every trust and policy assumption must be re-established at each boundary — scope does not flow downward automatically.

### Failure Modes
* A subagent inherits the full permissions of its orchestrating parent without explicit scope reduction.
* Agent-to-agent calls bypass the same policy enforcement applied to user-initiated calls.
* Prompt injection in an orchestrator's output propagates instructions into a subagent.

### Controls
* Every agent in a hierarchy must have its own identity and scope, independent of its parent.
* Agent-to-agent calls must pass through the same pre-execution validation, tenant-scope checks, and tool risk classification as user-initiated calls.
* Subagents must not trust instruction-like content received from an orchestrator without policy verification.
* Apply budget isolation per agent level: a subagent's token and tool-call budget must be capped independently of the parent's budget.

---

## Phase 7: Operations and Continuous Improvement

> Validate what comes out, instrument everything, test continuously, and ship safely with staged rollouts.

## 21. Output Validation and Harm Prevention

Validate everything that comes out of the model before it reaches any downstream system or user interface.

### Failure Modes
* The LLM returns malformed JSON; the system crashes or silently uses a partial result.
* A model output contains XSS payloads or injection content that is rendered in a user-facing UI.
* The agent produces factual hallucinations in a high-stakes domain (legal, medical, financial) with no detection.

### Controls
* Validate all structured LLM outputs against a schema before use. Define a retry-once policy for malformed outputs; escalate to a failure state if retries do not produce a valid structure.
* Sanitize all model-generated text before rendering in any UI context.
* For high-stakes domains, apply output moderation (content safety classifiers) as a post-generation step before the response is delivered.
* Log all output validation failures — they are a signal for prompt regression or model drift.

## 22. Observability and Auditability

With all controls in place, you need full visibility into every run. Without structured logs and tracing, production incidents are impossible to diagnose.

### Failure Modes
* Security teams cannot see who approved a risky AI action.
* Sudden token/cost spikes have no identifiable root cause.

### Controls
For every run/step, log (stripping all PII and secrets):
* Prompt template/version and model provider.
* Selected tools and filtered candidates.
* Tool parameter hashes (or safe redacted parameters).
* Validation decisions and approval requests/outcomes.
* Execution status, errors, and retry reasons.
* Token usage, latency, and explicit stop reasons.
* Distributed tracing with correlation IDs across all steps and agents in a run.
* SLO tracking: P95 latency, availability, and error rate per agent and per tool.
* Automated alerting on anomalous patterns: sudden burst of tool calls, unexpected token spike, repeated HITL triggers.
* Prompt token count as an explicit metric per turn.

## 23. Evaluation and Regression Testing

Observability tells you what happened. Evaluation tells you whether the system is still behaving correctly — across prompts, models, and providers.

### Failure Modes
* Changing a prompt or model provider silently breaks tool choice accuracy.
* Cost regressions go unnoticed until the billing cycle ends.

### Controls
* Maintain golden test cases for common tasks.
* Run adversarial prompts to test injection resistance and policy bypasses.
* Use fixtures for malformed tool responses to test error handling.
* Monitor budget regressions (tokens, tool calls, latency) in CI/CD.
* Run the same test suite across multiple model providers/versions to establish behavioral equivalence before switching.
* Schedule e2e test runs in production on a cron. Model provider updates happen without notice; continuous automated checks are the only reliable detection mechanism.
* Define statistical significance criteria for eval results — a single golden test is not sufficient for release confidence.

## 24. Deployment Readiness

The final gate before going live: staged rollouts, automated release gates, and behavioral baselines ensure production changes do not reach all users before they are verified.

### Failure Modes
* A prompt or model change is deployed to 100% of users without any staged rollout.
* The LLM provider silently updates the base model; no automated check catches the behavioral regression.
* There is no automated validation gate before a release reaches production.

### Controls
* Require passing e2e tests covering core business scenarios as a hard release gate. Tests will not catch everything, but critical user flows must be covered automatically.
* Run e2e test suites on a scheduled cron (e.g., daily) in production, not only in CI. LLM providers update base models without notice; behavioral regressions must be caught continuously, not just at deploy time.
* After any prompt, model, or tool change, deploy first to a small percentage of traffic (canary). Monitor quality metrics before applying to all users.
* Track behavioral baselines: snapshot how the system behaves today so regressions are measurable, not subjective.

---

## The Missing Layer: PlanVault™

This checklist is maintained by the engineering team at **[PlanVault™](https://planvault.ai/?utm_source=github&utm_medium=readme&utm_campaign=production-llm-agent-checklist&utm_content=next-step)**.

While you can build these guardrails from scratch, maintaining state machines, alias mapping, and cryptographic HITL approvals takes months of engineering effort. **PlanVault™ is an event-sourced execution layer** that handles all of this out-of-the-box.

👉 **Next Step:** If your team needs development-time guardrails, check out our [AI-Assisted Engineering Playbook](https://github.com/PlanVault/ai-engineering-playbook). If you are moving agents to production and need to enforce these controls without slowing down development, [let's schedule a 15-min Architecture Sync](https://calendly.com/admin-planvault/15min).

---

**License:** This checklist is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You are free to share and adapt it for your internal engineering guidelines.
