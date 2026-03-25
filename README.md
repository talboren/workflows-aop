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
| **Error model** | `workflow.fail` = policy denial (intentional). Error codes distinguish policy outcomes from transient failures. |

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
      tokenMap: "{{ steps.anonymise.output.token_map }}"
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
| [**Agent Builder**](./agent-builder.md) | `beforeChatRound` (guardrails), `beforeInference` / `afterInference` (PII anonymization) | Trigger output chaining, guardrail with `workflow.fail`, migration from `BeforeAgentWorkflowOutput` |
| [**Dashboards**](./dashboards.md) | `beforeCreate` (PII reduction) | Implicit output (input = output schema), `data.regexReplace` |
| [**Cases**](./cases.md) | `beforeCreate`, `beforeComment` (PII guardrail) | `ai.guardrail` step, `workflow.fail` for blocking |

---

## Archive

Previous iteration files (numbered examples, discussion docs, integration guide) are preserved in [`archive/`](./archive/) for historical reference.
