# Dashboards Integration

> The Dashboard plugin uses a lifecycle hook at one point: **before creation** — to run PII reduction on the title and description before the dashboard is persisted. This document covers trigger registration, workflow YAML, and caller code.

## Trigger Registration

```typescript
// src/platform/plugins/shared/dashboard/server/plugin.ts — setup()

import { z } from '@kbn/zod/v4';

// Lifecycle hook: runs before a dashboard is saved.
// Input and output schemas are the same shape — the workflow can modify
// title/description and the engine returns the modified fields as output
// without requiring an explicit workflow.output step.
workflowsExtensions.registerTriggerDefinition({
  id: 'dashboard.beforeCreate',
  eventSchema: z.object({
    title: z.string().describe('The dashboard title'),
    description: z.string().optional().describe('The dashboard description'),
  }),
  sync: {
    outputSchema: z.object({
      title: z.string(),
      description: z.string().optional(),
    }),
    maxTimeout: '10s',
    failurePolicy: 'open',         // broken workflow should not block dashboard saves
  },
});
```

Note: the `eventSchema` and `outputSchema` have the **same shape** (`{ title, description }`). This enables the implicit output pattern — see below.

---

## Workflow YAML: PII Reduction on Dashboard Fields

This workflow demonstrates the **implicit output pattern**. Since the trigger's input and output schemas match, the workflow does not need a `workflow.output` step. The engine returns the (potentially modified) event fields as the hook output.

```yaml
version: '1'
name: Dashboard PII Reduction
description: >
  Runs before a dashboard is saved. Scans the title and description
  for PII patterns (SSN, email, credit card) and replaces them with
  masked values. Uses implicit output — no workflow.output step needed
  because the input and output schemas are the same shape.
enabled: true
tags:
  - pii
  - dashboard

triggers:
  - type: dashboard.beforeCreate             # lifecycle hook — sync inherited from trigger

# No explicit outputs declaration needed — the trigger's outputSchema
# matches the eventSchema, so the engine returns modified event fields.

steps:
  # Step 1: Redact PII from the title
  # [EXISTS] data.regexReplace — regex-based replacement, available today
  - name: redact_title
    type: data.regexReplace                  # [EXISTS]
    with:
      input: "{{ event.title }}"
      patterns:
        - pattern: '\b\d{3}-\d{2}-\d{4}\b'
          replacement: '***-**-****'
        - pattern: '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
          replacement: '[EMAIL REDACTED]'
        - pattern: '\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b'
          replacement: '[CARD REDACTED]'

  # Step 2: Redact PII from the description (if present)
  - name: redact_description
    type: data.regexReplace                  # [EXISTS]
    if: "event.description : *"
    with:
      input: "{{ event.description }}"
      patterns:
        - pattern: '\b\d{3}-\d{2}-\d{4}\b'
          replacement: '***-**-****'
        - pattern: '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
          replacement: '[EMAIL REDACTED]'
        - pattern: '\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b'
          replacement: '[CARD REDACTED]'

  # No workflow.output step — the engine returns the modified event fields
  # (redact_title.output replaces event.title, redact_description.output
  # replaces event.description) as the hook output automatically.
```

### Why no `workflow.output`?

When the trigger's `eventSchema` and `outputSchema` have the same shape, the engine can return the event object (with any modifications applied by the steps) as the output. The workflow author doesn't need to manually wire each field through a `workflow.output` step.

This matters for usability: if the dashboard event had 7+ fields, an explicit `workflow.output` would require listing all 7 fields — both in the YAML and in the caller code. The implicit pattern eliminates that boilerplate.

If the schemas were **different** (e.g., the output includes extra fields like `piiDetected: boolean`), the workflow would need an explicit `workflow.output` step.

### Redaction vs. anonymization

This workflow uses **redaction** — PII patterns are masked and the originals are discarded. This is appropriate here because the dashboard is being saved for the first time and the original values do not need to be recovered later.

If the use case required **anonymization** — replacing values with reversible tokens so originals can be restored (e.g. for audit trails or de-anonymizing downstream responses) — the `ai.pii` step [PROPOSED STEP] should be used instead, following the pattern in the [Agent Builder guide](./agent-builder.md). That pattern persists a token map to ES via `replacementsId` and enables the `transform.pii_restore` step to restore originals on demand.

---

## Caller Code

### File: `src/platform/plugins/shared/dashboard/server/api/create/create.ts`

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

+  // ── Lifecycle hook: pre-create PII reduction ──────────────────────────
+  const workflowsClient = workflows.getWorkflowsClient();
+  const hookResult = await workflowsClient.invokeHook('dashboard.beforeCreate', {
+    title: soAttributes.title ?? '',
+    description: soAttributes.description ?? '',
+  });
+
+  if (hookResult.status === 'failed') {
+    // failurePolicy: 'open' → engine returns status: 'failed' but we could
+    // choose to proceed. For 'closed' triggers, we'd throw here.
+    // Since dashboard.beforeCreate is 'open', this block is optional:
+    logger.warn(`Dashboard pre-create hook failed: ${hookResult.error?.message}`);
+  }
+
+  if (hookResult.status === 'completed') {
+    soAttributes.title = hookResult.output.title;
+    soAttributes.description = hookResult.output.description;
+  }
+  // ────────────────────────────────────────────────────────────────────────

   // ... access control ...

   const savedObject = await core.savedObjects.client.create<DashboardSavedObjectAttributes>(
     DASHBOARD_SAVED_OBJECT_TYPE,
     soAttributes,
     { references: soReferences, /* ... */ }
   );

   return getDashboardCRUResponseBody(savedObject, 'create', dashboardStateSchema, isDashboardAppRequest);
 }
```

### What the dashboard team needs to do

1. **Register the trigger** in `plugin.ts` setup (code above)
2. **Add ~15 lines** in `create.ts` to call `invokeHook` and apply the result
3. **No workflow management** — users subscribe workflows to `dashboard.beforeCreate` via the workflow authoring UI; the engine handles resolution and execution
