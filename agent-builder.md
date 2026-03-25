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
// Runs before the LLM sees the message. Workflows subscribing to this hook use
// the ai.pii step, which delegates to the inference plugin's anonymization
// pipeline (already available on main behind xpack.anonymization.active).
// The step returns the anonymized message and a replacementsId the caller stores
// on the Conversation. On subsequent turns, the caller passes the existing
// replacementsId back so the same tokens are reused — keeping the LLM context
// consistent across turns.
workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.beforeInference',
  eventSchema: z.object({
    message: z.string().describe('The raw user message to anonymize'),
    agentId: z.string().optional().describe('The agent handling this round'),
    conversationId: z.string().optional().describe('Conversation ID for logging and context'),
    replacementsId: z.string().optional().describe(
      'ID of an existing token map in ES — present on turn 2+. ' +
      'The step extends this map rather than creating a new one, ' +
      'so tokens remain stable across the conversation.'
    ),
  }),
  sync: {
    outputSchema: z.object({
      message: z.string().describe('The anonymized message with PII replaced by tokens'),
      replacementsId: z.string().optional().describe(
        'ID of the token map persisted to ES. ' +
        'The caller stores this on the Conversation and passes it back on the next turn.'
      ),
    }),
    maxTimeout: '10s',
    failurePolicy: 'open',        // if anonymization fails, proceed with raw message
  },
});

// ── Hook 3: After Inference (PII De-anonymization) ──────────────────────────
// Runs after the LLM responds. Receives the replacementsId from the
// before-inference output, loads the token map from ES, and restores
// original PII values in the response.
workflowsExtensions.registerTriggerDefinition({
  id: 'agent-builder.afterInference',
  eventSchema: z.object({
    response: z.string().describe('The LLM response that may contain PII tokens'),
    agentId: z.string().optional(),
    replacementsId: z.string().optional().describe(
      'ID of the token map produced by the before-inference hook. ' +
      'Used to look up original values and restore them in the response.'
    ),
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
  Pre-inference hook that anonymizes the user message before it reaches the LLM.
  The ai.pii step delegates to the inference plugin's existing anonymization
  pipeline (PII detection via regex and NER, HMAC-SHA256 token generation,
  replacements storage via ReplacementsRepository) — the same pipeline that runs
  automatically inside chatComplete when xpack.anonymization.active is enabled.
  Exposing it as a workflow step makes the behaviour user-configurable: entity
  types and tool deanonymization policy live in the YAML rather than being
  hardcoded. On subsequent turns the existing map is extended so the same entity
  always maps to the same token, keeping the LLM context stable across turns.
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
    replacementsId:
      type: string
      description: >
        ID of the token map persisted to ES. The caller stores this on the
        Conversation and passes it back on the next turn.

steps:
  # [PROPOSED STEP] ai.pii — scans the message text for PII, replaces with HMAC
  # tokens, and always persists the token map to ES via the inference plugin's
  # ReplacementsRepository (.kibana-anonymization-replacements).
  # Persistence is unconditional — it is required for multi-turn consistency,
  # page refresh recovery, distributed execution, and tool deanonymization.
  # - replacements_id (optional): if present, loads the existing map from ES and
  #   extends it with any new PII found this turn, so the same entity always maps
  #   to the same token across the conversation.
  # - tool_deanonymization: controls whether and how the internal beforeToolCall
  #   hook is allowed to swap tokens back to real values before a tool executes.
  #   This is the workflow-configurable surface for tool-level deanonymization —
  #   the workflow author decides which tools can see real values.
  - name: anonymise
    type: ai.pii                             # [PROPOSED STEP]
    with:
      input: "{{ event.message }}"
      replacements_id: "{{ event.replacementsId }}"
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
      # tool_deanonymization: governs the internal beforeToolCall hook.
      # The workflow author decides which tools are trusted to receive real values.
      # 'allowlist' — only listed tool_ids receive real values; all others see tokens.
      # 'all'       — every tool call is deanonymized.
      # 'none'      — tool deanonymization is disabled entirely.
      #
      # Example: a risk score lookup needs the real entity name to query correctly.
      # A summarization tool does not — it can work with tokens.
      tool_deanonymization:
        mode: allowlist
        tool_ids:
          - 'security.entity_analytics.risk_score'

  # Return the anonymized message and the ES token map ID.
  # The caller stores replacementsId on the Conversation and threads it into
  # agentRunner.run() so beforeToolCall / afterToolCall hooks can access it.
  - name: return_result
    type: workflow.output                    # [EXISTS]
    with:
      message: "{{ steps.anonymise.output.anonymised_text }}"
      replacementsId: "{{ steps.anonymise.output.replacements_id }}"

consts:
  pii_hmac_key: "REDACTED"
```

---

## Workflow YAML: PII De-anonymization (After Inference)

```yaml
version: '1'
name: PII De-anonymization — After Inference
description: >
  Post-inference hook that restores original PII values in the LLM's response.
  Receives the replacementsId from the before-inference output, loads the token
  map from ES, and replaces all tokens in the response with their originals.
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
  - name: check_has_replacements
    type: if                                 # [EXISTS]
    condition: "event.replacementsId : *"
    steps:
      # [PROPOSED STEP] transform.pii_restore — loads token map from ES by replacementsId,
      # scans for tokens in the response, and replaces them with originals.
      - name: deanonymise
        type: transform.pii_restore          # [PROPOSED STEP]
        with:
          input: "{{ event.response }}"
          replacements_id: "{{ event.replacementsId }}"

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

## Caller Code: Multi-turn Token Map Threading

The PII anonymization flow uses `replacementsId` to thread the token map across the request lifecycle. The `replacementsId` is:

1. Passed **into** `beforeInference` if an existing map exists (turn 2+)
2. Returned **from** `beforeInference` and stored on the `Conversation`
3. Passed **into** `afterInference` for response de-anonymization
4. Available to **internal tool hooks** (`beforeToolCall`/`afterToolCall`) for mid-turn token swap

```typescript
// x-pack/platform/plugins/shared/agent_builder/server/services/agents/run_inference.ts

export async function runInferenceWithPiiProtection({
  prompt,
  agentId,
  conversationId,
  existingReplacementsId,   // loaded from Conversation before this call
  workflowsClient,
  conversationStore,
  llm,
  logger,
}: RunInferenceParams) {
  // ── Phase 1: Anonymize ─────────────────────────────────────────────────
  // Pass the existing replacementsId if this is a follow-up turn.
  // The step extends the existing map so tokens remain stable across turns.
  const anonResult = await workflowsClient.invokeHook('agent-builder.beforeInference', {
    message: prompt,
    agentId,
    conversationId,
    replacementsId: existingReplacementsId,
  });

  // failurePolicy: 'open' → if anonymization fails, proceed with original
  const anonymizedPrompt = anonResult.output?.message ?? prompt;
  const replacementsId = anonResult.output?.replacementsId ?? existingReplacementsId;

  if (anonResult.status === 'failed') {
    logger.warn(`PII anonymization failed (proceeding with original): ${anonResult.error?.message}`);
  }

  // Persist replacementsId on Conversation so the next turn can load it back.
  if (replacementsId && replacementsId !== existingReplacementsId && conversationId) {
    await conversationStore.updateReplacementsId(conversationId, replacementsId);
  }

  // ── Phase 2: Run inference ─────────────────────────────────────────────
  // replacementsId is threaded into the agent runner so internal tool hooks
  // (beforeToolCall / afterToolCall) can deanonymize params and re-anonymize
  // results mid-turn without going through the workflow engine.
  const llmResponse = await agentRunner.run({
    message: anonymizedPrompt,
    replacementsId,
  });

  // ── Phase 3: De-anonymize ──────────────────────────────────────────────
  // The after-inference hook loads the token map from ES using replacementsId
  // and restores original values in the response.
  const deanonResult = await workflowsClient.invokeHook('agent-builder.afterInference', {
    response: llmResponse,
    agentId,
    replacementsId,
  });

  const restoredResponse = deanonResult.output?.response ?? llmResponse;

  if (deanonResult.status === 'failed') {
    logger.warn(`PII de-anonymization failed (returning raw response): ${deanonResult.error?.message}`);
  }

  return restoredResponse;
}
```

### Data Flow (multi-turn)

```
  Turn 1                                    Turn 2
  ──────                                    ──────
  conversation.replacementsId = undefined   conversation.replacementsId = "repl-abc-123"
       │                                         │
       ▼                                         ▼
  invokeHook('beforeInference',            invokeHook('beforeInference',
    { message, replacementsId: undefined })  { message, replacementsId: "repl-abc-123" })
       │                                         │
       │  ai.pii step:                           │  ai.pii step:
       │  → creates new token map                │  → loads existing map "repl-abc-123"
       │  → writes to ES                         │  → extends it with any new PII
       │  → returns replacementsId: "repl-abc-123" → returns same replacementsId
       │                                         │
       ▼                                         ▼
  store "repl-abc-123" on Conversation      same "repl-abc-123" — no update needed
       │                                         │
       ▼                                         ▼
  llm.inference(anonymizedPrompt)           llm.inference(anonymizedPrompt)
  — tool hooks use "repl-abc-123"           — same token map, tokens are consistent
       │                                         │
       ▼                                         ▼
  invokeHook('afterInference',             invokeHook('afterInference',
    { response, replacementsId: "repl-abc-123" }) { response, replacementsId: "repl-abc-123" })
```

---

## Tool Lifecycle Hooks

`beforeToolCall` and `afterToolCall` are **internal Agent Builder extension points** — not workflow triggers. See [README section 9](./README.md#9-internal-tool-hooks-beforetoolcall--aftertoolcall) for the rationale (latency, allowlist policy, always-re-anonymize guarantee).

Any Agent Builder feature can register into these hooks via `registerToolLifecycleHooks`. The `tool_deanonymization` block declared in the `beforeInference` workflow YAML (see above) is passed through as `toolDeanonymizationPolicy` on every `beforeToolCall` invocation.

```typescript
// [PROPOSED] Internal Agent Builder registration API

agentBuilder.registerToolLifecycleHooks({
  beforeToolCall: async ({ toolId, toolParams, replacementsId, toolDeanonymizationPolicy }) => {
    if (!replacementsId) return;

    // Only deanonymize if this tool is in the allowlist.
    // Tools not listed keep receiving tokens — they never see real values.
    if (!isToolAllowed(toolId, toolDeanonymizationPolicy)) return;

    const tokenToOriginalMap = await inference.getTokenToOriginalMap(replacementsId);
    return {
      toolParams: deanonymizeToolParams(toolParams, tokenToOriginalMap),
    };
  },

  afterToolCall: async ({ toolReturn, replacementsId }) => {
    if (!replacementsId) return;

    // Always re-anonymize results regardless of allowlist — the LLM context
    // must never contain real values, even if the tool needed them to execute.
    const tokenToOriginalMap = await inference.getTokenToOriginalMap(replacementsId);
    return {
      toolReturn: reanonymizeToolReturn(toolReturn, tokenToOriginalMap),
    };
  },
});
```

`replacementsId` is returned by `beforeInference` and threaded through `agentRunner.run()` as a request-scoped value so every tool hook invocation receives it automatically:

```
beforeInference output
  └── replacementsId: "repl-abc-123"
        │
        ├── stored on Conversation (for next turn)
        │
        └── passed to agentRunner.run({ replacementsId })
              │
              ├── beforeToolCall({ toolId, toolParams, replacementsId })
              │   → fetch token map → deanonymize params
              │
              ├── tool executes against real data
              │
              └── afterToolCall({ toolReturn, replacementsId })
                  → re-anonymize results
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
| `replacementsId` managed ad-hoc | `replacementsId` is part of the `beforeInference` hook contract — threaded explicitly |

The new model eliminates the per-team loop, the ad-hoc contract, and the manual workflow ID management. Workflows subscribe via triggers — the engine handles resolution and execution.

---

## Multi-turn Token Map Persistence

Persistence is unconditional — see [README section 8](./README.md#8-multi-turn-state-replacementsid) for the full rationale (multi-turn consistency, page refresh, session restore, distributed execution, tool hook access).

**The `ai.pii` step handles persistence via the inference plugin's `ReplacementsRepository`:**
- Input: optional `replacements_id` (from previous turns)
- If present: loads the existing map from `.kibana-anonymization-replacements` and merges new tokens into it
- If absent: creates a new map in the same index
- Always: writes the updated map and returns the `replacements_id`

This is **internal to the step** — the workflow engine is agnostic to it. From the engine's perspective, `replacementsId` is just another string field in the hook's output.
