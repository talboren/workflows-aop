# Workflow Lifecycle Hooks Proposal

> **Purpose**: Design proposal for extending Kibana's workflow engine to support **event-driven triggers** and **lifecycle hooks** — a unified model that lets teams like Agent Builder, Dashboards, and Cases delegate cross-cutting concerns (guardrails, PII reduction, enrichment) to user-authored or system workflows.

## What Are Lifecycle Hooks?

Today, the workflow engine supports **events** — when something happens in the system (e.g., an alert fires), workflows run in the background. Events are fire-and-forget: the system doesn't wait for the workflow to finish.

This proposal introduces a new concept: **lifecycle hooks**. A lifecycle hook lets you run a workflow **before or after** something happens — synchronously, inline with the operation. The system waits for the workflow to complete, and the workflow can inspect, modify, or reject the operation.

| | Events | Lifecycle Hooks |
|---|---|---|
| **When** | After something happened | Before/after something is about to happen |
| **Execution** | Background (async) | Inline, blocking (sync) |
| **Can modify the operation?** | No — it already happened | Yes — can modify inputs or reject |
| **API** | `emitEvent()` | `invokeHook()` |
| **Example** | `dashboard.created` — run enrichment after save | `dashboard.beforeCreate` — redact PII before save |

Both events and lifecycle hooks are registered the same way and appear identical in the workflow authoring surface. The difference is in how they're invoked and whether they block the caller.

## Proposed Design Decisions

The following decisions are reflected throughout the examples:

| Decision | Summary |
|----------|---------|
| **Lifecycle Hooks** | Blocking (synchronous) workflows are called *lifecycle hooks*. They run inline before an operation completes. |
| **Two APIs** | `emitEvent()` for async fire-and-forget; `invokeHook()` for sync lifecycle participation. Registration uses the same `registerTriggerDefinition()` for both. |
| **Always by-value** | No by-ref mutation. The trigger definition owns the `outputSchema`. Workflows return data via explicit `workflow.output` or implicitly when input and output schemas match. |
| **Trigger-level mode** | The team registering the trigger decides sync vs async — not the workflow author. A `sync` block on the trigger definition marks it as a lifecycle hook. |
| **Naming convention** | `beforeX` / `afterX` = lifecycle hook (sync), `X.created` (past tense) = event (async). Workflow authors subscribe the same way to both; the distinction is transparent. |
| **Error model** | `workflow.fail` = policy denial (intentional). Error codes distinguish policy outcomes from transient failures. `reason` codes (e.g. `'guardrail_violation'`, `'pii_detected'`, `'timeout'`) let callers handle policy denial differently from operational failure. |
| **Multi-turn state** | For hooks that produce persistent state (e.g. PII token maps), the hook output carries a stable `replacementsId` pointer. The caller stores it on the domain object (e.g. `Conversation`) and passes it back on subsequent hook calls. The persistence mechanism is internal to the step — the engine is agnostic to it. Storage is already available today and provided by the existing `ReplacementsRepository` in the inference plugin (`.kibana-anonymization-replacements` index, behind `xpack.anonymization.active`). |

---

## Core Concepts

### 1. Unified Registration Model

Event-driven triggers and lifecycle hooks follow the **same registration model**. The only distinction is that teams explicitly mark which triggers support synchronous execution via a `sync` block, and they are invoked differently (`emitEvent()` vs `invokeHook()`).

This means workflow authors do not need to think about sync versus async. Both appear the same in the workflow authoring surface and are subscribed to in the same way. For example, subscribing to `dashboard.created` (past tense — event) runs async after creation, while subscribing to `dashboard.beforeCreate` (lifecycle hook) runs sync before creation. Both `beforeX` and `afterX` hooks are synchronous — the naming convention uses past tense verbs (`created`, `completed`) to distinguish async events.

```typescript
// Lifecycle hook — invoked via invokeHook(), blocks the caller
workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.beforeCreate',
  eventSchema: z.object({
    title: z.string(),
    description: z.string().optional(),
  }),
  sync: {
    outputSchema: z.object({
      title: z.string(),
      description: z.string().optional(),
    }),
    maxTimeout: '10s',
    failurePolicy: 'open',
  },
});

// Event — invoked via emitEvent(), fire-and-forget
workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.created',
  eventSchema: z.object({
    dashboardId: z.string(),
    title: z.string(),
  }),
});
```

Both are registered via the same `registerTriggerDefinition()` API. The `sync` block is what distinguishes a lifecycle hook from an event.

### 2. Two APIs

| | `emitEvent()` | `invokeHook()` |
|---|---|---|
| **Execution** | Async — schedules via Task Manager | Sync — executes directly, blocks the caller |
| **Returns** | `void` | `HookResult` (status, outputs, error) |
| **Use case** | "Something happened" (after the fact) | "Something is about to happen — check it" (before the fact) |
| **Payload mutation** | No (caller already moved on) | No (always by-value — caller reads outputs) |
| **Error propagation** | Failures are the workflow's problem | Failures propagate to the caller |

```typescript
// Event — fire-and-forget, caller continues immediately
await workflowsClient.emitEvent('dashboard.created', { dashboardId: saved.id, title });

// Lifecycle hook — blocks until workflow completes, returns typed result
const result = await workflowsClient.invokeHook('dashboard.beforeCreate', {
  title: soAttributes.title,
  description: soAttributes.description,
});
if (result.status === 'failed') {
  throw Boom.badRequest(result.error?.message ?? 'Workflow rejected dashboard creation');
}
soAttributes.title = result.output.title;
soAttributes.description = result.output.description;
```

### 3. Output Model — Always By-Value

The trigger definition owns the success output schema. A lifecycle hook either completes with output matching that schema or fails with a structured error.

**Explicit output** — the workflow uses a `workflow.output` step to return values when the output schema differs from the input:

```yaml
steps:
  - name: anonymise
    type: ai.pii
    with:
      input: "{{ event.message }}"
      # ...
  - name: return_result
    type: workflow.output
    with:
      message: "{{ steps.anonymise.output.anonymised_text }}"
      replacementsId: "{{ steps.anonymise.output.replacements_id }}"
```

**Implicit output** — when the input and output schemas are the same shape, the workflow does not need a `workflow.output` step. The engine returns the (potentially modified) event fields as the output:

```yaml
# Input: { title, description }  —  Output: { title, description }
# No workflow.output needed — engine returns modified event fields
steps:
  - name: redact_title
    type: data.regexReplace
    with:
      input: "{{ event.title }}"
      patterns:
        - pattern: '\b\d{3}-\d{2}-\d{4}\b'
          replacement: '***-**-****'
```

### 4. Error Handling

Policy denial is represented as **intentional workflow failure** via `workflow.fail`. Error codes distinguish policy outcomes from transient or operational failures:

```yaml
# Policy denial — workflow.fail with a reason code
- name: block_message
  type: workflow.fail
  with:
    message: "Comment blocked: PII detected"
    reason: "pii_detected"           # policy outcome
```

The caller receives:

```typescript
const result = await workflowsClient.invokeHook('cases.beforeComment', commentEvent);
if (result.status === 'failed') {
  // result.error.reason === 'pii_detected' → policy denial
  // result.error.reason === 'timeout' → operational failure
  // result.error.reason === 'execution_error' → transient failure
}
```

**Fail-open vs fail-closed** is configured per trigger at registration time:

| Policy | Behavior | Use case |
|--------|----------|----------|
| `open` (default) | Timeout/error → caller proceeds as if no workflow ran | Dashboard save, most CRUD |
| `closed` | Timeout/error → caller receives the error, must handle it | Security guardrails, compliance |

### 5. Multiple Workflows on the Same Hook

When multiple workflows subscribe to the same lifecycle hook, the engine runs them **sequentially** in priority order. Each workflow receives the **same original input**, and their outputs are **merged** into a single `HookResult`:

```
invokeHook('dashboard.beforeCreate', { title, description })
    │
    ├── Workflow A (priority: 1) → output: { title: "Redacted A", description }
    ├── Workflow B (priority: 2) → output: { title: "Redacted B", description }
    │
    └── Merged HookResult.output: { title: "Redacted B", description }
         (last writer wins per field, ordered by priority)
```

- Each workflow gets the original event as input (not the previous workflow's output)
- Outputs are merged — later workflows (higher priority number) override earlier ones per field
- If any workflow calls `workflow.fail`, execution short-circuits and the failure is returned
- Priority is set by the workflow author at subscription time, giving users control over execution order

**Per-entity filtering** — workflows can use the `on.condition` clause to filter which events they handle. For example, an Agent Builder guardrail that only runs for a specific agent:

```yaml
triggers:
  - type: agent-builder.beforeChatRound
    on:
      condition: "event.agentId == 'security-analyst-agent'"
```

This lets different agents run different workflows on the same hook. A generic guardrail (no condition) runs for all agents, while agent-specific workflows run only when the condition matches.

### 6. Trigger Registration API

```typescript
interface TriggerDefinitionConfig {
  id: string;
  eventSchema: ZodSchema;

  // If present, this trigger is a lifecycle hook (sync)
  // If absent, this trigger is an event (async)
  sync?: {
    outputSchema: ZodSchema;               // required — the contract for hook output
    maxTimeout: string;                     // e.g. '10s' — max allowed workflow timeout
    failurePolicy: 'open' | 'closed';      // default: 'open'
    maxConcurrentWorkflows?: number;        // default: 5
  };
}
```

### 7. `HookResult` Type

```typescript
interface HookResult {
  status: 'completed' | 'failed' | 'timeout';
  output: Record<string, unknown>;         // matches trigger's outputSchema
  error?: {
    message: string;
    reason?: string;                       // 'pii_detected', 'guardrail_violation', 'timeout', etc.
  };
}
```

### 8. Multi-turn State: `replacementsId`

Some hooks produce state that must survive beyond a single request — the clearest example being PII anonymization in an agent chat. When a user's message is anonymized before inference, the token map that links each token back to its original value needs to be available in several scenarios where in-memory state is unavailable:

- **Next turn in the same conversation** — the same entity must map to the same token so the LLM context stays consistent. If turn 2 produces a different token for `jsmith` than turn 1 did, the model sees two representations of the same entity.
- **Returning to an old conversation** — a user may close the browser and reopen a conversation hours or days later. The Conversation object is loaded from ES; any in-memory state from the original session is gone. The `replacementsId` on the stored Conversation is the only thread back to the token map.
- **Page refresh** — same as above at a shorter timescale. A page reload wipes in-memory state. Without ES persistence, the next message in a live conversation would start a fresh token map, breaking consistency with everything the LLM has already seen.
- **Distributed execution** — Kibana runs on multiple nodes. Consecutive requests from the same user may land on different nodes. An in-memory map on node A is invisible to node B.
- **Inside the agent runner loop** — tool calls that need real values (e.g. a risk score lookup) must deanonymize their params mid-turn. Tool hooks run inside the runner and are not connected to the workflow output chain; they need a stable pointer to fetch the map on demand.

In-memory chaining is not sufficient for any of these cases. Storing the full `tokenMap` object in hook inputs would only solve within-request threading, and even then it would require threading the full map through every internal function call in the runner.

**`replacementsId`** solves this problem, the `ai.pii` step [PROPOSED STEP] persists the token map to ES and returns a stable string ID. Any code that needs the map can fetch it on demand using that ID.

The persistence infrastructure already exists in Kibana today. The **inference plugin** owns the `.kibana-anonymization-replacements` system index and exposes a `ReplacementsRepository` with `create`, `get`, and `update` operations, plus a `/internal/inference/anonymization/replacements/` API. The **anonymization plugin** owns the `.kibana-anonymization-profiles` index (anonymization rules and per-space encryption keys). Both are available on `main` behind the `xpack.anonymization.active` feature flag (default: `false`). The `ai.pii` step would build on top of this existing infrastructure — no new storage layer is needed.

```
Turn 1                                    Turn 2
──────                                    ──────
conversation.replacementsId = undefined   conversation.replacementsId = "repl-abc-123"
     │                                         │
     ▼                                         ▼
invokeHook('beforeInference',            invokeHook('beforeInference',
  { message, replacementsId: undefined })  { message, replacementsId: "repl-abc-123" })
     │                                         │
     │  ai.pii step:                           │  ai.pii step:
     │  → creates new token map                │  → loads existing map
     │  → writes to ES                         │  → extends it with any new PII
     │  → returns replacementsId               │  → returns same replacementsId
     │
     ▼
store "repl-abc-123" on Conversation
     │
     ▼
agentRunner.run({ replacementsId: "repl-abc-123" })
     │
     ├── beforeToolCall(...)   ─── see section 9
     │
     └── afterToolCall(...)    ─── see section 9
```

This pattern is specific to hooks that produce persistent state (like `beforeInference`). Simpler hooks (guardrails, redaction) do not need it — they complete within a single request and carry no persistent state.

### 9. Internal Tool Hooks: `beforeToolCall` / `afterToolCall`

`beforeToolCall` and `afterToolCall` are **new internal Agent Builder extension points** — not workflow triggers. They are synchronous hooks registered directly into the agent runner that fire on every tool call within a chat turn.

**Why not workflow triggers?** Tool calls happen inside the agent runner loop and may occur dozens of times per turn. Dispatching through the workflow engine on every tool call would add per-tool latency and consume Task Manager resources for what is a purely in-process operation. Internal hooks run synchronously in the same process with negligible overhead.

**What they do:**

| Hook | When it fires | What it does |
|------|--------------|--------------|
| `beforeToolCall` | Before a tool executes | Checks the allowlist; if the tool is allowed, fetches the token map from ES and deanonymizes the tool params so the tool receives real values |
| `afterToolCall` | After a tool returns | Always re-anonymizes the tool result regardless of allowlist — the LLM context must never contain real values |

**Why the allowlist matters:** some tools _require_ real values to execute correctly. A risk score lookup or a SIEM query must use the actual entity name, not a token — the underlying data store has no knowledge of the anonymization tokens. Other tools (summarization, classification, formatting) can safely operate on tokenized data and should never see real values. The `tool_deanonymization` block in the `ai.pii` step YAML is where the workflow author declares which tools are trusted. The `beforeToolCall` hook enforces that policy at runtime.

```yaml
# In the beforeInference workflow YAML — the author decides which tools see real values
tool_deanonymization:
  mode: allowlist
  tool_ids:
    - 'security.entity_analytics.risk_score'  # needs real entity name to query
    # tools NOT listed here receive tokenized params — they never see real values
```

The three available modes are:

| Mode | Behaviour |
|------|-----------|
| `allowlist` | Only the listed `tool_ids` receive deanonymized params |
| `all` | Every tool call is deanonymized before execution |
| `none` | Tool deanonymization is disabled — all tools receive tokens |

`afterToolCall` always re-anonymizes regardless of mode — the round-trip guarantee is: real values may go _in_ to a tool, but they never come _back out_ into the LLM context.

See the [Agent Builder guide](./agent-builder.md) for the full implementation: trigger schemas, workflow YAML, caller code, and tool hook registration.

---

## Architecture

### Lifecycle Hook Flow (sync)

```
  Caller (e.g. Dashboard)                    Workflow Engine
  =======================                    ==============

  1. Build event DTO
     { title, description }
                    ─── invokeHook('dashboard.beforeCreate', event) ───>
                                                            2. Resolve trigger definition
                                                               → sync block present → hook mode
                                                            3. Resolve subscribed workflows
                                                               → filter by on.condition
                                                               → order by priority
                                                            4. For each eligible workflow (sequentially):
                                                               a. Check circuit breaker
                                                               b. executeWorkflow(originalEvent)
                                                                  timeout = min(wf.timeout, trigger.maxTimeout)
                                                               c. Collect output
                                                               d. If workflow.fail → short-circuit, return error
                                                            5. Merge outputs from all workflows
                    <── HookResult { status, output, error } ──
  6. Check result:
     - failed → throw / reject operation
     - completed → use merged output fields
  7. Continue with save
```

### Event Flow (async) — unchanged from today

```
  Caller (e.g. Alerting)                     Workflow Engine
  ======================                     ==============

  1. Something happens
                    ─── emitEvent('alert.created', payload) ───>
                                                            2. Resolve subscribed workflows
                                                            3. scheduleWorkflow (Task Manager)
                    <── void ──
  3. Continue immediately                                   4. Workflow runs async via TM
```

---

## Lifecycle Hook Guardrails

Lifecycle hooks block HTTP requests. A broken or slow workflow can degrade the experience for all users. The following guardrails ensure hooks are safe by default:

| Guardrail | Description | New or existing? |
|-----------|-------------|------------------|
| **Hard timeout** | Platform cap (30s default) + trigger-level cap (`maxTimeout`). Effective timeout = `min(workflow.timeout, trigger.maxTimeout, PLATFORM_MAX)` | Existing infra, new sync cap |
| **Circuit breaker** | Auto-suspend workflows that fail repeatedly (5 consecutive or 50% in rolling window). Cooldown → half-open → re-evaluate. | New |
| **Fail-open/closed** | Configurable per trigger. Most teams want fail-open (broken workflow doesn't break the feature). Security teams want fail-closed. | New |
| **Save-time validation** | Warn on dangerous patterns in hook workflows: no `wait` steps, no `workflow.executeAsync`, `foreach` capped at 100 iterations, max 20 steps. | New |
| **Chain depth limit** | Sync chain depth capped at 2 (existing `EventChainContext` infra, tighter limit for hooks). Prevents hook → hook → hook cascade. | Existing infra, tighter limit |
| **Concurrency limit** | Max N workflows per hook invocation (default 5, configurable via `maxConcurrentWorkflows`). Sequential execution ordered by priority; first `workflow.fail` short-circuits the rest. Outputs are merged. | New |
| **Resource limits** | Step output size limits, Layer 1 pre-emptive I/O limits, Layer 2 generic output check — all inherited from existing engine infrastructure. | Existing |
| **Admin controls** | Global kill switch (`workflows.hooks.enabled: false`), per-trigger disable, per-workflow disable, observability dashboard. | New |

See [`archive/08-sync-workflow-guardrails.md`](./archive/08-sync-workflow-guardrails.md) for the full guardrails proposal with detailed architecture.

---

## Legend

Throughout the examples, comments indicate what exists today vs. what is proposed:

- `[EXISTS]` — This syntax/feature works today in the workflow engine
- `[PROPOSED]` — This syntax/feature does not exist yet and is part of this design proposal
- `[PROPOSED STEP]` — A new step type that would need to be implemented

---

## Team Integration Guides

Each guide contains trigger registrations, workflow YAML examples, and caller code for a specific team:

| Guide | Lifecycle hooks | Key patterns |
|-------|----------------|--------------|
| [**Agent Builder**](./agent-builder.md) | `beforeChatRound` (guardrails), `beforeInference` / `afterInference` (PII anonymization) | `replacementsId` for multi-turn token map persistence, generic attachment passthrough with `field_rules`, internal tool lifecycle hooks (`beforeToolCall`/`afterToolCall`), guardrail with `workflow.fail`, migration from `BeforeAgentWorkflowOutput` |
| [**Dashboards**](./dashboards.md) | `beforeCreate` (PII reduction) | Implicit output (input = output schema), `data.regexReplace` |
| [**Cases**](./cases.md) | `beforeCreate`, `beforeComment` (PII guardrail) | `ai.guardrail` step, `workflow.fail` for blocking |

---

## Archive

Previous iteration files (numbered examples, discussion docs, integration guide) are preserved in [`archive/`](./archive/) for historical reference.
