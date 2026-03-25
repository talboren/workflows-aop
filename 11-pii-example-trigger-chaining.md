# PII Anonymization via Trigger Chaining (Revised Example)

> **Replaces**: `02-pii-anonymization-before.yaml` and `03-pii-anonymization-after.yaml`
> **Pattern**: Trigger output chaining — the before-inference output flows through the caller to the after-inference input. No shared ephemeral state needed.

## The Problem with the Ephemeral State Approach

The original proposal uses `state.ephemeral.set/get` to share the PII token map between the before-inference and after-inference workflows:

```
Before-inference workflow                After-inference workflow
========================                =========================
1. Detect PII in message                1. state.ephemeral.get("pii_map:roundId")
2. Replace with tokens                  2. Scan response for tokens
3. state.ephemeral.set("pii_map:roundId", tokenMap)   3. Replace tokens with originals
4. event.mutate → rewrite message       4. event.mutate → rewrite response
```

**Security problem**: `state.ephemeral.get` is a step any workflow can call. An agent with workflow execution access can execute `state.ephemeral.get("pii_map:<guessable-round-id>")` and retrieve the full PII mapping — the exact data the anonymization was supposed to protect. The ephemeral store has no access control scoped to the originating workflow.

**Architecture problems**:
- `event.mutate` is not achievable by-ref (the engine serializes to ES — see design review Problem 3)
- The two workflows are coupled via a shared key convention (`"pii_map:{{ event.roundId }}"`) — fragile and invisible
- `state.ephemeral` is a new proposed step type that adds storage infrastructure just for this pattern

## The Solution: Trigger Output Chaining

Instead of shared state, the **before-inference workflow returns the token map as output**. The **caller** (Agent Builder) holds it in local memory and passes it as input to the after-inference workflow. The token map never leaves the caller's process memory (except during the ES round-trip within each workflow execution).

```
Caller (Agent Builder)
======================
1. invokeEvent('agent-builder.beforeInference', { message }) → { anonymisedMessage, tokenMap }
2. llm.inference(anonymisedMessage) → llmResponse
3. invokeEvent('agent-builder.afterInference', { response: llmResponse, tokenMap }) → { restoredResponse }
4. Return restoredResponse to user
```

The token map flows **through the caller**, not through a shared store. Each workflow is stateless. No `state.ephemeral` needed.

---

## Step 1: Trigger Registrations (Agent Builder team)

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/plugin.ts — setup()

import { z } from '@kbn/zod/v4';

// Before-inference: anonymize the user's message
// Input: the raw message
// Output: the anonymized message + the token map (for de-anonymization later)
workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.beforeInference',
  eventSchema: z.object({
    message: z.string().describe('The raw user message to anonymize'),
    agentId: z.string().optional().describe('The agent handling this round'),
  }),
  sync: {
    outputSchema: z.object({
      message: z.string().describe('The anonymized message (tokens substituted for PII)'),
      tokenMap: z.record(
        z.string(),
        z.object({
          original: z.string(),
          entity: z.string(),
        })
      ).describe('Map of token → { original value, entity type }. Empty object if no PII found.'),
    }),
    maxTimeout: '10s',
    failurePolicy: 'open',
  },
});

// After-inference: de-anonymize the LLM's response
// Input: the LLM response + the token map from the before-inference output
// Output: the restored response with original PII values
workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.afterInference',
  eventSchema: z.object({
    response: z.string().describe('The LLM response that may contain PII tokens'),
    tokenMap: z.record(
      z.string(),
      z.object({
        original: z.string(),
        entity: z.string(),
      })
    ).describe('The token map produced by the before-inference workflow'),
    agentId: z.string().optional(),
  }),
  sync: {
    outputSchema: z.object({
      response: z.string().describe('The response with original PII values restored'),
    }),
    maxTimeout: '10s',
    failurePolicy: 'open',
  },
});
```

---

## Step 2: Workflow YAML — Before Inference (Anonymize)

```yaml
# 02-pii-anonymization-before.yaml (revised)
#
# Changes from original:
#   - No sync: true on trigger (inherited from trigger definition)
#   - No event.mutate (by-ref is not possible — see design review)
#   - No state.ephemeral.set (token map returned via workflow.output)
#   - outputs declaration matches trigger's outputSchema

version: '1'
name: PII Anonymization — Before Inference
description: >
  Pre-inference guardrail that detects PII in the user message,
  replaces each value with a deterministic HMAC-SHA256 token,
  and returns the anonymized message along with the token map.
  The caller passes the token map to the after-inference workflow.
enabled: true
tags:
  - anonymization
  - pii
  - agent-builder

triggers:
  - type: agent-builder.beforeInference

outputs:
  type: object
  properties:
    message:
      type: string
      description: The anonymized message with PII replaced by tokens
    tokenMap:
      type: object
      description: Map of token to original value and entity type

steps:
  # Step 1: Detect PII and replace with deterministic tokens
  - name: anonymise
    type: ai.pii
    with:
      input: "{{ event.message }}"
      entities:
        - EMAIL_ADDRESS
        - US_SSN
        - CREDIT_CARD
        - PHONE_NUMBER
        - IP_ADDRESS
        - PERSON_NAME
      action: replace
      replace_strategy: hmac_sha256
      hmac_secret: "{{ consts.pii_hmac_key }}"

  # Step 2: Return both the anonymized text and the token map
  # The caller receives these as the invokeEvent result
  - name: return_result
    type: workflow.output
    with:
      message: "{{ steps.anonymise.output.anonymised_text }}"
      tokenMap: "{{ steps.anonymise.output.token_map }}"

consts:
  pii_hmac_key: "REDACTED"
```

---

## Step 3: Workflow YAML — After Inference (De-anonymize)

```yaml
# 03-pii-anonymization-after.yaml (revised)
#
# Changes from original:
#   - No sync: true on trigger
#   - No event.mutate
#   - No state.ephemeral.get/delete — token map arrives as event input
#   - No shared key convention — the caller passes the map directly

version: '1'
name: PII De-anonymization — After Inference
description: >
  Post-inference step that restores original PII values in the LLM's
  response using the token map provided by the caller. No ephemeral
  state access needed — the token map is passed as input.
enabled: true
tags:
  - anonymization
  - pii
  - agent-builder

triggers:
  - type: agent-builder.afterInference

outputs:
  type: object
  properties:
    response:
      type: string
      description: The response with original PII values restored

steps:
  # Step 1: Check if there's anything to de-anonymize
  - name: check_has_tokens
    type: if
    condition: "event.tokenMap : *"
    steps:
      # Step 2: Scan the LLM response for tokens and swap originals back in
      - name: deanonymise
        type: transform.pii_restore
        with:
          input: "{{ event.response }}"
          token_map: "{{ event.tokenMap }}"

      # Step 3: Return the restored response
      - name: return_restored
        type: workflow.output
        with:
          response: "{{ steps.deanonymise.output.restored_text }}"
    else:
      # No token map — return response unchanged
      - name: return_unchanged
        type: workflow.output
        with:
          response: "{{ event.response }}"
```

---

## Step 4: Caller Code (Agent Builder)

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/services/agents/run_inference.ts

export async function runInferenceWithPiiProtection({
  prompt,
  agentId,
  workflowsClient,
  llm,
}: {
  prompt: string;
  agentId?: string;
  workflowsClient: WorkflowsClient;
  llm: LlmClient;
}) {
  // ── Phase 1: Anonymize ────────────────────────────────────────────────
  // invokeEvent returns the anonymized message + token map.
  // If the workflow fails (failurePolicy: 'open'), output falls back
  // to undefined and we proceed with the original prompt.
  const anonResult = await workflowsClient.invokeEvent('agent-builder.beforeInference', {
    message: prompt,
    agentId,
  });

  const anonymizedPrompt = anonResult.output?.message ?? prompt;
  const tokenMap = anonResult.output?.tokenMap ?? {};

  if (anonResult.error) {
    logger.warn(`PII anonymization failed (proceeding with original): ${anonResult.error.message}`);
  }

  // ── Phase 2: Run inference with the anonymized prompt ─────────────────
  const llmResponse = await llm.inference(anonymizedPrompt);

  // ── Phase 3: De-anonymize ─────────────────────────────────────────────
  // Pass the token map from Phase 1 as input to the de-anonymization workflow.
  // No ephemeral state — the token map flows through the caller.
  const deanonResult = await workflowsClient.invokeEvent('agent-builder.afterInference', {
    response: llmResponse,
    tokenMap,
    agentId,
  });

  const restoredResponse = deanonResult.output?.response ?? llmResponse;

  if (deanonResult.error) {
    logger.warn(`PII de-anonymization failed (proceeding with raw response): ${deanonResult.error.message}`);
  }

  return restoredResponse;
}
```

---

## Data Flow Diagram

```
  Agent Builder (caller)                    Workflow Engine
  ======================                    ==============

  1. User sends: "Call me at 555-12-3456"
     │
     ├── invokeEvent('beforeInference',  ──→  Anonymize workflow:
     │     { message: "Call me at              1. ai.pii detects US_SSN
     │       555-12-3456" })                   2. Replaces with token: "Call me at <tok_abc>"
     │                                         3. workflow.output:
     │                                            { message: "Call me at <tok_abc>",
     │   ◄── output ─────────────────────────     tokenMap: { "<tok_abc>": { original: "555-12-3456",
     │       { message, tokenMap }                                           entity: "US_SSN" } } }
     │
     │  tokenMap is held in caller's local memory (never in shared state)
     │
     ├── llm.inference("Call me at <tok_abc>")
     │   ◄── "Sure, I'll call <tok_abc>"
     │
     ├── invokeEvent('afterInference',   ──→  De-anonymize workflow:
     │     { response: "Sure, I'll call        1. Checks tokenMap is not empty
     │       <tok_abc>",                       2. transform.pii_restore scans for tokens
     │       tokenMap: { "<tok_abc>":          3. Replaces <tok_abc> → 555-12-3456
     │         { original: "555-12-3456",      4. workflow.output:
     │           entity: "US_SSN" } } })          { response: "Sure, I'll call 555-12-3456" }
     │
     │   ◄── output ─────────────────────────
     │       { response: "Sure, I'll call 555-12-3456" }
     │
     └── Return to user: "Sure, I'll call 555-12-3456"
```

---

## Why This Is Better

| Concern | Ephemeral state approach | Trigger chaining approach |
|---------|--------------------------|--------------------------|
| **Security** | Any workflow can call `state.ephemeral.get("pii_map:roundId")` and read the PII mapping | Token map only exists in the caller's local memory and in each workflow's execution context (scoped, not globally accessible) |
| **Coupling** | Two workflows coupled via shared key convention (`"pii_map:{{ event.roundId }}"`) | Each workflow is fully self-contained — inputs in, outputs out |
| **New infrastructure** | Requires `state.ephemeral.set/get/delete` — a new storage subsystem | Uses only `workflow.output` — which already exists today |
| **By-ref requirement** | Relies on `event.mutate` which is architecturally impossible (ES serialization breaks references) | Pure by-value — caller receives output and decides what to use |
| **Testability** | Hard to test in isolation — each workflow depends on ephemeral state being set | Each workflow is independently testable with mock inputs |
| **Observability** | Token map is hidden in ephemeral state — not visible in execution logs | Token map is part of the workflow output — visible in execution history |
| **Cleanup** | Must explicitly delete ephemeral state (what if the after-workflow fails?) | No cleanup needed — nothing to clean up |
| **Parallel safety** | Key collision risk if round IDs are reused or predictable | No shared state — no collision possible |

## How This Maps to the Design Review Principles

1. **Trigger-level sync capability** (Problem 1): Both triggers are registered by the AB team with a `sync` block. The workflow author just subscribes — no `sync: true` in YAML.

2. **`invokeEvent` with typed result** (Problem 2): Each `invokeEvent` call returns a typed `InvokeEventResult`. The caller destructures `output` directly — no `| void` ambiguity.

3. **Always by-value** (Problem 3): The anonymize workflow returns `{ message, tokenMap }`. The de-anonymize workflow returns `{ response }`. No mutation, no by-ref. The caller explicitly picks what to use.

4. **Parallel execution possible** (Problem 4): The anonymize step must be sequential (LLM needs the cleaned prompt). But if you had a guardrail trigger alongside, it could run in `Promise.all` with the inference call.
