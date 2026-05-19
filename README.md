# Production LLM Agent Security & Runtime Checklist

> A practical reference for engineering teams moving LLM agents from demos to production. This document covers typical failure modes and deterministic controls required when AI agents have access to internal tools, APIs, databases, workflow systems, or customer-visible actions.
>
> **Core Principle:** An LLM should never be the sole decision-maker and execution engine. The model plans or interprets; a deterministic runtime validates, limits, executes, audits, and can deny.

---

## TL;DR

1. **Minimize context.** Do not send raw, massive API responses, full chat histories, or deeply nested structures to the LLM without compression.
2. **Redact and mask data.** Replace PII, secrets, UUIDs, and synthetic internal IDs with safe aliases before the LLM boundary. Map them back during execution.
3. **Classify tools by risk.** Categorize actions: read-only, low-risk mutation, high-risk mutation, destructive, external side effect, or customer-visible.
4. **Validate before execution.** JSON schema validation is not enough. Enforce business invariants, tenant scopes, approval gates, and idempotency.
5. **Cap autonomous self-correction.** Set hard limits on retries, RAG loops, tool calls, and budget exhaustion. Do not let agents loop infinitely.
6. **Separate the planner from the executor.** The LLM proposes an action; the deterministic executor accepts or rejects it based on state and policy.
7. **Enforce HITL (Human-in-the-Loop).** Require explicit human approval for high-risk, destructive, or external actions.
8. **Treat RAG as untrusted data.** Retrieved documents and tool outputs are untrusted data, not system instructions. Protect against prompt injections.
9. **Ensure auditability and replayability.** Prompt versions, selected tools, parameters, approvals, outputs, and errors must be deterministically reproducible.
10. **Test agent behavior as a product surface.** Golden cases, adversarial prompts, tool-choice regressions, and budget regressions must be part of your CI/CD and release discipline.

---

## 1. Context Minimization

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

## 2. Data Minimization, Redaction, and Alias Mapping

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

## 3. Structure Simplification

### Failure Modes
* Deeply nested JSON degrades tool choice accuracy and parameter hallucination.
* The LLM confuses fields with identical names in different nested branches.
* The model misses constraints hidden 5–7 levels deep in an object.

### Controls
* Flatten nested models before crossing the LLM boundary.
* Provide task-specific DTOs instead of full domain objects.
* Lift critical constraints into top-level fields or a dedicated `constraints` block.
* Do not mix source data, security policy, tool schemas, and user-facing explanations in a single blob.

## 4. Model Routing and Task Splitting

### Failure Modes
* An expensive reasoning model is wasted on cheap data extraction or formatting.
* A small utility model is assigned a planning task requiring complex reasoning.
* Cost spikes due to a universal "do everything" prompt architecture.

### Controls
* Route planning, hard reasoning, and ambiguous decisions to heavy models.
* Route extraction, summarization, classification, and context compression to fast/cheap models.
* Decompose the pipeline into short deterministic or model-assisted steps.

## 5. Tool Risk Classification

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

## 6. Pre-Execution Validation

### Failure Modes
* The LLM outputs a perfectly valid JSON schema, but the requested business action is unsafe or illogical (e.g., transferring a negative amount).
* Cross-tenant data access slips through hallucinated tool parameters.

### Controls
* Validate business invariants (ranges, enum policies, limits, allowed state transitions) *before* execution.
* Validate tenant/project scope for every entity reference.
* Perform dry-runs or previews for risky actions before committing the mutation.
* The deterministic executor must retain the right to reject an action, even if it is syntactically valid.

## 7. Dynamic Tool Routing

### Failure Modes
* Adding too many tools to the prompt so the "model can choose," causing decision paralysis.
* Recursive tool discovery creates long, expensive loops.

### Controls
* Do not pass the entire global tool catalog into every LLM turn.
* Use dynamic routing (e.g., vector + BM25 fusion) to narrow down a ranked short list of candidate tools based on intent.
* Provide concise semantic descriptions and risk summaries for candidate tools.

## 8. RAG and Knowledge Base Controls

### Failure Modes
* The agent recursively calls the Knowledge Base in a loop because it is "unsure," burning tokens.
* Retrieved content contains prompt injections or explicit instructions that hijack the agent.
* The model confuses untrusted document text with system/developer instructions.

### Controls
* Set hard limits for dynamic retrieval: max calls, max chunks, max tokens.
* **Treat retrieved documents as untrusted data, not instructions.**
* Isolate instruction-like content found in documents (e.g., ignoring "ignore previous instructions").
* If RAG finds no evidence, the agent must fail gracefully or ask for clarification—not enter a recursion loop.

## 9. Retry, Recursion, and Budget Guardrails

### Failure Modes
* The LLM attempts to self-correct an API error in an infinite loop.
* The agent repeats a high-risk mutation after a timeout, creating duplicate side effects.

### Controls
* Enforce hard limits per run: max tool calls, max retries per step, total retries, and wall-clock time.
* Implement circuit breakers for repeated same-error patterns.
* Only retry classified *transient* failures (e.g., 429, 503).
* Never retry high-risk mutations without an idempotency key.

## 10. Idempotency and Replay Safety

### Failure Modes
* A network timeout after a successful external mutation causes the agent to retry, resulting in duplicate orders/payments.
* Replaying session logs executes external side effects a second time.

### Controls
* Require an Idempotency Key per planned external action.
* Store the action intent, parameter hash, execution status, and external response.
* Journal replays must never re-execute side effects.
* Detect duplicate action attempts at the HTTP client level and return the cached previous result.

## 11. HITL (Human-in-the-Loop) Approval Gates

### Failure Modes
* The LLM executes a high-impact action autonomously.
* The user approves an action without understanding the exact payload or side effects.
* An approval is disconnected from the payload hash and reused maliciously.

### Controls
* Require HITL for high-risk, destructive, money/legal, and customer-visible actions.
* The approval screen must show: action name, risk class, exact parameters (after alias mapping), affected entities, and policy reason.
* Bind the approval cryptographically to the action hash, actor, session, and timestamp.
* Any parameter change by the LLM invalidates the previous approval.

## 12. Planner / Executor Separation

### Failure Modes
* The model directly mutates state or forms HTTP requests without a deterministic validation layer in between.
* The final answer claims an action was completed, even if the API call failed.

### Controls
* **The LLM proposes; the runtime executes.**
* The executor validates, authorizes, applies budgets, requests HITL if needed, and applies the state transition.
* The final user response must be based on the committed runtime state, not on the LLM's "plan."

## 13. Prompt Injection Boundaries

### Failure Modes
* A tool output says "ignore previous instructions" and the model obeys.
* A retrieved webpage attempts to exfiltrate secrets via an API call.

### Controls
* System, developer, and runtime policies must never be presented as part of the editable context.
* Box or quote documents containing instructions as raw data.
* Because the LLM never has access to plaintext secrets (see Section 2), injections cannot exfiltrate them.
* Execution must not trust LLM claims; it must verify policy independently.

## 14. Context Provenance

### Failure Modes
* The model cannot distinguish between user input, retrieved docs, tool results, and system policy.
* Debugging is impossible because it's unclear where the LLM got a specific "fact".

### Controls
* Mark context blocks by source (e.g., user message, runtime policy, retrieved document, tool result).
* Retain document IDs, version/freshness, and confidence scores for retrieved data.
* The audit trail must be able to answer: *"Which specific context block influenced this decision?"*

## 15. Observability and Auditability

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

## 16. Evaluation and Regression Testing

### Failure Modes
* Changing a prompt or model provider silently breaks tool choice accuracy.
* Cost regressions go unnoticed until the billing cycle ends.

### Controls
* Maintain golden test cases for common tasks.
* Run adversarial prompts to test injection resistance and policy bypasses.
* Use fixtures for malformed tool responses to test error handling.
* Monitor budget regressions (tokens, tool calls, latency) in CI/CD.

---

*This checklist is maintained by the engineering team at **[PlanVault](https://planvault.ai)** — an event-sourced execution layer for AI agents. We mapped these failure modes while building our core engine to prevent prompt injections, enforce approval gates, and secure tool calls. If your team is struggling with AI governance in production, feel free to use these principles or check out how we solve it.*
