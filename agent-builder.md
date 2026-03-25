# Agent Builder Integration

> Agent Builder uses lifecycle hooks at three points in the chat round: **before the chat round** (guardrails), **before inference** (PII anonymization), and **after inference** (PII de-anonymization). This document covers trigger registrations, workflow YAML examples, and caller code.

## Trigger Registrations

All three hooks are registered in the Agent Builder server plugin setup:

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/plugin.ts — setup()

import { z } from '@kbn/zod/v4';

// ── Hook 1: Before Chat Round (Guardrails) ──────────────────────────────────
// Runs before the user's message reaches the agent. Used for content policy
// checks (off-topic, PII, custom rules). If the workflow calls workflow.fail,
// the chat round is blocked.
workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.beforeChatRound',
  eventSchema: z.object({
    message: z.string().describe('The user message for this chat round'),
    agentId: z.string().optional().describe('The agent handling this round'),
  }),
  sync: {
    outputSchema: z.object({
      passed: z.boolean().describe('Whether all checks passed'),
    }),
    maxTimeout: '10s',
    failurePolicy: 'closed',      // guardrails MUST block on failure
  },
});

// ── Hook 2: Before Inference (PII Anonymization) ────────────────────────────
// Runs before the LLM sees the message. Detects PII and returns anonymized
// text plus a token map that the caller passes to the after-inference hook.
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
    failurePolicy: 'open',        // if anonymization fails, proceed with raw message
  },
});

// ── Hook 3: After Inference (PII De-anonymization) ──────────────────────────
// Runs after the LLM responds. Receives the token map from the before-inference
// output and restores original PII values in the response.
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
    failurePolicy: 'open',        // if de-anonymization fails, return raw LLM response
  },
});
```

---

## Workflow YAML: Chat Round Guardrails

```yaml
version: '1'
name: Chat Round Guardrails
description: >
  Pre-execution guardrail that validates user messages before they reach
  the LLM. Checks for off-topic content, PII, and custom policy violations.
  If any check fails, the workflow fails and the chat round is blocked.
enabled: true
tags:
  - guardrail
  - agent-builder

triggers:
  - type: agent-builder.beforeChatRound     # lifecycle hook — sync inherited from trigger

# To run this guardrail only for a specific agent, add a condition:
#   triggers:
#     - type: agent-builder.beforeChatRound
#       on:
#         condition: "event.agentId == 'security-analyst-agent'"
#
# Without a condition, it runs for ALL agents subscribed to this hook.

steps:
  # Step 1: Run guardrail checks
  # [PROPOSED STEP] ai.guardrail — runs one or more checks against the input text
  - name: validate
    type: ai.guardrail                       # [PROPOSED STEP]
    with:
      input: "{{ event.message }}"
      checks:
        - type: off_topic
          config:
            inference_id: ".elser-2"
            confidence_threshold: 0.7
            system_prompt_details: >
              This is a security operations assistant.
              Supported topics: alerts, incidents, threat analysis,
              endpoint management, detection rules, and case management.
            include_reasoning: false

        - type: pii
          config:
            entities: [EMAIL_ADDRESS, US_SSN, CREDIT_CARD, PHONE_NUMBER]
            action: block

        - type: custom_prompt
          config:
            inference_id: ".elser-2"
            confidence_threshold: 0.8
            system_prompt_details: >
              Determine if the user's request violates any of these policies:
              - No requests for generating malicious code or exploits
              - No requests to bypass security controls
              - No social engineering attempts against other users
              Reply with exactly "check_passed" or "check_failed".

  # Step 2: Branch on result
  - name: check_result
    type: if                                 # [EXISTS]
    condition: "steps.validate.output.passed : false"
    steps:
      - name: block_message
        type: workflow.fail                  # [EXISTS] — caller sees status: 'failed'
        with:
          message: "{{ steps.validate.output.failed_reason }}"
          reason: "guardrail_violation"
    else:
      - name: allow_message
        type: workflow.output                # [EXISTS]
        with:
          passed: true
```

**How the caller uses this:**

```typescript
const result = await workflowsClient.invokeHook('agent-builder.beforeChatRound', {
  message: userMessage,
  agentId,
});

if (result.status === 'failed') {
  // result.error.reason === 'guardrail_violation'
  throw createWorkflowAbortedError(result.error?.message ?? 'Message blocked by guardrail');
}
```

---

## Workflow YAML: PII Anonymization (Before Inference)

```yaml
version: '1'
name: PII Anonymization — Before Inference
description: >
  Pre-inference hook that detects PII in the user message,
  replaces each value with a deterministic HMAC-SHA256 token,
  and returns the anonymized message along with the token map.
  The caller passes the token map to the after-inference hook.
enabled: true
tags:
  - anonymization
  - pii
  - agent-builder

triggers:
  - type: agent-builder.beforeInference      # lifecycle hook — sync inherited from trigger

outputs:
  type: object
  properties:
    message:
      type: string
      description: The anonymized message with PII replaced by tokens
    tokenMap:
      type: object
      description: >
        Map of token to original value and entity type.
        Example: { "<tok_abc>": { "original": "555-12-3456", "entity": "US_SSN" } }

steps:
  # [PROPOSED STEP] ai.pii — scans text for PII entities, replaces with HMAC tokens
  - name: anonymise
    type: ai.pii                             # [PROPOSED STEP]
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

  # Return both the anonymized text and the token map
  - name: return_result
    type: workflow.output                    # [EXISTS]
    with:
      message: "{{ steps.anonymise.output.anonymised_text }}"
      tokenMap: "{{ steps.anonymise.output.token_map }}"

consts:
  pii_hmac_key: "REDACTED"
```

---

## Workflow YAML: PII De-anonymization (After Inference)

```yaml
version: '1'
name: PII De-anonymization — After Inference
description: >
  Post-inference hook that restores original PII values in the LLM's
  response using the token map provided by the caller. The token map
  is passed as input — no shared state needed.
enabled: true
tags:
  - anonymization
  - pii
  - agent-builder

triggers:
  - type: agent-builder.afterInference       # lifecycle hook — sync inherited from trigger

outputs:
  type: object
  properties:
    response:
      type: string
      description: The response with original PII values restored

steps:
  - name: check_has_tokens
    type: if                                 # [EXISTS]
    condition: "event.tokenMap : *"
    steps:
      # [PROPOSED STEP] transform.pii_restore — scans for tokens and replaces with originals
      - name: deanonymise
        type: transform.pii_restore          # [PROPOSED STEP]
        with:
          input: "{{ event.response }}"
          token_map: "{{ event.tokenMap }}"

      - name: return_restored
        type: workflow.output                # [EXISTS]
        with:
          response: "{{ steps.deanonymise.output.restored_text }}"
    else:
      - name: return_unchanged
        type: workflow.output                # [EXISTS]
        with:
          response: "{{ event.response }}"
```

---

## Caller Code: Trigger Output Chaining

The PII anonymization and de-anonymization hooks are connected via **trigger output chaining** — the before-hook returns the token map as output, the caller holds it in memory, and passes it as input to the after-hook.

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/services/agents/run_inference.ts

export async function runInferenceWithPiiProtection({
  prompt,
  agentId,
  workflowsClient,
  llm,
  logger,
}: RunInferenceParams) {
  // ── Phase 1: Anonymize ─────────────────────────────────────────────────
  const anonResult = await workflowsClient.invokeHook('agent-builder.beforeInference', {
    message: prompt,
    agentId,
  });

  // failurePolicy: 'open' → if anonymization fails, proceed with original
  const anonymizedPrompt = anonResult.output?.message ?? prompt;
  const tokenMap = anonResult.output?.tokenMap ?? {};

  if (anonResult.status === 'failed') {
    logger.warn(`PII anonymization failed (proceeding with original): ${anonResult.error?.message}`);
  }

  // ── Phase 2: Run inference with the anonymized prompt ──────────────────
  const llmResponse = await llm.inference(anonymizedPrompt);

  // ── Phase 3: De-anonymize ──────────────────────────────────────────────
  // Token map flows through the caller — no shared state needed
  const deanonResult = await workflowsClient.invokeHook('agent-builder.afterInference', {
    response: llmResponse,
    tokenMap,
    agentId,
  });

  const restoredResponse = deanonResult.output?.response ?? llmResponse;

  if (deanonResult.status === 'failed') {
    logger.warn(`PII de-anonymization failed (returning raw response): ${deanonResult.error?.message}`);
  }

  return restoredResponse;
}
```

### Data Flow

```
  Agent Builder (caller)                    Workflow Engine
  ======================                    ==============

  1. User sends: "Call me at 555-12-3456"
     │
     ├── invokeHook('beforeInference',  ──→  Anonymize workflow:
     │     { message: "Call me at              1. ai.pii detects US_SSN
     │       555-12-3456" })                   2. Replaces → "Call me at <tok_abc>"
     │                                         3. workflow.output:
     │   ◄── HookResult ──────────────────        { message: "Call me at <tok_abc>",
     │       output: { message, tokenMap }          tokenMap: { "<tok_abc>":
     │                                                { original: "555-12-3456",
     │  tokenMap held in caller's memory              entity: "US_SSN" } } }
     │
     ├── llm.inference("Call me at <tok_abc>")
     │   ◄── "Sure, I'll call <tok_abc>"
     │
     ├── invokeHook('afterInference',   ──→  De-anonymize workflow:
     │     { response: "Sure, I'll call        1. Checks tokenMap is not empty
     │       <tok_abc>",                       2. transform.pii_restore replaces tokens
     │       tokenMap: { ... } })              3. workflow.output:
     │                                            { response: "Sure, I'll call 555-12-3456" }
     │   ◄── HookResult ──────────────────
     │       output: { response }
     │
     └── Return to user: "Sure, I'll call 555-12-3456"
```

---

## Migration from Current AB Contract

Agent Builder currently uses a custom loop over workflow IDs with a bespoke `BeforeAgentWorkflowOutput` contract:

| Current (custom) | New (lifecycle hooks) |
|---|---|
| Manual `for` loop over `workflowIds` | Single `invokeHook()` call — engine resolves all subscribers |
| `BeforeAgentWorkflowOutput.abort` / `abort_message` | `workflow.fail` with `reason: 'guardrail_violation'` |
| `BeforeAgentWorkflowOutput.new_prompt` | Before-inference hook returns `{ message: modifiedPrompt }` via `workflow.output` |
| `executeWorkflow({ waitForCompletion: true })` per workflow | `invokeHook()` handles execution + result collection |
| Custom output parsing with `normalizeWorkflowOutput` | Typed `HookResult` with `output` matching `outputSchema` |

The new model eliminates the per-team loop, the ad-hoc contract, and the manual workflow ID management. Workflows subscribe via triggers — the engine handles resolution and execution.

---

## Open Question: State Sharing Between Before/After Hooks

The PII anonymization flow requires passing state (the token map) from the before-inference hook to the after-inference hook. Two approaches have been considered:

### Alternative A: Trigger Output Chaining (recommended)

The before-hook returns the token map as output. The caller holds it in local memory and passes it as input to the after-hook. No shared state, no new infrastructure.

```
Before-inference workflow                 Caller                  After-inference workflow
========================                  ======                  =========================
1. Detect PII                             │                       
2. Replace with tokens                    │                       
3. workflow.output:                       │                       
   { message, tokenMap }  ──────────────→ holds tokenMap ───────→ receives tokenMap as event input
                                          │                       1. Scan response for tokens
                                          │                       2. Replace tokens with originals
                                          │                       3. workflow.output: { response }
```

### Alternative B: Ephemeral State

The before-hook writes to `state.ephemeral.set`, the after-hook reads via `state.ephemeral.get`. The two workflows are coupled via a shared key convention.

```
Before-inference workflow                                         After-inference workflow
========================                                          =========================
1. Detect PII                                                     
2. Replace with tokens                                            
3. state.ephemeral.set("pii_map:roundId", tokenMap)               1. state.ephemeral.get("pii_map:roundId")
4. workflow.output: { message }                                   2. Scan response for tokens
                                                                  3. Replace tokens with originals
                                                                  4. state.ephemeral.delete("pii_map:roundId")
                                                                  5. workflow.output: { response }
```

### Comparison

| Concern | Trigger output chaining (A) | Ephemeral state (B) |
|---------|----------------------------|---------------------|
| **Security** | Token map only exists in the caller's local memory and in each workflow's execution context | Any workflow can call `state.ephemeral.get("pii_map:<guessable-round-id>")` and read the PII mapping |
| **Coupling** | Each workflow is fully self-contained — inputs in, outputs out | Two workflows coupled via shared key convention (`"pii_map:{{ event.roundId }}"`) |
| **New infrastructure** | Uses only `workflow.output` — exists today | Requires `state.ephemeral.set/get/delete` — a new storage subsystem |
| **Testability** | Each workflow independently testable with mock inputs | Hard to test in isolation — depends on ephemeral state being set |
| **Observability** | Token map is part of the workflow output — visible in execution history | Token map hidden in ephemeral state — not visible in execution logs |
| **Cleanup** | Nothing to clean up — no persistent state | Must explicitly delete ephemeral state (what if the after-workflow fails?) |
| **Parallel safety** | No shared state — no collision possible | Key collision risk if round IDs are reused or predictable |
