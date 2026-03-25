# Integration Guide — How Teams Plug Into Workflows

> **Purpose**: This document shows the TypeScript code each team would write to integrate with the workflow system. It covers trigger registration, event emission, and result handling. All code is pseudocode/diffs against the actual codebase.

## Table of Contents

1. [The `emitEvent` API Extension](#1-the-emitevent-api-extension)
2. [Integration Pattern (3 steps)](#2-integration-pattern)
3. [Dashboard Integration](#3-dashboard-integration)
4. [Cases Integration](#4-cases-integration)
5. [Agent Builder Integration](#5-agent-builder-integration)
6. [The Trigger Event Handler Change](#6-the-trigger-event-handler-change)

---

## 1. The `emitEvent` API Extension

### Today (`workflows_extensions/server/types.ts`)

```typescript
// emitEvent always returns void — fire-and-forget
export type EmitEventFn = (params: EmitEventParams) => Promise<void>;

export interface EmitEventParams {
  triggerId: string;
  spaceId: string;
  payload: Record<string, unknown>;
  request: KibanaRequest;
}
```

### Proposed

```typescript
// emitEvent returns a result for sync workflows, void for async-only
export interface EmitEventResult {
  status: 'completed' | 'failed';
  outputs?: Record<string, unknown>;
  error?: { message: string; reason?: string };
}

export type EmitEventFn = (params: EmitEventParams) => Promise<EmitEventResult | void>;
// Returns void when:
//   - No workflows are subscribed to this trigger
//   - All subscribed workflows are async (no sync: true)
// Returns EmitEventResult when:
//   - At least one subscribed workflow has sync: true on its trigger
```

### WorkflowsClient (request-scoped) change

```typescript
// Today
export interface WorkflowsClient {
  emitEvent(triggerId: string, payload: Record<string, unknown>): Promise<void>;
}

// Proposed
export interface WorkflowsClient {
  emitEvent(triggerId: string, payload: Record<string, unknown>): Promise<EmitEventResult | void>;
}
```

---

## 2. Integration Pattern

Every team follows the same 3 steps:

### Step 1: Define the trigger (common)

Create a shared trigger definition in your plugin's `common/` directory:

```typescript
// my-plugin/common/triggers/on_create.ts
import { z } from '@kbn/zod/v4';
import type { CommonTriggerDefinition } from '@kbn/workflows-extensions/common';

export const MY_CREATE_TRIGGER_ID = 'my-plugin.onCreate' as const;

export const myCreateEventSchema = z.object({
  title: z.string().describe('The object title'),
  description: z.string().optional().describe('The object description'),
});

export type MyCreateEvent = z.infer<typeof myCreateEventSchema>;

export const commonMyCreateTriggerDefinition: CommonTriggerDefinition = {
  id: MY_CREATE_TRIGGER_ID,
  eventSchema: myCreateEventSchema,
};
```

### Step 2: Register in plugin setup (server + public)

```typescript
// my-plugin/server/plugin.ts — setup()
plugins.workflowsExtensions.registerTriggerDefinition(commonMyCreateTriggerDefinition);

// my-plugin/public/plugin.ts — setup()
plugins.workflowsExtensions.registerTriggerDefinition({
  ...commonMyCreateTriggerDefinition,
  title: i18n.translate('myPlugin.onCreate.title', { defaultMessage: 'On Create' }),
  description: i18n.translate('myPlugin.onCreate.description', {
    defaultMessage: 'Fires when a new object is created',
  }),
  icon: React.lazy(() =>
    import('@elastic/eui/es/components/icon/assets/document').then(({ icon }) => ({ default: icon }))
  ),
  documentation: {
    examples: [
      `triggers:\n  - type: ${MY_CREATE_TRIGGER_ID}\n    sync: true`,
    ],
  },
});
```

### Step 3: Emit the event in your code path

```typescript
// In the code that creates the object
const client = (await context.workflows).getWorkflowsClient();
const event = { title: obj.title, description: obj.description };

const result = await client.emitEvent(MY_CREATE_TRIGGER_ID, event);

if (result?.status === 'failed') {
  throw new Error(result.error?.message ?? 'Workflow rejected the operation');
}

// By-ref: event.title and event.description may have been modified by the workflow
obj.title = event.title;
obj.description = event.description;

// Continue with save
await saveObject(obj);
```

---

## 3. Dashboard Integration

### File: `src/platform/plugins/shared/dashboard/server/api/create/create.ts`

#### Today

```typescript
export async function create(
  requestCtx: RequestHandlerContext,
  dashboardStateSchema: ReturnType<typeof getDashboardStateSchema>,
  createBody: DashboardCreateRequestBody,
  createParams?: DashboardCreateRequestParams,
  isDashboardAppRequest: boolean = false
): Promise<DashboardCreateResponseBody> {
  const { core } = await requestCtx.resolve(['core']);
  const { access_control: accessControl, ...restOfData } = createBody;

  const { attributes: soAttributes, references: soReferences, error: transformInError } =
    transformDashboardIn(restOfData, isDashboardAppRequest);
  if (transformInError) throw Boom.badRequest(`Invalid data. ${transformInError.message}`);

  // ... access control ...

  const savedObject = await core.savedObjects.client.create<DashboardSavedObjectAttributes>(
    DASHBOARD_SAVED_OBJECT_TYPE,
    soAttributes,
    { references: soReferences, /* ... */ }
  );

  return getDashboardCRUResponseBody(savedObject, 'create', dashboardStateSchema, isDashboardAppRequest);
}
```

#### Proposed (diff)

```diff
 export async function create(
   requestCtx: RequestHandlerContext,
   dashboardStateSchema: ReturnType<typeof getDashboardStateSchema>,
   createBody: DashboardCreateRequestBody,
   createParams?: DashboardCreateRequestParams,
   isDashboardAppRequest: boolean = false
 ): Promise<DashboardCreateResponseBody> {
-  const { core } = await requestCtx.resolve(['core']);
+  const { core, workflows } = await requestCtx.resolve(['core', 'workflows']);
   const { access_control: accessControl, ...restOfData } = createBody;

   const { attributes: soAttributes, references: soReferences, error: transformInError } =
     transformDashboardIn(restOfData, isDashboardAppRequest);
   if (transformInError) throw Boom.badRequest(`Invalid data. ${transformInError.message}`);

+  // --- NEW: Run pre-create workflows (sync if any are subscribed) ---
+  const dashboardEvent = {
+    title: soAttributes.title ?? '',
+    description: soAttributes.description ?? '',
+  };
+  const workflowsClient = workflows.getWorkflowsClient();
+  const wfResult = await workflowsClient.emitEvent('dashboard.onCreate', dashboardEvent);
+  if (wfResult?.status === 'failed') {
+    throw Boom.badRequest(wfResult.error?.message ?? 'Workflow rejected dashboard creation');
+  }
+  // By-ref: event fields may have been modified by the workflow
+  soAttributes.title = dashboardEvent.title;
+  soAttributes.description = dashboardEvent.description;
+  // --- END NEW ---

   // ... access control ...

   const savedObject = await core.savedObjects.client.create<DashboardSavedObjectAttributes>(
     DASHBOARD_SAVED_OBJECT_TYPE,
     soAttributes,
     { references: soReferences, /* ... */ }
   );

   return getDashboardCRUResponseBody(savedObject, 'create', dashboardStateSchema, isDashboardAppRequest);
 }
```

### Trigger registration

```typescript
// src/platform/plugins/shared/dashboard/server/plugin.ts — setup()
import { z } from '@kbn/zod/v4';

plugins.workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.onCreate',
  eventSchema: z.object({
    title: z.string(),
    description: z.string().optional(),
  }),
});
```

---

## 4. Cases Integration

### File: `x-pack/platform/plugins/shared/cases/server/client/cases/create.ts`

#### Today (simplified)

```typescript
export const create = async (
  data: CasePostRequest,
  clientArgs: CasesClientArgs,
  casesClient: CasesClient
): Promise<Case> => {
  // ... validation, auth ...
  const normalizedCase = normalizeCreateCaseRequest(query, customFieldsConfiguration);

  const newCase = await caseService.createCase({
    attributes: transformNewCase({ ...normalizedCase, /* ... */ }),
    id: savedObjectID,
  });
  // ... user actions, notifications ...
  return flattenCaseSavedObject({ savedObject: newCase, /* ... */ });
};
```

#### Proposed (diff)

```diff
 export const create = async (
   data: CasePostRequest,
   clientArgs: CasesClientArgs,
-  casesClient: CasesClient
+  casesClient: CasesClient,
+  workflowsClient?: WorkflowsClient
 ): Promise<Case> => {
   // ... validation, auth ...
   const normalizedCase = normalizeCreateCaseRequest(query, customFieldsConfiguration);

+  // --- NEW: Run pre-create workflows ---
+  if (workflowsClient) {
+    const caseEvent = {
+      title: normalizedCase.title,
+      description: normalizedCase.description,
+    };
+    const wfResult = await workflowsClient.emitEvent('cases.onCreate', caseEvent);
+    if (wfResult?.status === 'failed') {
+      throw createCaseError({
+        message: wfResult.error?.message ?? 'Workflow rejected case creation',
+        logger,
+      });
+    }
+    normalizedCase.title = caseEvent.title;
+    normalizedCase.description = caseEvent.description;
+  }
+  // --- END NEW ---

   const newCase = await caseService.createCase({
     attributes: transformNewCase({ ...normalizedCase, /* ... */ }),
     id: savedObjectID,
   });
   // ... user actions, notifications ...
   return flattenCaseSavedObject({ savedObject: newCase, /* ... */ });
 };
```

### Case comment guardrail

Same pattern in `x-pack/platform/plugins/shared/cases/server/client/attachments/add.ts`:

```diff
+  // --- NEW: Run pre-comment workflows ---
+  if (workflowsClient) {
+    const commentEvent = {
+      caseId: addArgs.caseId,
+      comment: commentReq.comment,
+      owner: commentReq.owner,
+    };
+    const wfResult = await workflowsClient.emitEvent('cases.createComment', commentEvent);
+    if (wfResult?.status === 'failed') {
+      throw createCaseError({
+        message: wfResult.error?.message ?? 'Comment blocked by workflow',
+        logger,
+      });
+    }
+  }
+  // --- END NEW ---
```

### Trigger registration

```typescript
// x-pack/platform/plugins/shared/cases/server/plugin.ts — setup()
import { z } from '@kbn/zod/v4';

plugins.workflowsExtensions.registerTriggerDefinition({
  id: 'cases.onCreate',
  eventSchema: z.object({
    title: z.string(),
    description: z.string().optional(),
  }),
});

plugins.workflowsExtensions.registerTriggerDefinition({
  id: 'cases.createComment',
  eventSchema: z.object({
    caseId: z.string(),
    comment: z.string(),
    owner: z.string(),
  }),
});
```

---

## 5. Agent Builder Integration

### Today: Brittle custom contract

Agent Builder currently uses a **manual loop** over workflow IDs with a **custom output contract** (`BeforeAgentWorkflowOutput`):

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/hooks/agent_workflows/types.ts
export interface BeforeAgentWorkflowOutput {
  abort?: boolean;
  abort_message?: string;
  new_prompt?: string;
}
```

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/hooks/agent_workflows/run_before_agent_workflows.ts
for (const workflowId of workflowIds) {
  const result = await executeWorkflow({
    workflowId,
    workflowParams: { prompt: currentNextInput.message ?? '' },
    request: context.request,
    spaceId,
    workflowApi,
    waitForCompletion: true,
  });

  if (!result.success) throw createWorkflowExecutionError(result.error, { workflow: workflowId });

  const execution = result.execution;
  if (execution.status === ExecutionStatus.FAILED) {
    throw createWorkflowExecutionError(execution.error_message, { workflow: ... });
  }

  const output: BeforeAgentWorkflowOutput = normalizeWorkflowOutput(execution.output);

  if (output.new_prompt) {
    currentNextInput = { ...currentNextInput, message: output.new_prompt };
  }

  if (output.abort || output.abort_message) {
    throw createWorkflowAbortedError(output.abort_message ?? '...', { workflow });
  }
}
```

**Problems with this approach:**
- AB manually manages the list of workflow IDs (from UI settings + agent config)
- The contract (`abort`/`abort_message`/`new_prompt`) is ad-hoc and not enforced by the workflow schema
- Other systems (Dashboard, Cases) can't reuse this pattern — they'd each build their own loop

### Proposed: Standard `emitEvent` model

#### Trigger registration

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/plugin.ts — setup()
import { z } from '@kbn/zod/v4';

plugins.workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.chatRound',
  eventSchema: z.object({
    message: z.string().describe('The user message for this chat round'),
    agentId: z.string().optional().describe('The agent handling this round'),
  }),
});

// For PII anonymization before/after inference (see examples 02/03)
plugins.workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.beforeInference',
  eventSchema: z.object({
    message: z.string(),
    roundId: z.string(),
    agentId: z.string().optional(),
  }),
});

plugins.workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.afterInference',
  eventSchema: z.object({
    response: z.string(),
    roundId: z.string(),
    agentId: z.string().optional(),
  }),
});
```

#### Rewritten `runBeforeAgentWorkflows`

```typescript
// PROPOSED replacement for run_before_agent_workflows.ts
export async function runBeforeAgentWorkflows({
  context,
  workflowsClient,  // WorkflowsClient from context.workflows
  logger,
}: RunBeforeAgentWorkflowsParams): Promise<void | HookHandlerResult<HookLifecycle.beforeAgent>> {

  // Build the event object — trimmed-down DTO, not the full round object
  const roundEvent = {
    message: context.nextInput.message ?? '',
    agentId: context.agentId,
  };

  // Single call replaces the entire for-loop over workflow IDs.
  // emitEvent resolves all subscribed workflows automatically.
  // If any has sync: true, it blocks and returns the result.
  const result = await workflowsClient.emitEvent('agent-builder.chatRound', roundEvent);

  if (result?.status === 'failed') {
    // workflow.fail was called — this replaces the old abort/abort_message check
    throw createWorkflowAbortedError(
      result.error?.message ?? 'Workflow blocked the chat round',
      { reason: result.error?.reason }
    );
  }

  // By-ref: if a workflow used event.mutate to change event.message,
  // roundEvent.message is now the modified version.
  // This replaces the old new_prompt handling.
  if (roundEvent.message !== context.nextInput.message) {
    return { nextInput: { ...context.nextInput, message: roundEvent.message } };
  }
}
```

**What changed:**
- No more manual workflow ID management — workflows subscribe via their trigger
- No more custom `BeforeAgentWorkflowOutput` contract — `workflow.fail` handles abort, `event.mutate` handles prompt rewriting
- No more for-loop — `emitEvent` resolves and runs all subscribed workflows
- Same pattern as Dashboard and Cases — one integration pattern for the whole platform

---

## 6. The Trigger Event Handler Change

### File: `workflows_management/server/event_driven/trigger_event_handler.ts`

The trigger event handler is where the async-vs-sync branching happens. Today it always calls `scheduleWorkflow` (Task Manager). The proposed change adds a branch for sync workflows.

#### Key change (pseudocode)

```typescript
// In createTriggerEventHandler, after resolving matching workflows:

const syncWorkflows = workflows.filter(w => hasSyncTrigger(w, triggerId));
const asyncWorkflows = workflows.filter(w => !hasSyncTrigger(w, triggerId));

// Async workflows: schedule via Task Manager (unchanged)
if (asyncWorkflows.length > 0) {
  await scheduleMatchingWorkflows(api, asyncWorkflows, spaceId, eventParams, ...);
}

// Sync workflows: execute directly and collect results
if (syncWorkflows.length > 0) {
  const results: EmitEventResult[] = [];
  for (const workflow of syncWorkflows) {
    const executionId = await api.runWorkflow(workflowForExecution, spaceId, eventContext, request);
    const execution = await pollForCompletion(executionId, spaceId);
    results.push(mapExecutionToResult(execution));

    // If any sync workflow fails, stop and return the failure
    if (execution.status === ExecutionStatus.FAILED) {
      return {
        status: 'failed',
        error: { message: execution.error_message, reason: execution.error_reason },
      };
    }
  }

  return {
    status: 'completed',
    outputs: mergeOutputs(results),
  };
}

// No sync workflows subscribed — return void (fire-and-forget)
return undefined;
```

#### Helper: check if a workflow has `sync: true` for this trigger

```typescript
function hasSyncTrigger(workflow: WorkflowDetailDto, triggerId: string): boolean {
  const triggers = workflow.definition?.triggers ?? [];
  return triggers.some(t => t.type === triggerId && t.sync === true);
}
```

---

## Summary: What Each Team Needs To Do

| Team | Register Trigger | Emit Event | Handle Result |
|------|-----------------|------------|---------------|
| **Dashboard** | `dashboard.onCreate` with `{ title, description }` schema | Before `savedObjects.client.create` in `create.ts` | Check `status === 'failed'`, use by-ref modified fields |
| **Cases** | `cases.onCreate` + `cases.createComment` | Before `caseService.createCase` / `model.createComment` | Check `status === 'failed'`, use by-ref modified fields |
| **Agent Builder** | `agent-builder.chatRound` + `.beforeInference` + `.afterInference` | Replace the for-loop in `runBeforeAgentWorkflows` | `workflow.fail` → abort, `event.mutate` → new prompt |
| **Any new team** | Follow the 3-step pattern in section 2 | 5-10 lines of code at the integration point | Standard `EmitEventResult` handling |

The key insight: **every team writes the same ~10 lines of integration code**. The workflow system handles resolution, execution, sync/async branching, and result collection.
