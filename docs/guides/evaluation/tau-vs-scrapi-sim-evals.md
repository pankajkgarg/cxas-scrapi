# tau-bench vs. CXAS Scrapi Simulation Evals — Format Comparison

## Structural Philosophy

| | tau-bench / tau2 | CXAS Scrapi Simulation Evals |
|---|---|---|
| **Task definition** | Single `user_instruction` string | Sequential `steps[]` with per-step goals |
| **World state** | `init_state` — full declarative DB snapshot | `session_parameters` — variable injection only |
| **Success check** | Code-verifiable: DB state diff after run | LLM-judged: natural language `expectations[]` |
| **Step tracking** | N/A — single task | Gemini tracks `status` per step each turn |
| **User simulator** | Separate LLM given `user_instruction` | Gemini given steps + live `step_progresses` |
| **Policy** | Policy document handed to agent at runtime | Embedded in agent instructions |

---

## Side-by-Side Format

### tau2
```json
{
  "task_id": "retail_return_001",
  "difficulty": "easy",
  "task_type": "single_action",
  "domains": ["retail"],
  "user_instruction": "Return the black Adidas shoes from order W1234567. You are mildly frustrated.",
  "init_state": {
    "orders": { "W1234567": { "status": "delivered", "items": [{"product_id": "shoe_001"}] } },
    "customer": { "id": "C999", "name": "Alex Smith" }
  },
  "ground_truth_actions": [
    { "tool": "get_order_details", "params": { "order_id": "W1234567" } },
    { "tool": "process_return",    "params": { "order_id": "W1234567" } }
  ],
  "verifiable": true,
  "n_turns_estimate": 5
}
```

### CXAS Scrapi
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

**1. Ground truth vs. expectations**
tau2 checks `ground_truth_actions` deterministically against actual tool calls made — pass/fail is code-computed. Scrapi `expectations` are natural-language strings evaluated by Gemini post-conversation — pass/fail is LLM-judged and non-deterministic.

**2. World state vs. variable injection**
tau2's `init_state` declares the full starting DB state; the user simulator and verifier both know what records exist. Scrapi's `session_parameters` inject variables into the session — tool responses are only as controlled as your platform's mock/variable system.

**3. Single task vs. multi-step**
tau2 gives the simulator one `user_instruction` for the whole conversation. Scrapi splits the conversation into sequential `steps[]`, each with its own `goal`, `success_criteria`, and `max_turns`. The Gemini user-sim tracks which step is active on every turn and returns updated `step_progresses` alongside the next utterance — one LLM call does both.

**4. Persona**
tau2 embeds persona in the `user_instruction` prose. Scrapi has a dedicated `response_guide` field per step (tone, credentials to reveal, what to say), plus DTMF (`dtmf: <keys>`) and silence (`event: user_inactive`) escape hatches for voice scenarios — neither of which tau2 supports.

**5. Task metadata**
tau2 carries `difficulty`, `task_type`, `domains[]`, `verifiable`, and `n_turns_estimate` as structured fields enabling coverage analysis. Scrapi has only free-form `tags[]`.

**6. What the user-sim LLM sees**
tau2 gives the simulator the full conversation history including tool traces. Scrapi deliberately shows the user-sim only clean `User: / Agent:` text — it is kept ignorant of tool calls and transfers, which mirrors what a real caller would hear. Tool traces are reserved for the separate expectations-evaluation LLM at end-of-conversation.

---

## What Each Does Better

| CXAS Scrapi does better | tau2 does better |
|---|---|
| Multi-step sequential goal tracking | Deterministic, code-checkable pass/fail |
| Per-step turn budgets (`max_turns`) | Full world state declaration (`init_state`) |
| Voice-native: DTMF, silence, audio modality | Structured task metadata (difficulty, type, domain) |
| Static utterance escape hatch | Policy document integration |
| Direct integration with live production agents | Reproducible ground truth across runs |
