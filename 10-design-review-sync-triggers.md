# Design Review: Sync Workflow Triggers — Three Problems, Three Solutions

> **Status**: Proposal for team review
> **Context**: Review of the sync `emitEvent` design from the [workflows-aop](https://github.com/talboren/workflows-aop) proposal, informed by how Temporal, AWS Step Functions, Inngest, and other workflow engines solve the same problems.

## Executive Summary

The current proposal introduces sync (blocking) workflow execution via `sync: true` on the workflow's YAML trigger, a single `emitEvent()` API with a dual return type (`EmitEventResult | void`), and two communication patterns (`event.mutate` by-ref vs `workflow.output` by-value). This review identifies three fundamental design problems and proposes solutions for each.

| # | Problem | Solution |
|---|---------|----------|
| **1** | The subscriber (workflow YAML) decides sync/async — the caller can't predict behavior | Move guardrails + `outputSchema` to `registerTriggerDefinition` via a `sync` capability block |
| **2** | One API (`emitEvent`) has an ambiguous return type that changes based on subscriber config | Split into `emitEvent()` (async, void) and `invokeEvent()` (sync, typed result) |
| **3** | `event.mutate` (by-ref) allows invisible mutation of the caller's data | Always by-value — the caller receives output and decides what to accept |
| **4** | Sync workflows block the main process sequentially, adding unnecessary latency | Support parallel execution via `Promise.all` — run workflow and main process concurrently |

---

## Problem 1: Who Decides Sync vs Async?

### What the current proposal does

The workflow *author* adds `sync: true` to their trigger clause in YAML:

```yaml
# Workflow YAML — author decides
triggers:
  - type: dashboard.onCreate
    sync: true              # [PROPOSED]
```

The `emitEvent()` function then checks at runtime whether any subscribed workflow has `sync: true`, and branches between `scheduleWorkflow` (Task Manager) and `executeWorkflow` (direct execution).

### Why this is problematic

**The workflow author doesn't own the code path being blocked.** The dashboard team owns `create.ts`. They wrote the HTTP handler. They are responsible for its latency. But a workflow author in a different space can add `sync: true` to a `dashboard.onCreate` trigger and suddenly:

- The dashboard save blocks for up to 30 seconds
- If the workflow calls `workflow.fail`, all dashboard creation fails for all users
- If the workflow has a bug, the circuit breaker (proposed in `08-sync-workflow-guardrails.md`) must intervene

This is the **"action at a distance"** problem. A YAML file changes the behavior of production TypeScript code without the code's owner knowing.

**Mixed-mode execution adds complexity.** If two workflows subscribe to `dashboard.onCreate` — one with `sync: true` and one without — the engine must handle both modes for the same trigger. The sync ones run directly, the async ones go through Task Manager. The return type to the caller depends on whether any sync subscriber exists. This branching logic is a source of bugs and cognitive overhead.

**Guardrails require back-referencing.** The guardrails proposal (`08-sync-workflow-guardrails.md`) introduces per-trigger constraints like `maxTimeout`, `failurePolicy`, and `maxConcurrentWorkflows`. But these are properties of the *trigger registration*, while the sync decision lives in the *workflow YAML*. Save-time validation must cross-reference the workflow's `sync: true` against the trigger's constraints — an extra layer of indirection that wouldn't exist if the mode lived on the trigger.

### How other engines handle this

**No major workflow engine lets the subscriber decide sync vs async.** The caller or the trigger definition always owns this decision:

| Engine | Who decides | Mechanism |
|--------|-------------|-----------|
| **Temporal** | Caller | Three distinct APIs: `signal()` (async), `executeUpdate()` (sync), `query()` (read-only sync) |
| **AWS Step Functions** | Workflow type + Caller | Express vs Standard workflow types; `StartSyncExecution` vs `StartExecution` invocations |
| **AWS EventBridge Pipes** | Pipe config (emitter) | `REQUEST_RESPONSE` (sync) vs `FIRE_AND_FORGET` (async) on the pipe, not the target |
| **Inngest** | Caller | `step.invoke()` (sync, returns result) vs `step.sendEvent()` (async, fire-and-forget) |
| **Zapier** | Platform | Always async — no sync option exists |
| **Power Automate** | Flow author (emitter side) | Toggle on the Response action |
| **GitHub Actions** | Caller | `workflow_call` (sync, inline) vs `workflow_dispatch` (async, separate run) |

**References:**
- [Temporal: Workflow Update announcement](https://temporal.io/blog/announcing-a-new-operation-workflow-update) — "Signals are asynchronous write requests... Updates are synchronous, tracked write requests"
- [Temporal: Message passing docs](https://docs.temporal.io/encyclopedia/workflow-message-passing/) — comparison table: Signals vs Updates
- [AWS: Choosing workflow type](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-standard-vs-express.html) — Standard (async) vs Express (sync-capable)
- [AWS: Synchronous Express Workflows](https://aws.amazon.com/blogs/compute/new-synchronous-express-workflows-for-aws-step-functions/) — sync as a distinct invocation type
- [EventBridge Pipes: Lambda parameters](https://docs.aws.amazon.com/eventbridge/latest/pipes-reference/API_PipeTargetLambdaFunctionParameters.html) — `FIRE_AND_FORGET` vs `REQUEST_RESPONSE`
- [Inngest: Invoking functions directly](https://inngest.com/docs/guides/invoking-functions-directly) — "invoke provides direct, RPC-like function calls"
- [Inngest: Sending events from functions](https://www.inngest.com/docs/guides/sending-events-from-functions) — "sendEvent does not wait for triggered functions to complete"

### Solution: `sync` capability block on `registerTriggerDefinition`

The team that registers the trigger owns whether sync invocation is supported. The workflow author just subscribes — the capability is inherited. There is no explicit `mode` flag — the **presence of the `sync` block** means the trigger supports `invokeEvent()`; its **absence** means async-only via `emitEvent()`.

```typescript
// Trigger that supports sync invocation — registered by the dashboard team
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
    maxConcurrentWorkflows: 3,
  },
});

// Async-only trigger — registered by the alerting team
// No `sync` block → invokeEvent() will throw if called on this trigger
workflowsExtensions.registerTriggerDefinition({
  id: 'alert.created',
  eventSchema: z.object({
    alertId: z.string(),
    severity: z.string(),
  }),
});
```

Note that `emitEvent()` (fire-and-forget) works on **any** trigger, regardless of whether `sync` is defined. This allows async consumers (audit logging, telemetry) to subscribe to a trigger that also supports sync invocation. Only `invokeEvent()` requires the `sync` block.

**What changes in the workflow YAML:**

```yaml
# Before (current proposal)
triggers:
  - type: dashboard.onCreate
    sync: true                    # workflow author decides — problematic

# After (proposed)
triggers:
  - type: dashboard.beforeCreate  # sync capability is inherited from trigger definition
                                   # no sync/async config in YAML
```

**Why `outputSchema`?** When a trigger supports sync invocation, the caller expects structured output. The `outputSchema` defines what the workflow must return via `workflow.output`. The engine validates this at three layers:

1. **Save-time**: When a workflow is saved with a sync trigger subscription, the engine checks that the workflow's declared `outputs` is compatible with the trigger's `outputSchema`. Incompatible outputs → save rejected.
2. **Runtime (step-level)**: The existing `workflow.output` step already validates output against the workflow's declared `outputs` schema via `buildFieldsZodValidator`. If the trigger's `outputSchema` is propagated as the workflow's output contract, this validation works without changes.
3. **Post-execution (defense-in-depth)**: Before returning the result to the caller, validate the execution output against `outputSchema` one more time.

**Why guardrails live here:** `maxTimeout`, `failurePolicy`, and `maxConcurrentWorkflows` are properties of the integration point, not of individual workflows. They belong on the trigger definition because:
- The dashboard team knows their HTTP handler has a 2-second latency budget — they set `maxTimeout: '2s'`
- The cases team wants sync as best-effort — they set `failurePolicy: 'open'`
- The AB team needs strict enforcement — they set `failurePolicy: 'closed'`

### Full trigger definition type

```typescript
interface SyncCapability<OutputSchema extends z.ZodType> {
  outputSchema: OutputSchema;
  maxTimeout: string;
  failurePolicy: 'open' | 'closed';
  maxConcurrentWorkflows?: number;    // default: 5
}

interface TriggerDefinition<
  EventSchema extends z.ZodType = z.ZodType,
  OutputSchema extends z.ZodType = z.ZodType
> {
  id: string;
  eventSchema: EventSchema;

  /**
   * If present, this trigger supports synchronous invocation via invokeEvent().
   * If absent, only emitEvent() (async) is supported.
   *
   * emitEvent() works on ALL triggers regardless of this field.
   */
  sync?: SyncCapability<OutputSchema>;
}
```

The engine infers the trigger's capabilities from the shape of the registration:

| `sync` block | `emitEvent()` | `invokeEvent()` |
|-------------|---------------|-----------------|
| Present | Works (fire-and-forget) | Works (sync, returns result) |
| Absent | Works (fire-and-forget) | Throws: "This trigger has no sync capability" |

---

## Problem 2: One API with an Ambiguous Return Type

### What the current proposal does

```typescript
// Current proposal — one API, dual return type
export interface WorkflowsClient {
  emitEvent(triggerId: string, payload: Record<string, unknown>): Promise<EmitEventResult | void>;
}
```

The return type is `EmitEventResult | void`. The caller must check at runtime whether a result was returned:

```typescript
const result = await workflowsClient.emitEvent('dashboard.onCreate', dashboardEvent);
if (result?.status === 'failed') {           // is `result` even defined?
  throw Boom.badRequest(result.error?.message);
}
```

### Why this is problematic

**The return type changes based on runtime configuration.** Whether `emitEvent` returns `void` or `EmitEventResult` depends on whether any subscribed workflow has `sync: true`. The caller can't know at compile time what to expect. This is the TypeScript equivalent of a function that sometimes returns a number and sometimes returns undefined, depending on a database value.

**The caller must write defensive code for both branches.** Every call site needs `if (result?.status === ...)` even though for a given trigger, the behavior is always the same. This is boilerplate that obscures intent.

**The name `emitEvent` implies fire-and-forget.** "Emit" suggests broadcasting a notification — the EventEmitter pattern. Developers reading `await workflowsClient.emitEvent(...)` won't expect it to block for 10 seconds, reject the operation, or modify the payload. The `09-naming-and-trigger-mode-proposal.md` document acknowledges this problem but solves it with a single renamed API (`triggerWorkflows`) that still has the `| void` return type ambiguity.

### How Temporal solved this

Temporal had exactly this problem and solved it by creating three APIs with distinct type signatures. Before the Update API existed (GA 2024), the workaround was "send a Signal, then poll with Query" — which they [explicitly called out as error-prone](https://temporal.io/blog/announcing-a-new-operation-workflow-update).

```typescript
// Temporal's solution: the CALLER picks the API, the type system enforces the contract

// Signal — always async, always void
await handle.signal(approve, { name: 'me' });

// Update — always sync, always returns typed result
const previous: Language = await handle.executeUpdate(setLanguage, {
  args: [Language.CHINESE],
});

// Query — always sync, always returns typed result, read-only
const languages: Language[] = await handle.query(getLanguages, {
  includeUnsupported: false,
});
```

There is no single `sendMessage()` function with a polymorphic return type. Each API has one behavior, one return type, one contract.

### Solution: Split into `emitEvent()` and `invokeEvent()`

```typescript
export interface WorkflowsClient {
  /**
   * Fire-and-forget: schedule subscribed workflows via Task Manager.
   * Always returns void. Works on ANY trigger (with or without sync capability).
   */
  emitEvent(triggerId: string, payload: Record<string, unknown>): Promise<void>;

  /**
   * Synchronous invocation: execute subscribed workflows directly and return the result.
   * Always returns InvokeEventResult. Only works on triggers that have a `sync` block.
   * Throws if the trigger has no sync capability.
   */
  invokeEvent(triggerId: string, payload: Record<string, unknown>): Promise<InvokeEventResult>;
}

export interface InvokeEventResult {
  status: 'completed' | 'failed' | 'timeout';
  output?: Record<string, unknown>;   // validated against trigger's outputSchema
  error?: { message: string; reason?: string };
}
```

**The engine enforces correct usage:**

```typescript
// Sync invocation — trigger has a `sync` block
const result = await workflowsClient.invokeEvent('dashboard.beforeCreate', event);

// Async invocation — works on any trigger
await workflowsClient.emitEvent('alert.created', payload);

// Also valid — emitEvent works on triggers with sync capability too
// (useful for async consumers like audit logging on the same trigger)
await workflowsClient.emitEvent('dashboard.beforeCreate', event);

// ERROR — invokeEvent on a trigger without sync capability
await workflowsClient.invokeEvent('alert.created', payload);
// → throws: "Trigger 'alert.created' has no sync capability. Use emitEvent() instead."
```

**Why not a single API with a better name?** The `09-naming-and-trigger-mode-proposal.md` recommends renaming to `triggerWorkflows()` with the same dual return type. This solves the naming problem but not the type safety problem. The caller still gets `Promise<WorkflowResult | void>` and must check at runtime. Two APIs eliminate the ambiguity entirely — each call site is self-documenting:

```typescript
// Reading this code, you know EXACTLY what happens:
const { output, error } = await workflowsClient.invokeEvent('dashboard.beforeCreate', event);
//     ^^^^^^^^^^^^^^      always defined, never void           ^^^^^^^^^^^^^^^^^^^^
//     typed result                                             sync trigger — blocks

await workflowsClient.emitEvent('alert.created', payload);
//                               ^^^^^^^^^^^^^^
//                               async trigger — fire and forget, no result
```

### Relationship to Problem 1

The two APIs and the trigger `sync` capability block reinforce each other:

- **`sync` block on the trigger** declares that sync invocation is supported and defines the guardrails. The workflow author can't make a trigger sync — only the registrar can.
- **Two APIs** give the caller compile-time clarity: `invokeEvent` always returns a result, `emitEvent` always returns void.
- **The engine validates**: calling `invokeEvent` on a trigger without a `sync` block is a hard error.
- **`emitEvent` is universal**: it works on any trigger, allowing async consumers (audit, telemetry) to subscribe to triggers that also support sync invocation.

Without the `sync` block, there's no declaration of sync capability or guardrails. Without two APIs, the return type is ambiguous. Together, they create a system where misuse is caught early and every call site is self-documenting.

---

## Problem 3: By-Ref Mutation vs By-Value Output

### What the current proposal does

Two patterns are proposed side-by-side:

**By-ref (`event.mutate`)** — the workflow modifies the caller's event object in-place:

```typescript
// Caller passes an object
const dashboardEvent = { title: soAttributes.title, description: soAttributes.description };
await workflowsClient.emitEvent('dashboard.onCreate', dashboardEvent);
// dashboardEvent.title and .description are now modified by the workflow
soAttributes.title = dashboardEvent.title;
```

**By-value (`workflow.output`)** — the workflow returns new data, the caller copies it back:

```typescript
const result = await workflowsClient.emitEvent('dashboard.onCreate', dashboardEvent);
if (result?.outputs) {
  soAttributes.title = result.outputs.title;
  soAttributes.description = result.outputs.description;
}
```

The proposal favors by-ref for ergonomics, citing the "7+ fields" problem where by-value becomes verbose.

### Why by-ref (`event.mutate`) is problematic

**Invisible mutation.** The caller passes an object to `emitEvent` and gets it back modified. There's no signal in the code that `dashboardEvent.title` changed. A developer reading the code sees:

```typescript
const dashboardEvent = { title: 'My Dashboard', description: 'Some text' };
await workflowsClient.emitEvent('dashboard.onCreate', dashboardEvent);
soAttributes.title = dashboardEvent.title;  // this looks like a no-op
```

Nothing in this code suggests that `dashboardEvent.title` might now be different from `'My Dashboard'`. This violates the [principle of least surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).

**No schema for mutations.** With `event.mutate`, the workflow can modify *any* field on the event object. There's no contract for which fields are "mutable" vs "read-only." A workflow intended to redact the description could accidentally overwrite the title. The caller has no way to restrict what the workflow touches.

**Breaks with multiple workflows.** If three sync workflows subscribe to the same trigger, and all use `event.mutate`, they chain mutations on the same object. The order of execution determines the final state. If workflow B's mutation depends on seeing workflow A's output, you have an implicit ordering dependency that's invisible in both YAML files.

**Breaks with sandboxing.** If the engine ever deep-clones the event for isolation, logging, or parallel execution, `event.mutate` silently stops working — the workflow mutates a clone, and the caller's original object is unchanged. The pattern is inherently fragile.

**No prior art in workflow engines.** None of the surveyed engines (Temporal, AWS Step Functions, Inngest, Zapier, Power Automate) support by-ref mutation of the caller's input. All use by-value return:

| Engine | Output pattern |
|--------|---------------|
| Temporal Update | `executeUpdate()` returns a value; original workflow state unchanged by caller |
| AWS Step Functions | `StartSyncExecution` returns `{ output }` as a JSON string |
| Inngest | `step.invoke()` returns the function's return value |
| Power Automate | HTTP Response action returns a body |

### Solution: Always by-value, caller decides what to accept

The caller receives the workflow's output as a **separate object** and explicitly decides what to merge back. The original input is never mutated.

```typescript
const event = { title: soAttributes.title, description: soAttributes.description };
const { output, error } = await workflowsClient.invokeEvent('dashboard.beforeCreate', event);

if (error) {
  throw Boom.badRequest(error.message);
}

// event is UNCHANGED — the workflow got a deep clone
// output is the workflow's returned data, validated against outputSchema

// Caller decides what to accept:
soAttributes.title = output.title;
soAttributes.description = output.description;
```

**Solving the "7+ fields" ergonomics problem:**

The concern that by-value is "unlovable" with many fields is a caller-side ergonomics issue, not a design flaw. It's solved with a one-liner:

```typescript
// Option A: spread/merge all output fields
Object.assign(soAttributes, output);

// Option B: destructure the specific fields you want
const { title, description } = output;
Object.assign(soAttributes, { title, description });
```

The key insight: **`Object.assign(soAttributes, output)` is one line regardless of whether there are 2 fields or 20.** The ergonomics concern disappears once you use standard JS merge patterns instead of field-by-field assignment.

**How `outputSchema` enforces the contract:**

The trigger's `outputSchema` (from Problem 1) ensures the workflow returns the right shape:

1. **Save-time**: Workflow's declared `outputs` must be compatible with the trigger's `outputSchema`
2. **Runtime**: The existing `workflow.output` step validates values against the declared schema
3. **Post-execution**: `invokeEvent` validates the result against `outputSchema` before returning to caller

The `event.mutate` step type is not needed. `workflow.output` (which already exists) is sufficient.

### What the workflow YAML looks like

No change from what exists today — the workflow returns data via `workflow.output`:

```yaml
version: '1'
name: Dashboard PII Reduction
triggers:
  - type: dashboard.beforeCreate

# outputs are validated against the trigger's outputSchema at save time
outputs:
  type: object
  properties:
    title:
      type: string
    description:
      type: string

steps:
  - name: redact_title
    type: data.regexReplace
    with:
      input: "{{ event.title }}"
      patterns:
        - pattern: '\b\d{3}-\d{2}-\d{4}\b'
          replacement: '***-**-****'

  - name: redact_description
    type: data.regexReplace
    if: "event.description : *"
    with:
      input: "{{ event.description }}"
      patterns:
        - pattern: '\b\d{3}-\d{2}-\d{4}\b'
          replacement: '***-**-****'

  - name: return_redacted
    type: workflow.output
    with:
      title: "{{ steps.redact_title.output }}"
      description: "{{ steps.redact_description.output | default: event.description }}"
```

---

## Problem 4: Sequential Blocking Adds Unnecessary Latency

### What the current proposal does

In the current proposal, sync workflows run **before** the main process. The caller awaits the workflow, inspects the result, and only then proceeds with the actual operation:

```typescript
// Current: sequential — workflow runs first, then the process
const result = await workflowsClient.emitEvent('agent-builder.chatRound', roundEvent);
if (result?.status === 'failed') throw ...;

// Only now does the actual inference start
const response = await llm.inference(roundEvent.message);
```

Total time = workflow duration + inference duration. For a guardrail that checks "is this message off-topic?" via an LLM call, the workflow might take 2-3 seconds. The inference itself takes 5-10 seconds. Sequential execution means 7-13 seconds total.

### Why this is problematic for certain use cases

Not all sync workflows need to **precede** the main process. Some are **concurrent guards** — they run alongside the process and only matter if they fail:

- **Off-topic detection**: Check if the prompt is off-topic while inference is already running. If the guardrail passes, the inference result is used. If it fails, discard the inference result and reject.
- **PII scanning**: Scan the input for PII while the actual operation proceeds. If PII is found, abort. If not, the operation already completed — no wasted time.
- **Rate limit / quota check**: Verify the user hasn't exceeded limits while the operation starts. Abort if they have.

In all these cases, the workflow doesn't **transform** the input — it's a pass/fail gate. The main process doesn't need the workflow's output to start. Running them in parallel saves the workflow's entire duration from the critical path.

### Solution: Parallel execution via `Promise.all`

The caller should be able to run the workflow and the main process concurrently. If the workflow fails, the whole thing fails. If the workflow passes, the main process result is used.

```typescript
// Parallel: workflow and inference run concurrently
const [guardrailResult, inferenceResponse] = await Promise.all([
  workflowsClient.invokeEvent('agent-builder.beforeChatRound', { message: prompt }),
  llm.inference(prompt),
]);

if (guardrailResult.error) {
  // Guardrail failed — discard inferenceResponse, reject the operation
  throw createWorkflowAbortedError(guardrailResult.error.message);
}

// Guardrail passed — use the inference result
return inferenceResponse;
```

Total time = max(workflow duration, inference duration) instead of sum. If the guardrail takes 2s and inference takes 8s, total is 8s instead of 10s.

### This works naturally with the two-API split

The key insight is that **this is not a new API or engine feature** — it falls out naturally from `invokeEvent` returning a `Promise`. The caller controls the concurrency. The engine doesn't need to know that the caller is running something else in parallel.

But it only works cleanly with the by-value pattern (Problem 3). If the workflow used `event.mutate` to transform the input, you couldn't start the main process in parallel — it would be operating on the un-transformed input. By-value means:

- The workflow returns its result (pass/fail, or transformed data) as a **separate object**
- The main process can use the **original input** to start immediately
- The caller merges results after both complete

### When to use sequential vs parallel

| Pattern | Use when | Example |
|---------|----------|---------|
| **Sequential** (`await invokeEvent` then process) | Workflow **transforms** the input and the process needs the transformed version | PII anonymization before inference — the LLM must see the redacted text |
| **Parallel** (`Promise.all([invokeEvent, process])`) | Workflow is a **pass/fail gate** and the process doesn't need the workflow's output to start | Off-topic detection, rate limiting, quota checks |

The caller makes this decision — not the engine, not the workflow. This is another benefit of the by-value, caller-controls-everything approach.

### Example: Agent Builder with parallel guardrail

```typescript
// Before: sequential — 2s guardrail + 8s inference = 10s
const wfResult = await workflowsClient.invokeEvent('agent-builder.beforeChatRound', {
  message: prompt,
});
if (wfResult.error) throw createWorkflowAbortedError(wfResult.error.message);
const response = await llm.inference(prompt);

// After: parallel — max(2s, 8s) = 8s
const [wfResult, response] = await Promise.all([
  workflowsClient.invokeEvent('agent-builder.beforeChatRound', { message: prompt }),
  llm.inference(prompt),
]);
if (wfResult.error) throw createWorkflowAbortedError(wfResult.error.message);
// guardrail passed — use inference result
```

This pattern also composes with `AbortController` for early cancellation — if the guardrail fails quickly (e.g., 200ms), the caller can abort the inference to save resources:

```typescript
const abortController = new AbortController();

const [wfResult, response] = await Promise.allSettled([
  workflowsClient.invokeEvent('agent-builder.beforeChatRound', { message: prompt }),
  llm.inference(prompt, { signal: abortController.signal }),
]);

if (wfResult.status === 'fulfilled' && wfResult.value.error) {
  abortController.abort();
  throw createWorkflowAbortedError(wfResult.value.error.message);
}
```

---

## How the Four Solutions Work Together

### Architecture flow

```
  Dashboard Team (code owner)         Workflow Engine              Workflow Author (YAML)
  ═══════════════════════════         ══════════════              ═══════════════════════

  1. Register trigger at setup:
     registerTriggerDefinition({
       id: 'dashboard.beforeCreate',
       eventSchema: z.object({...}),
       sync: {                           ─── Stored ───→  Trigger Registry
         outputSchema: z.object({...}),                    - eventSchema
         maxTimeout: '10s',                                - sync capability:
         failurePolicy: 'open',                              outputSchema,
       },                                                    guardrails
     })
                                                                │
                                                                │  At workflow save time:
                                        ◄── Validate ──────────┤  - workflow outputs
                                                                │    compatible with
                                                                │    outputSchema?
                                                                │  - workflow steps
                                                                │    safe for sync?
                                                                │    (no wait, no
                                                                ▼    executeAsync)

                                                          3. Workflow subscribes:
                                                             triggers:
                                                               - type: dashboard.beforeCreate
                                                             outputs: { title, description }
                                                             steps: [...]

  4. At request time:
     const event = {
       title: attrs.title,
       description: attrs.description,
     };

     const { output, error } =
       await workflowsClient
         .invokeEvent(                  ─── invokeEvent() ──→  5. Engine checks:
           'dashboard.beforeCreate',                              - trigger has sync block? ✓
           event                                                  - circuit breaker open? ✗
         );                                                       - resolve subscribed workflows
                                                                  - deep-clone event as input
                                                                  - executeWorkflow() directly
                                                                    with timeout enforcement
                                                                  - validate output against
                                                                    outputSchema
                                        ◄── InvokeEventResult ── 6. Return typed result

  7. Caller decides what to accept:
     if (error) throw Boom.badRequest(error.message);
     Object.assign(attrs, output);

     // Original `event` is UNCHANGED
     // `output` is validated, typed, and safe

  8. Save to ES:
     await savedObjects.client.create(
       DASHBOARD_TYPE, attrs, ...
     );
```

### Before/after comparison

**Before (current proposal):**

```typescript
// Who decides sync? → Workflow YAML (sync: true)
// What API? → emitEvent (ambiguous return type)
// How does data come back? → event.mutate (invisible mutation) OR workflow.output (verbose)

const dashboardEvent = { title, description };
const result = await workflowsClient.emitEvent('dashboard.onCreate', dashboardEvent);
// result is EmitEventResult | void — depends on subscribers
if (result?.status === 'failed') {
  throw Boom.badRequest(result.error?.message);
}
// Was dashboardEvent mutated? Maybe. Depends on the workflow.
soAttributes.title = dashboardEvent.title;         // no-op or mutated? unclear
soAttributes.description = dashboardEvent.description;
```

**After (this proposal):**

```typescript
// Who decides sync? → Trigger definition (sync block present)
// What API? → invokeEvent (always returns typed result)
// How does data come back? → output object (by-value, caller merges)

const { output, error } = await workflowsClient.invokeEvent('dashboard.beforeCreate', {
  title,
  description,
});

if (error) {
  throw Boom.badRequest(error.message);
}

Object.assign(soAttributes, output);
```

---

## Impact on Existing Proposal Files

| File | Impact |
|------|--------|
| `01-guardrails-sync.yaml` | Remove `sync: true` from trigger clause. Trigger `agent-builder.chatRound` is registered with a `sync` block by AB team. `workflow.fail` works unchanged. |
| `02-pii-anonymization-before.yaml` | Remove `sync: true`. Replace `event.mutate` step with `workflow.output` step. Add `outputs:` declaration. |
| `03-pii-anonymization-after.yaml` | Same — remove `event.mutate`, use `workflow.output`. |
| `04-dashboard-oncreate-byref.yaml` | Retire in favor of by-value approach. The by-ref variant is no longer proposed. |
| `05-dashboard-oncreate-byvalue.yaml` | Becomes the canonical pattern. Rename trigger to `dashboard.beforeCreate`. Remove `sync: true`. |
| `06-alert-enrichment.yaml` | Unchanged — async trigger, `emitEvent`, no output. |
| `07-cases-guardrail.yaml` | Remove `sync: true`. Trigger `cases.beforeCreateComment` registered with a `sync` block by cases team. |
| `08-sync-workflow-guardrails.md` | Guardrail config moves from standalone proposal into `registerTriggerDefinition`. The content remains valid as rationale. |
| `09-naming-and-trigger-mode-proposal.md` | Superseded by this document. The single-API recommendation (`triggerWorkflows`) is replaced by the two-API split. |
| `integration-guide.md` | Update call sites to use `invokeEvent`/`emitEvent`. Remove `event.mutate` references. Add `outputSchema` to trigger registrations. |

## Open Questions

1. **Backward compatibility**: `emitEvent` exists in production today (async-only). It continues to work for async triggers. No breaking change.

2. **Convention for sync trigger naming**: Should we enforce `before*` / `after*` naming? `dashboard.beforeCreate` (sync) and `dashboard.afterCreate` (async) make the lifecycle phase explicit. Alternatively, `dashboard.create.before` / `dashboard.create.after` for consistent nesting.

3. **`outputSchema` omission shortcut**: When a sync trigger's `outputSchema` mirrors `eventSchema` (the common "transform" case), should the engine auto-default `outputSchema = eventSchema` if omitted? This reduces boilerplate for registrars but makes the contract less explicit.

4. **Multiple sync workflows**: When N workflows subscribe to a sync trigger, they run sequentially (ordered by creation date). The first `workflow.fail` short-circuits the rest. Outputs from earlier workflows are NOT chained as inputs to later ones — each gets the original event. The final `output` returned to the caller is from the last successful workflow. Should we instead merge outputs from all workflows?

5. **AB migration path**: Agent Builder currently uses a custom loop with `BeforeAgentWorkflowOutput` (`abort`, `abort_message`, `new_prompt`). Migration: register `agent-builder.beforeChatRound` with a `sync` block containing `outputSchema: z.object({ message: z.string() })` and `failurePolicy: 'closed'`. The `abort` semantic maps to `workflow.fail`. The `new_prompt` semantic maps to returning `{ message: newPrompt }` via `workflow.output`. The custom `runBeforeAgentWorkflows` loop is replaced by a single `invokeEvent` call.
