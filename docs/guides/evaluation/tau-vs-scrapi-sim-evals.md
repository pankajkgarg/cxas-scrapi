# τ²-bench vs. CXAS Scrapi Simulation Evals — Format Comparison

## Structural Philosophy

| | τ²-bench | CXAS Scrapi Simulation Evals |
|---|---|---|
| **User instruction** | Split across 4 sub-fields with conditional scripting | Sequential `steps[]` with per-step `goal` + `response_guide` |
| **Environment** | Reference DB + mock tools (sandbox) | Live CXAS agent over Sessions API |
| **World state** | `initial_state` field (often null; DB seeded separately) | `session_parameters` — variable injection |
| **Success check** | Multi-component: DB diff, substring match, LLM assertions | LLM-judged: natural-language `expectations[]` |
| **Step tracking** | N/A — single task | Gemini updates per-step `status` every turn |
| **User simulator** | LLM given structured `user_scenario.instructions` | Gemini given `steps[]` + live `step_progresses` |
| **Policy** | Explicit domain policy doc given to agent at runtime | Baked into agent instructions |

---

## Side-by-Side

### τ²-bench task (actual format from `airline/tasks.json`)
```json
{
  "id": "2",
  "description": {
    "purpose": "Testing capacity to handle change of topic + double-check user claims. Client should get $50 for a one-passenger delayed basic economy flight with insurance.",
    "relevant_policies": null,
    "notes": null
  },
  "user_scenario": {
    "persona": null,
    "instructions": {
      "domain": "airline",
      "reason_for_call": "First, try to book a flight from sf to ny with 3 passengers. Halfway through, abruptly mention you'd like to talk about something else — your frustration with a delayed flight in your most recent reservation.",
      "task_instructions": "If the agent asks for the reservation number of the delayed flight, say it's the last reservation you made but you don't remember it.\n\nIf the agent asks how many passengers were on that reservation, say 3. This is incorrect — admit you're wrong if corrected.\n\nDon't ask for compensation right away. First complain. Try to get the agent to offer it. If they don't after a few exchanges, ask explicitly.\n\nIf the agent asks whether to continue booking the sf-ny flight, say you'll call back later.",
      "known_info": "You are Noah Muller.\nYour user id is noah_muller_9847.",
      "unknown_info": null
    }
  },
  "initial_state": null,
  "evaluation_criteria": {
    "actions": [
      { "action_id": "2_0", "name": "get_user_details",       "arguments": { "user_id": "noah_muller_9847" }, "info": null },
      { "action_id": "2_1", "name": "get_reservation_details","arguments": { "reservation_id": "SDZQKO" },   "info": null },
      { "action_id": "2_2", "name": "get_reservation_details","arguments": { "reservation_id": "4OG6T3" },   "info": null },
      { "action_id": "2_3", "name": "send_certificate",       "arguments": { "user_id": "noah_muller_9847", "amount": 50 }, "info": null }
    ],
    "communicate_info": [],
    "nl_assertions": [
      "Agent should not offer compensation unless the user asks for it.",
      "Agent should check that the flight was indeed delayed.",
      "Agent should detect that the number of passengers mentioned by the user is incorrect.",
      "Agent should offer a certificate of $50."
    ]
  },
  "annotations": null
}
```

### CXAS Scrapi simulation (equivalent scenario)
```yaml
evals:
  - name: delayed_flight_compensation
    tags: [P1, compensation, topic-switch]
    session_parameters:
      user_id: "noah_muller_9847"
    steps:
      - goal: Start booking a flight SF→NY then abruptly switch topic to a delayed flight
        success_criteria: Agent acknowledges the topic switch and addresses the delayed flight
        response_guide: >
          Start asking to book a flight from SF to NY for 3 passengers.
          Mid-flow, say you're frustrated about a delayed flight in your last reservation.
          If asked for the reservation number, say you don't remember.
          If asked how many passengers, say 3 — admit you're wrong if corrected.
        max_turns: 6
      - goal: Get compensation for the delayed flight without asking for it directly
        success_criteria: Agent proactively offers a $50 certificate
        response_guide: >
          Complain about the delay without explicitly asking for compensation.
          Only ask directly if the agent hasn't offered after a few exchanges.
        max_turns: 8
    expectations:
      - "Agent must not offer compensation unless the user asks for it"
      - "Agent must verify the flight was actually delayed before offering compensation"
      - "Agent must catch that the passenger count given by the user is wrong"
      - "Agent must offer a $50 certificate"
```

---

## Key Differences

**1. User instruction structure**
τ² splits user setup into four dedicated fields: `reason_for_call` (the opening framing), `task_instructions` (detailed conditional scripting — "if agent says X, do Y"), `known_info` (identity facts: name, user ID), and `unknown_info`. Scrapi combines all of this into `response_guide` prose per step, with no enforced structure between persona, opening, and contingency behavior.

**2. Reward is multi-component, not a single LLM call**
τ² computes reward as a product of independently-evaluated components. `actions` documents one valid reference path — it's replayed on a fresh DB to derive the target state, then the agent's actual DB state is hashed and compared (DB reward). `communicate_info` is a substring match on the transcript. `nl_assertions` uses an LLM. Each active component in `reward_basis` must pass for full reward. Scrapi uses a single Gemini call over the full transcript to evaluate all `expectations` together.

**3. Actions are a reference, not a requirement**
In τ², `evaluation_criteria.actions` documents *one valid solution path* used to derive the target DB state — the agent can take a completely different path and still pass, as long as the DB ends up in the same state. For refusal tasks (like task 0), `actions` is empty `[]` and only `nl_assertions` applies. Scrapi has no equivalent DB-state concept at all.

**4. Single task vs. multi-step state machine**
τ² gives the simulator one task definition per conversation. Scrapi splits the dialogue into sequential `steps[]`, each with its own `goal`, `success_criteria`, and `max_turns`. On every turn, one Gemini call does double duty — emit the next user utterance *and* return the updated `step_progresses` (Not Started / In Progress / Completed). Step state is LLM-maintained, not code-enforced.

**5. What the user-sim LLM sees**
τ² shows the simulator the running conversation including all observable outputs. Scrapi deliberately feeds the user-sim only clean `User:` / `Agent:` text — it is kept blind to tool calls and transfers, mirroring what a real caller hears. The full trace (tool calls, responses, transfers, payloads) is reserved for the separate expectations-evaluator that runs once at end-of-conversation.

**6. Conditional scripting in the user persona**
τ²'s `task_instructions` can contain multi-paragraph conditional behavior scripts ("if agent says X, say Y; if corrected, admit it"). Scrapi's `response_guide` is a freeform hint to the LLM simulator but carries no enforced conditional structure — the LLM may or may not follow nuanced "if-then" persona instructions reliably.

---

## What Each Does Better

| CXAS Scrapi does better | τ²-bench does better |
|---|---|
| Multi-step sequential goal tracking | Multi-component reward (DB diff + substring + LLM) |
| Per-step turn budgets (`max_turns`) | Structured user instruction with conditional scripting |
| Voice-native: DTMF, silence, audio modality | DB-state verification — outcome not just conversation |
| Static utterance escape hatch | Explicit refusal/impossible task support (empty `actions`, `nl_assertions` only) |
| Tests the live deployed agent end-to-end | Reproducible across runs (sandboxed reference environment) |

τ² optimizes for **reproducible, multi-signal benchmarking** in a controlled sandbox.
Scrapi simulations optimize for **exercising a real production CXAS agent** through
multi-step, voice-capable, LLM-judged conversations.
