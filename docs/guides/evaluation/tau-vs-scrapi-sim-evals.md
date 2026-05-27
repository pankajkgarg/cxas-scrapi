# τ-bench / τ²-bench vs. CXAS Scrapi Simulation Evals

A format comparison for anyone deciding how to author conversational agent
evaluations in CXAS, or porting scenarios to/from the τ benchmarks. Scope is the
**scrapi simulation YAML** (`evals:` → `steps:`), not goldens or turn evals.

## Structural Philosophy

| | τ-bench / τ²-bench | CXAS Scrapi Simulation Evals |
|---|---|---|
| **Task definition** | Single `instruction` string | Sequential `steps[]`, each with its own goal |
| **Environment** | Reference DB + mock tools (sandbox) | Live CXAS agent over Sessions API |
| **World state** | `init_state` — full declarative DB snapshot | `session_parameters` — variable injection |
| **Success check** | Code-verifiable: DB-state diff + output-string match | LLM-judged: natural-language `expectations[]` |
| **Step tracking** | N/A — single task | Gemini updates per-step `status` every turn |
| **User simulator** | LLM given `instruction` | Gemini given `steps[]` + live `step_progresses` |
| **Policy** | Explicit domain policy doc given to agent | Baked into agent instructions |

---

## Side-by-Side

### τ-bench task (`Task` dataclass; τ² keeps the same shape)
```json
{
  "user_id": "alex_smith_999",
  "instruction": "You are Alex Smith. Return the black Adidas shoes from order #W1234567. You are mildly frustrated. Do not reveal the order ID unless asked.",
  "actions": [
    { "name": "get_order_details", "kwargs": { "order_id": "#W1234567" } },
    { "name": "return_delivered_order_items",
      "kwargs": { "order_id": "#W1234567", "item_ids": ["shoe_001"] } }
  ],
  "outputs": ["return has been initiated"]
}
```
Reward is computed, not judged: the `actions` are replayed on a fresh DB and the
resulting state is hashed, then compared to the DB state after the agent's run;
separately, every string in `outputs` must appear in the transcript. Both must
pass for reward = 1.

### CXAS Scrapi simulation
```yaml
evals:
  - name: process_return
    tags: [P0, returns]
    session_parameters:
      order_W1234567_status: "delivered"
    steps:
      - goal: Report wanting to return shoes from order W1234567
        success_criteria: Agent locates the order and initiates a return
        response_guide: You are a mildly frustrated customer. The order is W1234567, black Adidas shoes.
        max_turns: 8
    expectations:
      - "Agent must look up the order before processing the return"
      - "Agent must confirm the return was initiated"
```

---

## Key Differences

**1. Verifiable reward vs. LLM-judged expectations**
τ replays ground-truth `actions` and diffs database state — pass/fail is
deterministic and reproducible across runs. Scrapi `expectations` are
natural-language strings scored by Gemini after the conversation, so the same
transcript can score differently run to run.

**2. Declarative world state vs. variable injection**
τ's `init_state` defines the entire starting DB; both the user simulator and the
reward function know exactly what records exist. Scrapi simulations inject
`session_parameters` as variables — there is no post-run state assertion, so
correctness depends on what the live backend returns.

**3. Single instruction vs. multi-step state machine**
τ hands the simulator one `instruction` for the whole dialogue. Scrapi splits the
conversation into ordered `steps[]`, each with its own `goal`,
`success_criteria`, and `max_turns`. On every turn one Gemini call does double
duty — emit the next user utterance *and* return the updated `step_progresses`
(Not Started / In Progress / Completed). The step state machine is therefore
LLM-maintained, not enforced in code.

**4. Persona**
τ embeds persona in the `instruction` prose. Scrapi has a dedicated
`response_guide` per step (tone, credentials to reveal, what to say), plus
voice-native escape hatches — `dtmf: <keys>` for keypad entry and
`event: user_inactive` for silence/no-input — that τ has no concept of.

**5. Task metadata**
τ² distinguishes domains, dual-control vs. single-control tasks, and ships an
explicit policy doc per domain. Scrapi simulations carry only free-form `tags[]`.

**6. What the user-sim LLM sees**
τ shows the simulator the running conversation. Scrapi deliberately feeds the
user-sim only clean `User:` / `Agent:` text — it is kept *blind to tool calls and
transfers*, mirroring what a real caller hears. The full trace (tool calls,
responses, transfers, payloads) is reserved for the separate
expectations-evaluator that runs once at end-of-conversation.

> **Note on structured assertions.** τ's tool-level checking has no equivalent in
> the scrapi *simulation* YAML — but it does exist elsewhere in CXAS: turn evals
> (`type: tool_called` / `tool_input` / `tool_output`) and the underlying platform
> scenario format (`scenarioExpectations[].toolExpectation.expectedToolCall`).
> The gap is the simulation surface, not the platform.

---

## What Each Does Better

| CXAS Scrapi does better | τ-bench / τ²-bench does better |
|---|---|
| Multi-step sequential goal tracking | Deterministic, code-checkable reward |
| Per-step turn budgets (`max_turns`) | Full declarative world state (`init_state`) |
| Voice-native: DTMF, silence, audio modality | Structured tool-call ground truth (`actions`) |
| Static-utterance escape hatch | Domain policy + dual-control task modeling |
| Tests the **live deployed agent** end-to-end | Reproducible across runs (sandbox env) |

τ optimizes for **reproducible, code-verified benchmarking** against a reference
sandbox. Scrapi simulations optimize for **exercising a real production CXAS agent**
through multi-step, voice-capable, LLM-judged conversations.
