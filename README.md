# Workflow Examples & Snippets — Design Validation

> **Purpose**: These are *proposal* examples for team review, not production code. They demonstrate how workflows can be used for guardrails, anonymization, PII reduction, and other cross-cutting concerns across Kibana systems (Agent Builder, Dashboards, Cases, etc.). The goal is to stress-test the workflow architecture before writing implementation code.

## Legend

Throughout the YAML examples, comments indicate what exists today vs. what is proposed:

- `# [EXISTS]` — This syntax/feature works today in the workflow engine
- `# [PROPOSED]` — This syntax/feature does not exist yet and is part of the design proposal
- `# [PROPOSED STEP]` — A new step type that would need to be implemented

## Key Concepts

### 1. Blocking (sync) `emitEvent`

**Today**: `emitEvent()` is always fire-and-forget. It resolves matching workflows and schedules them via Task Manager. It returns `void`.

**Proposed**: When a workflow's trigger specifies `sync: true`, `emitEvent` blocks the caller until the workflow completes, and returns the result (outputs, status, error). The trigger event handler uses `executeWorkflow` (direct execution) instead of `scheduleWorkflow` (Task Manager).

```yaml
# In the workflow YAML
triggers:
  - type: dashboard.onCreate
    sync: true              # [PROPOSED] — tells emitEvent to block
```

```typescript
// In the dashboard plugin code
const result = await workflowsClient.emitEvent('dashboard.onCreate', dashboardEvent);
// result is EmitEventResult | void
// - void when no sync workflows are subscribed
// - EmitEventResult when a sync workflow ran
```

### 2. By-ref vs. By-value

Two patterns for how workflows communicate changes back to the caller:

#### By-ref (proposed — shift-right to workflows)

The workflow modifies the event object directly via `event.mutate`. The caller gets back the modified object without needing to know the output contract.

```typescript
const caseObj = { title: 'My Case', description: 'Some PII here: 555-12-3456' };
await workflowsClient.emitEvent('cases.onCreate', caseObj);
// caseObj.description is now 'Some PII here: ***-**-****' (modified by workflow)
saveCaseToES(caseObj);
```

#### By-value (exists today)

The workflow returns data via `workflow.output`. The caller must know the output schema and copy fields back.

```typescript
const caseObj = { title: 'My Case', description: 'Some PII here: 555-12-3456' };
const result = await workflowsClient.emitEvent('cases.onCreate', caseObj);
if (result?.outputs) {
  caseObj.title = result.outputs.title;             // caller copies each field
  caseObj.description = result.outputs.description;
}
saveCaseToES(caseObj);
```

**Trade-off**: By-ref is simpler for the caller (especially with many fields). By-value gives the caller explicit control over what changes are accepted. See examples `04` (by-ref) and `05` (by-value) for a side-by-side comparison.

### 3. Workflow contracts via `workflow.output` and `workflow.fail`

- **`workflow.output`** — Returns structured data to the caller. Validated against the workflow's `outputs` schema. Status: `completed`.
- **`workflow.fail`** — Fails the workflow with a structured error (`message`, `reason`). The caller sees `status: 'failed'` and the error object.

These exist today and are used by the guardrail and by-value examples.

### 4. Event-driven triggers (custom)

Teams register custom trigger types via `registerTriggerDefinition()` in their plugin's `setup()`. Each trigger has an `eventSchema` that defines the object shape the workflow receives. See `integration-guide.md` for code snippets.

## Examples

| File | Use Case | Pattern | Sync? |
|------|----------|---------|-------|
| `01-guardrails-sync.yaml` | AI guardrails for chat rounds | `workflow.fail` for blocking | Yes |
| `02-pii-anonymization-before.yaml` | PII anonymization before LLM inference | Ephemeral state + `event.mutate` | Yes |
| `03-pii-anonymization-after.yaml` | PII de-anonymization after LLM response | Ephemeral state retrieval | Yes |
| `04-dashboard-oncreate-byref.yaml` | PII reduction on dashboard fields | By-ref via `event.mutate` | Yes |
| `05-dashboard-oncreate-byvalue.yaml` | Same, but by-value with `workflow.output` | By-value via outputs | Yes |
| `06-alert-enrichment.yaml` | Alert enrichment with AI summary | Async event-driven | No |
| `07-cases-guardrail.yaml` | PII blocking on case comments | `workflow.fail` for blocking | Yes |

## Architecture

### Sync (blocking) flow

```
  System Component                    Workflow Engine
  ================                    ===============
  
  1. Build event object
     (trimmed-down DTO,
      not the full saved object)
                    ─── emitEvent(triggerId, eventObj) ───>
                                                            2. Resolve matching workflows
                                                            3. Check sync: true on trigger
                                                            4. executeWorkflow (direct, not TM)
                                                            5. Run steps
                                                               - May modify eventObj (by-ref)
                                                               - May call workflow.output (by-value)
                                                               - May call workflow.fail (block/error)
                    <── EmitEventResult { status, outputs, error } ──
  6. Check result:
     - status === 'failed' → throw/reject
     - status === 'completed' → use modified eventObj
       or copy from outputs
  7. Continue with create/save
```

### Async (fire-and-forget) flow — unchanged from today

```
  System Component                    Workflow Engine
  ================                    ===============
  
  1. Something happens
     (alert fires, etc.)
                    ─── emitEvent(triggerId, payload) ───>
                                                            2. Resolve matching workflows
                                                            3. scheduleWorkflow (Task Manager)
                    <── void ──
  3. Continue immediately                                   4. Workflow runs async via TM
```

## Open Questions

1. **Who decides sync vs async?** The proposal puts `sync: true` on the workflow's trigger definition. But what if the emitter wants to control it? (e.g., dashboard team always wants sync for `onCreate` but async for `onView`)

2. **Multiple sync workflows**: If 3 workflows subscribe to `dashboard.onCreate` with `sync: true`, do they run sequentially? In parallel? What if one fails — do the others still run?

3. **Timeouts**: What is the max execution time for a sync workflow? This blocks an HTTP request. Guardrails should be fast (< 5s), but PII reduction with AI could be slower.

4. **By-ref implementation**: How does `event.mutate` actually work under the hood? Does the execution engine pass a mutable reference, or does it merge the mutation result back into the caller's object after completion?

5. **Trigger-implied inputs**: When a workflow subscribes to `dashboard.onCreate`, should the event schema automatically become the workflow's inputs? Or should the workflow author still declare inputs explicitly?

6. **AB migration path**: Agent Builder currently uses a custom loop + `BeforeAgentWorkflowOutput` contract. How do we migrate to the standard `emitEvent` model without breaking existing workflows that rely on `abort`/`abort_message`/`new_prompt`?

## Related Files

- `integration-guide.md` — TypeScript code snippets showing how each team integrates
- Existing trigger docs: `src/platform/plugins/shared/workflows_extensions/dev_docs/TRIGGERS.md`
- Current AB guardrail code: `x-pack/platform/plugins/shared/agent_builder/server/hooks/agent_workflows/`
- Dashboard create: `src/platform/plugins/shared/dashboard/server/api/create/create.ts`
- Cases create: `x-pack/platform/plugins/shared/cases/server/client/cases/create.ts`
