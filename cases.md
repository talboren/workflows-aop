# Cases Integration

> The Cases plugin uses lifecycle hooks at two points: **before case creation** and **before comment creation** — to run PII guardrails that block operations containing sensitive data. This document covers trigger registrations, workflow YAML, and caller code.

## Trigger Registrations

```typescript
// x-pack/platform/plugins/shared/cases/server/plugin.ts — setup()

import { z } from '@kbn/zod/v4';

// ── Hook 1: Before Case Creation ─────────────────────────────────────────
// Runs before a case is persisted. Can validate/modify title and description,
// or block creation entirely via workflow.fail.
workflowsExtensions.registerTriggerDefinition({
  id: 'cases.beforeCreate',
  eventSchema: z.object({
    title: z.string().describe('The case title'),
    description: z.string().optional().describe('The case description'),
  }),
  sync: {
    outputSchema: z.object({
      title: z.string(),
      description: z.string().optional(),
    }),
    maxTimeout: '10s',
    failurePolicy: 'open',          // broken workflow should not block case creation
  },
});

// ── Hook 2: Before Comment Creation ──────────────────────────────────────
// Runs before a comment is posted on a case. Scans the comment text for PII
// and blocks the comment if PII is found.
workflowsExtensions.registerTriggerDefinition({
  id: 'cases.beforeComment',
  eventSchema: z.object({
    caseId: z.string().describe('The case ID'),
    comment: z.string().describe('The comment text'),
    owner: z.string().describe('The case owner (e.g. securitySolution)'),
  }),
  sync: {
    outputSchema: z.object({
      passed: z.boolean().describe('Whether the comment passed all checks'),
    }),
    maxTimeout: '10s',
    failurePolicy: 'closed',        // PII guardrails MUST block on failure
  },
});
```

---

## Workflow YAML: Comment PII Guardrail

```yaml
version: '1'
name: Case Comment PII Guardrail
description: >
  Blocks case comments that contain PII. Scans the comment text for
  sensitive data patterns (SSN, credit cards, emails, phone numbers)
  and rejects the comment if any are found.
enabled: true
tags:
  - guardrail
  - pii
  - cases

triggers:
  - type: cases.beforeComment                # lifecycle hook — sync inherited from trigger

steps:
  # Step 1: Scan for PII in the comment
  # [PROPOSED STEP] ai.guardrail — same step type used in Agent Builder guardrails,
  # showing the pattern is reusable across different systems with different triggers.
  - name: validate_comment
    type: ai.guardrail                       # [PROPOSED STEP]
    with:
      input: "{{ event.comment }}"
      checks:
        - type: pii
          config:
            entities:
              - EMAIL_ADDRESS
              - US_SSN
              - CREDIT_CARD
              - PHONE_NUMBER
              - IP_ADDRESS
            action: block

  # Step 2: Block or allow
  - name: check_result
    type: if                                 # [EXISTS]
    condition: "steps.validate_comment.output.passed : false"
    steps:
      - name: block_comment
        type: workflow.fail                  # [EXISTS] — caller sees status: 'failed'
        with:
          message: >
            Comment blocked: PII detected ({{ steps.validate_comment.output.failed_reason }}).
            Please remove sensitive information before posting.
          reason: "pii_detected"
    else:
      - name: allow_comment
        type: workflow.output                # [EXISTS]
        with:
          passed: true
```

**Pattern reusability**: The `ai.guardrail` step is the same step type used in the Agent Builder chat round guardrails. Only the trigger and the input field differ. A workflow author can copy this pattern to any other trigger with minimal changes.

---

## Caller Code

### Case Creation: `x-pack/platform/plugins/shared/cases/server/client/cases/create.ts`

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

+  // ── Lifecycle hook: pre-create ────────────────────────────────────────
+  if (workflowsClient) {
+    const hookResult = await workflowsClient.invokeHook('cases.beforeCreate', {
+      title: normalizedCase.title,
+      description: normalizedCase.description,
+    });
+
+    if (hookResult.status === 'failed') {
+      // failurePolicy: 'open' → log and proceed
+      logger.warn(`Case pre-create hook failed: ${hookResult.error?.message}`);
+    }
+
+    if (hookResult.status === 'completed') {
+      normalizedCase.title = hookResult.output.title;
+      normalizedCase.description = hookResult.output.description;
+    }
+  }
+  // ──────────────────────────────────────────────────────────────────────

   const newCase = await caseService.createCase({
     attributes: transformNewCase({ ...normalizedCase, /* ... */ }),
     id: savedObjectID,
   });
   // ... user actions, notifications ...
   return flattenCaseSavedObject({ savedObject: newCase, /* ... */ });
 };
```

### Comment Creation: `x-pack/platform/plugins/shared/cases/server/client/attachments/add.ts`

```diff
+  // ── Lifecycle hook: pre-comment guardrail ─────────────────────────────
+  if (workflowsClient) {
+    const hookResult = await workflowsClient.invokeHook('cases.beforeComment', {
+      caseId: addArgs.caseId,
+      comment: commentReq.comment,
+      owner: commentReq.owner,
+    });
+
+    if (hookResult.status === 'failed') {
+      // failurePolicy: 'closed' → block the comment
+      throw createCaseError({
+        message: hookResult.error?.message ?? 'Comment blocked by workflow',
+        logger,
+      });
+    }
+  }
+  // ──────────────────────────────────────────────────────────────────────
```

Note the difference in error handling:
- `cases.beforeCreate` has `failurePolicy: 'open'` → the caller logs a warning but proceeds
- `cases.beforeComment` has `failurePolicy: 'closed'` → the caller throws and blocks the operation

### What the Cases team needs to do

1. **Register both triggers** in `plugin.ts` setup (code above)
2. **Add ~15 lines** in `create.ts` for the case creation hook
3. **Add ~12 lines** in `add.ts` for the comment guardrail hook
4. **No workflow management** — users subscribe workflows to `cases.beforeCreate` or `cases.beforeComment` via the workflow authoring UI
