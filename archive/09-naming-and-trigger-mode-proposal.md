# Proposal: API Naming & Trigger-Level Mode Configuration

> **Status**: Discussion — needs team alignment before implementation

## Problem 1: `emitEvent` Is Misleading for Sync Workflows

The current proposal uses `emitEvent()` for both fire-and-forget and blocking calls. This creates confusion:

| What the name implies | What actually happens (sync case) |
|---|---|
| "Emit" → fire and forget | Blocks the HTTP request |
| "Event" → notification, after the fact | Interception, before the fact |
| Returns `void` | Returns a result that can reject the operation |
| No side effects on the caller | May mutate the payload (by-ref) |

When a developer reads `await workflowsClient.emitEvent('dashboard.onCreate', event)`, there is no signal in the code that this call:
- Blocks until all sync workflows complete
- May modify `event.title` and `event.description`
- May throw (if `workflow.fail` fires)

This is a readability and maintainability problem, especially for teams integrating for the first time.

---

## Options Considered

### Option A: Two Separate APIs

| Method | Semantics | Returns |
|--------|-----------|---------|
| `emitEvent(triggerId, payload)` | Fire-and-forget, async | `void` |
| `invokeHook(triggerId, payload)` | Synchronous interception, blocks, returns result | `HookResult` |

```typescript
// Async — "something happened, notify interested workflows"
workflowsClient.emitEvent('alert.created', alertPayload);

// Sync — "something is about to happen, let workflows inspect/modify/reject"
const result = await workflowsClient.invokeHook('dashboard.onCreate', dashboardEvent);
if (result?.status === 'failed') {
  throw Boom.badRequest(result.error.message);
}
```

**Pros:**
- Maximum clarity at every call site — no ambiguity about what happens
- Different return types match the actual contracts
- "Hook" vocabulary is well-understood (git hooks, React lifecycle, middleware)

**Cons:**
- Two methods to learn and maintain
- Under the hood, both route to the same trigger resolution and execution pipeline — the split is purely a caller-side concern
- If a trigger later changes from async to sync, every call site must be updated from `emitEvent` to `invokeHook`

### Option B: Single Unified API with Neutral Name (Recommended)

Rename `emitEvent` to a name that doesn't imply async-only semantics:

| Candidate | Assessment |
|-----------|------------|
| `triggerWorkflows()` | Uses existing domain language ("triggers"), neutral about sync/async. Slight concern: "trigger" is already overloaded (trigger definitions, trigger types, trigger conditions). |
| `invokeWorkflows()` | Clear and direct. Less overloaded than "trigger". |
| `applyWorkflows()` | Suggests transformation (good for by-ref), but doesn't convey blocking/rejection. |

**Recommended name: `triggerWorkflows()`**

```typescript
// The caller doesn't decide sync vs async — the trigger definition does (see Problem 2 below)
const result = await workflowsClient.triggerWorkflows('dashboard.onCreate', dashboardEvent);

if (result?.status === 'failed') {
  throw Boom.badRequest(result.error?.message ?? 'Workflow rejected dashboard creation');
}

// By-ref: dashboardEvent may have been modified
soAttributes.title = dashboardEvent.title;
```

**Pros:**
- Single API surface, simpler mental model
- "trigger" maps directly to the YAML concept (`triggers:` block)
- The trigger's registered mode (sync/async) determines behavior transparently
- If a trigger changes mode, no call site changes needed

**Cons:**
- The call site still doesn't visually distinguish sync from async (addressed by trigger-level mode — see below)
- "trigger" word is used in multiple contexts (though consistently in the workflow domain)

### Option C: Same Root, Different Verbs

| Method | Meaning |
|--------|---------|
| `emitEvent()` | Async, fire-and-forget |
| `awaitEvent()` | Sync, blocking |

**Rejected:** `awaitEvent` reads as "wait for an event to occur" (observer pattern) rather than "process this event synchronously." Confusing.

---

## Recommendation

**Go with Option B: `triggerWorkflows()`** — single unified API where the trigger's registered mode determines whether the call blocks.

The rationale: sync vs. async is a property of the **trigger**, not of the **call site**. The dashboard team registers `dashboard.onCreate` as sync because they need pre-save interception. Every `triggerWorkflows('dashboard.onCreate', ...)` call should block — there's no scenario where the same trigger is sometimes sync and sometimes async within the same code path. Making the caller redundantly specify the mode (as in Option A) adds ceremony without value.

---

## Problem 2: `sync: true` Belongs on the Trigger, Not the Workflow

### Current Proposal (workflow-level)

```yaml
# Workflow YAML — the workflow author decides sync/async
triggers:
  - type: dashboard.onCreate
    sync: true               # Workflow author sets this
```

**Problems:**
- The workflow author doesn't own the code path being blocked. The dashboard team does.
- A workflow author could add `sync: true` to a trigger the owning team never designed for blocking execution.
- Guardrails (timeout cap, failure policy) have to be validated against the trigger's constraints at save time — an extra validation step that wouldn't be needed if the mode were declared at registration.
- If two workflows subscribe to the same trigger, one with `sync: true` and one without, the engine must handle mixed-mode execution — added complexity.

### Proposed: Mode on the Trigger Definition

The team registering the trigger declares the mode. All workflows subscribing to that trigger run in the declared mode.

```typescript
// Sync trigger — triggerWorkflows() ALWAYS blocks, ALWAYS returns result
workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.onCreate',
  mode: 'sync',              // not "allowed: true" — it IS sync
  maxTimeout: '10s',
  failurePolicy: 'open',
  eventSchema: z.object({ title: z.string(), description: z.string() }),
});

// Async trigger — triggerWorkflows() ALWAYS returns void, never blocks
workflowsExtensions.registerTriggerDefinition({
  id: 'alert.created',
  mode: 'async',             // fire-and-forget
  eventSchema: z.object({ alertId: z.string() }),
});
```

**What changes in the workflow YAML:**

```yaml
# Before (current proposal)
triggers:
  - type: dashboard.onCreate
    sync: true                    # workflow author's choice

# After (trigger-level mode)
triggers:
  - type: dashboard.onCreate      # mode is inherited from trigger definition
                                   # no sync: true needed — dashboard.onCreate IS sync
```

The workflow YAML becomes simpler. The author just subscribes to a trigger; the trigger's behavior is defined by the team that owns it.

### Full Trigger Registration API

```typescript
export interface TriggerDefinitionConfig {
  id: string;
  eventSchema: ZodSchema;

  // Execution mode — determined by the registering team
  mode: 'sync' | 'async';            // default: 'async'

  // Sync-only settings (ignored when mode: 'async')
  maxTimeout?: string;                // e.g., '10s' — max allowed workflow timeout
  failurePolicy?: 'open' | 'closed'; // default: 'open'
  maxConcurrentWorkflows?: number;    // default: 5
}
```

### How the Engine Uses This

```
triggerWorkflows('dashboard.onCreate', payload)
    │
    ▼
Resolve trigger definition
    │
    ├── mode: 'async'  ──→  scheduleWorkflow() via Task Manager  ──→  return void
    │
    └── mode: 'sync'   ──→  executeWorkflow() directly  ──→  return WorkflowResult
                              │
                              ├── timeout enforced: min(workflow.timeout, trigger.maxTimeout)
                              ├── circuit breaker check
                              └── sequential execution (up to maxConcurrentWorkflows)
```

### Why This Is Better

| Aspect | `sync: true` on workflow | `mode` on trigger |
|--------|--------------------------|-------------------|
| **Who decides** | Workflow author (doesn't own the code path) | Trigger registrar (owns the code path) |
| **Mixed mode** | Possible — engine must handle sync + async subscribers on same trigger | Impossible — all subscribers run in the same mode |
| **Guardrail enforcement** | Extra save-time validation: "is this workflow's timeout ≤ trigger's cap?" | Simpler: trigger defines the constraints, engine enforces them |
| **Workflow YAML** | Must specify `sync: true` | No extra config — mode is implicit from trigger |
| **Code path clarity** | Dashboard calls `triggerWorkflows()` and... maybe it blocks? depends on subscribers | Dashboard calls `triggerWorkflows()` on a sync trigger — it always blocks |
| **Accidental sync** | Author adds `sync: true` on a trigger not designed for it | Can't happen — the trigger's mode is fixed |

### Edge Case: What If a Team Wants Both?

Example: Dashboard team wants `dashboard.onCreate` to be sync (pre-save validation), but also wants async workflows to run post-save (e.g., audit logging).

**Solution:** Register two separate triggers:

```typescript
// Pre-save: sync, blocks the request
workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.beforeCreate',
  mode: 'sync',
  maxTimeout: '10s',
  failurePolicy: 'open',
  eventSchema: z.object({ title: z.string(), description: z.string() }),
});

// Post-save: async, fire-and-forget
workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.afterCreate',
  mode: 'async',
  eventSchema: z.object({ dashboardId: z.string(), title: z.string() }),
});
```

```typescript
// In create.ts
const wfResult = await workflowsClient.triggerWorkflows('dashboard.beforeCreate', dashboardEvent);
// ... handle result, save ...
workflowsClient.triggerWorkflows('dashboard.afterCreate', { dashboardId: saved.id, title: saved.title });
```

This is actually **cleaner** — the naming (`beforeCreate` vs `afterCreate`) makes the lifecycle explicit.

---

## Summary of Proposed Changes

| What | Before | After |
|------|--------|-------|
| **API method name** | `emitEvent()` | `triggerWorkflows()` |
| **Sync/async decision** | `sync: true` on workflow YAML trigger | `mode: 'sync' \| 'async'` on trigger registration |
| **Guardrail config** | Proposed in `08-sync-workflow-guardrails.md` as per-trigger registration options | Same, but now a required part of trigger registration (not separate) |
| **Workflow YAML** | `triggers: [{ type: dashboard.onCreate, sync: true }]` | `triggers: [{ type: dashboard.beforeCreate }]` (mode inherited) |
| **Return type** | `EmitEventResult \| void` | `WorkflowResult \| void` (same shape, better name) |

### Proposed `WorkflowResult` Type

```typescript
export interface WorkflowResult {
  status: 'completed' | 'failed' | 'timeout';
  outputs?: Record<string, unknown>;
  error?: { message: string; reason?: string };
}

export interface WorkflowsClient {
  /**
   * Trigger all workflows subscribed to the given trigger.
   *
   * - If the trigger is registered with mode: 'sync', this call blocks until
   *   all subscribed workflows complete and returns the aggregated result.
   * - If the trigger is registered with mode: 'async' (or no trigger is
   *   registered), workflows are scheduled via Task Manager and this returns void.
   */
  triggerWorkflows(
    triggerId: string,
    payload: Record<string, unknown>
  ): Promise<WorkflowResult | void>;
}
```

---

## Open Questions

1. **Backward compatibility**: We already have `emitEvent` in production. Do we keep it as a deprecated alias, or make a clean break?

2. **Naming bikeshed**: `triggerWorkflows` vs `invokeWorkflows` vs `runWorkflows`? The team should weigh in. Key criteria: doesn't imply async, maps to YAML vocabulary, reads well at the call site.

3. **before/after naming convention**: If teams split triggers into `beforeX` and `afterX`, should we enforce this as a convention? It makes lifecycle phases explicit but adds more trigger IDs.

4. **Can a trigger's mode change after registration?** If Dashboard ships `dashboard.beforeCreate` as sync and later wants to make it async, what's the migration path for existing workflows?
