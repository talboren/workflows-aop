# Guardrails for Sync (Blocking) Workflows

> **Why this matters**: A sync workflow blocks the calling HTTP request — a dashboard save, a case creation, a chat round. A misbehaving workflow doesn't just hurt itself; it degrades the experience for **every user** hitting that code path. This document proposes the guardrails needed to make sync workflows safe by default.

## The Threat Model

| Threat | Example | Impact |
|--------|---------|--------|
| **Slow workflow** | ES query on a growing index; no timeout configured | Dashboard save takes 60s instead of 200ms |
| **Stuck workflow** | Bug causes infinite loop or deadlock | HTTP request hangs until server-side timeout (typically 2 min) |
| **Expensive workflow** | AI guardrail calls external LLM on every keystroke | Latency spikes, LLM rate limits, cost blowup |
| **Cascade failure** | Sync workflow emits another sync event | Chain of blocking calls, exponential latency |
| **Malicious/broken workflow** | `workflow.fail` always fires, blocking all case creation | Feature is completely broken for all users in the space |
| **Resource exhaustion** | ES search returns 10M docs, OOM during sync execution | Kibana process crash, all users affected |

---

## Proposed Guardrails

### 1. Hard Timeout for Sync Workflows

**The most important guardrail.** Sync workflows MUST have a maximum execution time enforced by the engine, independent of what the workflow author configures.

```
Effective timeout = min(workflow.settings.timeout, SYNC_MAX_TIMEOUT)
```

| Setting | Value | Notes |
|---------|-------|-------|
| `SYNC_MAX_TIMEOUT` (platform default) | **30s** | Configurable in `kibana.yml` |
| Recommended per-trigger override | 5–10s | Trigger registrars can set a tighter cap |
| Step-level timeout | Inherited from existing `timeout` property | Already exists today |

**Behavior when timeout fires:**
- Workflow execution is **aborted** (same as existing `TimeoutError` handling).
- `emitEvent` returns `{ status: 'timeout', error: { message: '...' } }`.
- **The caller proceeds** (fail-open) — see section 3.

**Why 30s?** Kibana's default HTTP `server.requestTimeout` is 120s. A sync workflow at 30s leaves room for the caller's own logic, serialization, and response. For latency-sensitive paths (chat, dashboard save), trigger registrars should cap at 5–10s.

> **What exists today:** The engine already has workflow-level and step-level timeout infrastructure (`EnterWorkflowTimeoutZoneNodeImpl`, `EnterStepTimeoutZoneNodeImpl`). For sync execution, the engine would wrap the entire workflow in a timeout zone capped at `SYNC_MAX_TIMEOUT`.

### 2. Trigger-Level Timeout Cap

When a team registers a trigger, they declare the maximum allowed timeout for sync workflows subscribing to it:

```typescript
workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.onCreate',
  name: 'Dashboard Create',
  eventSchema: { ... },
  sync: {
    allowed: true,
    maxTimeout: '10s',       // Workflows subscribing with sync: true cannot exceed this
    failurePolicy: 'open',   // 'open' = proceed on failure/timeout, 'closed' = block
  },
});
```

**At workflow save time**, when a workflow has `sync: true` for trigger `dashboard.onCreate`, the engine validates:
- `workflow.settings.timeout <= trigger.sync.maxTimeout` (or uses the trigger cap if not set)
- If the workflow specifies a timeout higher than the trigger cap → **save rejected with validation error**

This prevents a workflow author from setting `timeout: 1h` on a sync trigger that's supposed to be fast.

### 3. Fail-Open vs. Fail-Closed (Configurable per Trigger)

The trigger registrar decides what happens when a sync workflow fails or times out:

| Policy | Behavior | Use case |
|--------|----------|----------|
| **fail-open** (default) | Timeout/error → caller proceeds as if no workflow ran; event payload unchanged | Dashboard save, most CRUD operations |
| **fail-closed** | Timeout/error → caller receives the error and must handle it (e.g., reject the operation) | Security guardrails, compliance-critical checks |

```yaml
# fail-open: dashboard save succeeds even if the workflow is broken
triggers:
  - type: dashboard.onCreate
    sync: true
    # trigger's failurePolicy: 'open' → if this workflow times out, save proceeds

# fail-closed: chat round is blocked if guardrail workflow fails
triggers:
  - type: agent-builder.chatRound
    sync: true
    # trigger's failurePolicy: 'closed' → if this workflow times out, chat is blocked
```

**Rationale:** Most teams want sync workflows as a "best-effort enhancement" — they don't want a broken workflow to take down their feature. Security/compliance teams want the opposite. Let the trigger registrar decide.

### 4. Circuit Breaker

Track success/failure/timeout rates per workflow. Auto-disable workflows that repeatedly fail.

| Parameter | Default | Notes |
|-----------|---------|-------|
| Window | Last 20 executions | Rolling window |
| Failure threshold | 50% failures in window | Configurable |
| Consecutive failure threshold | 5 consecutive failures | Whichever trips first |
| Cooldown | 5 min | Auto-re-enable after cooldown, then re-evaluate |
| Max trips before permanent disable | 3 | After 3 cooldown cycles still failing → permanently disabled, admin notified |

**When the circuit breaker trips:**
1. Workflow is marked `suspended` (new status, distinct from `disabled`).
2. `emitEvent` skips the workflow entirely (returns `void` for that subscription).
3. An audit log entry is written: `workflow.circuit_breaker_tripped`.
4. Notification sent to workflow owner (in-app notification or email if configured).
5. After cooldown, the circuit is half-open: the next matching event runs the workflow; if it succeeds, the circuit closes. If it fails, the circuit re-opens.

**Why this matters:** Without a circuit breaker, a broken sync workflow on `cases.createComment` would block **every comment in every case** until someone manually disables it. The circuit breaker limits the blast radius to a few failed attempts.

### 5. Save-Time Validation for Sync Workflows

When a workflow with a `sync: true` trigger is saved, apply stricter validation than for async workflows:

| Rule | Reason |
|------|--------|
| `timeout` is required (or trigger's `maxTimeout` is used) | Prevent unbounded execution |
| No `workflow.executeAsync` step | Sync workflows shouldn't spawn fire-and-forget children |
| No `wait` step | `wait` steps are designed for long-running async flows |
| ES queries must have `size` ≤ 1000 | Prevent unbounded result sets in a blocking context |
| `foreach` must have `max-iterations` ≤ 100 | Prevent long loops blocking the caller |
| Total step count ≤ 20 | Complex workflows belong in async triggers |
| No recursive `emitEvent` (static analysis) | Prevent sync → sync → sync chains |

These are **warnings at save time**, not hard blocks (except timeout). The author can override with acknowledgment, but the defaults guide toward safe patterns.

### 6. Sync Execution Chain Depth Limit

> **Exists today:** The `EventChainContext` already tracks event chain depth and enforces a cap to prevent infinite loops.

For sync workflows specifically, the chain depth limit should be **stricter**:

| Context | Max depth | Notes |
|---------|-----------|-------|
| Async event chains (today) | 10 | Existing behavior |
| Sync event chains (proposed) | **2** | A sync workflow can trigger at most 1 more sync workflow |

If a sync workflow at depth 2 tries to `emitEvent` another sync trigger, the call is **silently skipped** (logged as a warning) and returns `void`. This prevents the exponential latency cascade.

### 7. Concurrency Limits per Trigger Type

When multiple sync workflows subscribe to the same trigger:

| Scenario | Behavior |
|----------|----------|
| 1 sync workflow | Direct execution, return result |
| N sync workflows (N ≤ max) | Execute **sequentially** (ordered by creation date); first `workflow.fail` short-circuits the rest |
| N sync workflows (N > max) | Only the first `max` run; the rest are skipped with warning |

```typescript
// Trigger registration
workflowsExtensions.registerTriggerDefinition({
  id: 'cases.createComment',
  sync: {
    allowed: true,
    maxTimeout: '10s',
    maxConcurrentWorkflows: 3,   // At most 3 sync workflows per event
    failurePolicy: 'open',
  },
});
```

**Default `maxConcurrentWorkflows`: 5.** This prevents a scenario where 50 workflows all subscribe to `dashboard.onCreate` with `sync: true` and each takes 5s → 250s total blocking time.

### 8. Resource Limits (Inherited from Existing Infrastructure)

Sync workflows inherit all existing engine guardrails:

| Guardrail | Exists today? | Notes for sync context |
|-----------|--------------|----------------------|
| Step output size limit (`max-step-size`) | Yes | Prevents OOM from large step outputs |
| Layer 1 pre-emptive I/O limits | Yes | ES `maxResponseSize`, HTTP `maxContentLength` |
| Layer 2 generic output check | Yes | Base class enforcement on all steps |
| Concurrency manager | Yes | `concurrency.strategy: drop` is useful for sync workflows |
| Event chain depth cap | Yes | Tightened for sync (see section 6) |
| Step-level timeout | Yes | Each step can have its own timeout |
| Workflow-level timeout | Yes | Capped by `SYNC_MAX_TIMEOUT` for sync |

### 9. Admin Controls

| Control | Description |
|---------|-------------|
| **Global sync kill switch** | `workflows.sync.enabled: false` in `kibana.yml` — disables all sync workflow execution; `emitEvent` always returns `void` |
| **Per-trigger sync disable** | API to disable sync execution for a specific trigger type |
| **Per-workflow force-disable** | Admin can disable a specific workflow (already exists) |
| **Sync execution dashboard** | Kibana dashboard showing: p50/p95/p99 latency per trigger, timeout rate, circuit breaker trips, top slow workflows |
| **Audit log** | All sync executions logged with: trigger type, workflow ID, duration, outcome (completed/failed/timeout/circuit-breaker-skipped) |

### 10. Observability & Alerting

| Signal | Source | Alert threshold |
|--------|--------|----------------|
| Sync execution p95 latency | APM span on `emitEvent` | > 5s for any trigger type |
| Timeout rate | Workflow execution logs (`status: timeout`) | > 10% in 15 min window |
| Circuit breaker trips | Audit log (`workflow.circuit_breaker_tripped`) | Any occurrence |
| `workflow.fail` rate | Workflow execution logs | > 50% for any single workflow |
| Sync execution count | Workflow execution logs | Spike detection (> 3x baseline) |

---

## Architecture: How It Fits Together

```
                 ┌───────────────────────────────────────────────────────┐
                 │                   emitEvent() call                    │
                 └───────────────────────┬───────────────────────────────┘
                                         │
                                         ▼
                 ┌───────────────────────────────────────────────────────┐
                 │  1. Resolve matching workflows for trigger            │
                 │  2. Filter to sync: true workflows                    │
                 │  3. Check chain depth (max 2 for sync)                │
                 │  4. Check circuit breaker status per workflow          │
                 │     → tripped? skip workflow, return void             │
                 └───────────────────────┬───────────────────────────────┘
                                         │
                                         ▼
                 ┌───────────────────────────────────────────────────────┐
                 │  For each eligible sync workflow (up to max):         │
                 │                                                       │
                 │  ┌─────────────────────────────────────────────────┐  │
                 │  │  executeWorkflow() with timeout =               │  │
                 │  │    min(workflow.timeout, trigger.maxTimeout,     │  │
                 │  │        SYNC_MAX_TIMEOUT)                        │  │
                 │  │                                                 │  │
                 │  │  ┌── Step 1 ──┐  ┌── Step 2 ──┐  ┌── ... ──┐  │  │
                 │  │  │ timeout    │  │ size limit  │  │         │  │  │
                 │  │  │ size limit │  │ abort ctrl  │  │         │  │  │
                 │  │  └────────────┘  └────────────┘  └─────────┘  │  │
                 │  └─────────────────────────────────────────────────┘  │
                 │                                                       │
                 │  Outcomes:                                             │
                 │  • completed → return { status, outputs }             │
                 │  • failed (workflow.fail) → return { status, error }  │
                 │  • timeout → return { status: 'timeout', error }      │
                 │                → update circuit breaker                │
                 └───────────────────────┬───────────────────────────────┘
                                         │
                                         ▼
                 ┌───────────────────────────────────────────────────────┐
                 │  Caller receives EmitEventResult:                     │
                 │                                                       │
                 │  if failurePolicy === 'open':                         │
                 │    timeout/error → proceed (use original event)       │
                 │  if failurePolicy === 'closed':                       │
                 │    timeout/error → propagate to caller (reject op)    │
                 └───────────────────────────────────────────────────────┘
```

---

## Summary: Defense in Depth

```
Layer          What it prevents                        New or existing?
─────          ─────────────────────                    ────────────────
Timeout        Slow/stuck workflows blocking requests   Existing infra, new sync cap
Circuit break  Broken workflows repeatedly blocking     NEW
Save-time      Dangerous patterns (no timeout, waits)   NEW
Fail-open      One broken workflow breaking a feature    NEW (configurable)
Chain depth    Sync → sync → sync cascade               Existing infra, tighter limit
Concurrency    50 sync workflows all blocking at once   NEW
Size limits    OOM during sync execution                Existing infra
Admin controls Emergency kill switch                    NEW
Observability  Catching problems before users notice    NEW (dashboards, alerts)
```

---

## Open Questions

1. **Should the circuit breaker be per-workflow or per-workflow-per-trigger?** A workflow might work fine for one trigger but fail for another (different event schema).

2. **Sequential vs. parallel execution of multiple sync workflows:** Sequential is safer (predictable ordering, early termination on `workflow.fail`), but parallel would reduce total latency. Start with sequential?

3. **Who can create sync workflows?** Should `sync: true` require elevated permissions (e.g., only space admins)? This would prevent regular users from accidentally blocking system operations.

4. **Quota per space:** Should there be a maximum number of sync workflows per trigger per space? E.g., "at most 3 sync workflows on `dashboard.onCreate` in the Marketing space."

5. **Gradual rollout:** Should we start with sync execution behind a feature flag (`workflows.sync.enabled: false` by default) and require explicit opt-in during the beta phase?

6. **Timeout UX:** When a sync workflow times out and `failurePolicy: open`, should the user see a toast notification like "A workflow timed out but your save succeeded"? Or silently proceed?
