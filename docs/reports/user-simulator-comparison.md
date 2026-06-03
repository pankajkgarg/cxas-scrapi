# User Simulator Comparison: CXAS SCRAPI vs. τ-bench

**Date:** 2026-06-03
**Scope:** Compares the LLM-driven user simulator in this repo
(`cxas_scrapi.evals.simulation_evals`) with the user simulator in Sierra
Research's [τ-bench](https://github.com/sierra-research/tau-bench)
(`tau_bench/envs/user.py`), and identifies gaps where the SCRAPI simulator
could be strengthened.

---

## 1. Executive summary

Both projects use an LLM to role-play a human user that talks to an agent over
multiple turns so the agent can be evaluated on whole-conversation behavior
rather than single-turn responses. They diverge sharply in *philosophy*:

- **SCRAPI** is a **structured, goal-tracking, voice-first** simulator. The
  same LLM call both *speaks* as the user and *self-reports* progress through a
  scripted list of `steps` (a built-in state machine). It is purpose-built for
  Conversational Agents / CCAI (telephony) testing — it natively models DTMF,
  silence/no-input, welcome events, and audio modality, and can inject mock
  tool variables.

- **τ-bench** is a **minimal, persona-driven, text-only** simulator whose only
  job is to *talk*. It is deliberately "dumb" about success: the user never
  grades itself. Task success is computed separately and objectively from the
  **environment's ground-truth state** (a hash of the backend database plus
  required output strings). τ-bench ships **multiple user strategies**
  (`llm`, `react`, `verify`, `reflection`, plus `human`) and is
  **model/provider-agnostic** via litellm.

The headline finding: SCRAPI's simulator is richer on *I/O surface* (voice,
mocking, multi-step reporting) but weaker on *evaluation rigor and
simulator fidelity*. Its biggest gaps versus τ-bench are (1) the actor and the
judge are the **same LLM call** (self-grading), (2) there is **no
ground-truth / state-based success signal**, (3) there is **only one
simulation strategy** (no verify/reflection self-correction), and (4)
**weaker anti-hallucination and naturalness guardrails** in the user prompt.

---

## 2. How each simulator works

### 2.1 SCRAPI (`simulation_evals.py` + `prompts/llm_user_prompts.py`)

- **Driver model:** Gemini (default `gemini-3.1-flash-lite`), Vertex-only.
- **Input contract:** a `test_case` with an ordered list of `steps`. Each step
  has `goal`, `success_criteria`, `response_guide` (persona/context hints),
  `max_turns`, optional `static_utterance` (send exact text), and
  `inject_variables`. Top level also has `session_parameters` (mock tool
  variables) and post-hoc `expectations`.
- **Turn loop:** `LLMUserConversation._next_user_utterance()` builds one prompt
  (`LLM_USER_PROMPT`) that contains the script, the current `step_progress`,
  and the transcript. A **single structured JSON response** returns both
  `next_user_utterance` **and** the fully updated `step_progresses` list. So
  the simulator *acts and self-grades in the same call.*
- **State machine:** each step is `Not Started → In Progress → Completed`, with
  an LLM-written `justification`. The prompt prescribes very specific behavior:
  describe only the problem (never hint at `success_criteria`), a **two-turn
  acknowledgment** ("Okay, I'll try that" → "That worked!"), **loop detection**
  (escalate to a human if the agent repeats itself a 3rd time in the last 4
  turns), and DTMF / silence handling for voice.
- **Termination:** all steps `Completed`, OR global `current_turn >= 30`
  (`_MAX_TURNS`), OR the agent ends the session via `end_session`.
- **Scoring:** two separate signals — step completion (self-judged by the user
  simulator) and `expectations`, judged *after the fact* by a **separate**
  Gemini judge call (`EVALUATE_EXPECTATIONS_PROMPT`) over the full trace. A run
  "passes" only if all goals completed AND all expectations met.
- **Extras:** parallel runs, retry/backoff, golden-dataset export, and a
  platform conversation-history fetch for accurate trace reconstruction.

### 2.2 τ-bench (`tau_bench/envs/user.py`)

- **Driver model:** any litellm-supported model (default `gpt-4o`), selectable
  with `--user-model`; strategy selectable with `--user-strategy`.
- **Input contract:** one free-text `instruction` (persona + goal + every datum
  the user "knows", e.g. their name, order ID, preferences).
- **Base class:** `BaseUserSimulationEnv` with `reset()`, `step()`,
  `get_total_cost()`. A `load_user()` factory dispatches the strategy.
- **System prompt (LLM strategy), verbatim rules:**
  - "Just generate one line at a time to simulate the user's message."
  - "Do not give away all the instruction at once. Only provide the
    information that is necessary for the current step."
  - "Do not hallucinate information that is not provided in the instruction…
    if the agent asks for the order id but it is not mentioned… do not make up
    an order id, just say you do not remember or have it."
  - "If the instruction goal is satisfied, generate `###STOP###` as a
    standalone message… to end the conversation."
  - "Do not repeat the exact instruction… use your own words."
  - "Try to make the conversation as natural as possible, and stick to the
    personalities in the instruction."
- **Strategies (the differentiator):**
  - `llm` — the base prompt above.
  - `react` — emit a private `Thought:` then a `User Response:`; only the
    response is sent to the agent (chain-of-thought for better decisions).
  - `verify` — after generating a response, a **supervisor LLM** classifies it
    `true`/`false` (satisfactory?); if false, regenerate, up to a max.
  - `reflection` — if unsatisfactory, generate a written reflection on what
    went wrong and propose an improved response (self-refinement loop).
  - `human` — collect real keyboard input (cost 0).
- **Termination:** the user emits `###STOP###`, or the env hits its max steps.
- **Scoring:** **not** done by the user simulator. The env computes reward
  objectively by comparing the resulting backend database state against the
  task's expected state (DB hash) plus checking required output strings. This
  is deterministic and hard to game.
- **Extras:** per-simulation **token cost tracking**; provider-agnostic.

---

## 3. Side-by-side

| Dimension | SCRAPI | τ-bench |
|---|---|---|
| Primary domain | Voice / CCAI agents (telephony) | Text / chat tool-agents (retail, airline) |
| Driver model | Gemini only (Vertex) | Any model via litellm (default gpt-4o) |
| User input format | Ordered `steps` (goal + success_criteria + response_guide) | Single free-text `instruction`/persona |
| Strategies | One (LLM + structured self-grading) | Five: `llm`, `react`, `verify`, `reflection`, `human` |
| Self-correction of the user turn | None | `verify` + `reflection` strategies |
| Who judges success? | The user simulator self-grades steps; a **separate** LLM judges `expectations` post-hoc | Environment computes reward from **ground-truth DB state** |
| Ground-truth / state assertions | None in the sim path (only LLM judgment) | Yes — DB hash + required outputs |
| End-of-conversation signal | All steps done / max turns / agent `end_session` | User emits `###STOP###` / env max steps |
| Voice features (DTMF, silence, welcome, audio) | Yes, first-class | No (text only) |
| Mock tool responses | Yes (`session_parameters`, `inject_variables`) | Via the env's tools/DB, not the user |
| Exact-input testing | Yes (`static_utterance`) | No |
| Anti-hallucination guardrail | Implicit/weak (user is handed all credentials) | Explicit ("don't make up an order id…") |
| Naturalness / paraphrase rule | Not stated (prompt is prescriptive/robotic) | Explicit ("use your own words", "as natural as possible") |
| Incremental info reveal | Partial (don't hint at `success_criteria`) | Explicit ("don't give away all the instruction at once") |
| Loop / stuck handling | Built-in loop detection → escalate to human | Not built into the user (env handles steps) |
| Per-step turn budget | Soft — `max_turns` is only shown to the LLM, not enforced in code | Env-enforced max steps |
| Cost tracking | Turns + wall-clock only | Token/$ cost per run |
| Regression artifacts | Golden-dataset export | Trajectory logs |

---

## 4. Where the SCRAPI simulator is lacking

Ordered roughly by impact.

### 4.1 Actor and judge are the same call (self-grading bias) — **high**
SCRAPI's user simulator produces the next utterance **and** decides whether each
step's `success_criteria` is met **in one JSON response**, and that
self-assessment is what *drives termination*. A model grading its own
conversation tends to be optimistic and can declare success prematurely.
τ-bench deliberately keeps the user "blind" to scoring and derives success from
objective environment state. SCRAPI does have a separate `expectations` judge,
but it runs only *after* the conversation and does not gate the step machine.

> **Recommendation:** Separate "speaking" from "grading." Let the user
> simulator only emit an utterance (+ an optional intent like `done`/`continue`),
> and move step-completion judgments into the independent judge that already
> exists for `expectations`. At minimum, run the step judge as a second call
> rather than folding it into the actor's output.

### 4.2 No ground-truth / state-based success signal — **high**
Every success signal in the simulation path is an LLM opinion
(`success_criteria` + `expectations`). τ-bench's strongest property is that it
checks **what actually happened** in the backend (DB hash, required outputs),
which is deterministic and cannot be talked into passing. SCRAPI *can* assert on
tool calls/state in the **golden** path, but the **simulation** path has no
programmatic assertion on tool arguments, tool outputs, or resulting state.

> **Recommendation:** Add optional programmatic checks to a simulation step —
> e.g. "tool `X` was called with arg `account_id == 9820598207`" or
> "final session parameter `order_status == shipped`" — evaluated from the
> already-captured `detailed_trace` / platform history, independent of any LLM.

### 4.3 Only one simulation strategy (no verify/reflection) — **medium/high**
τ-bench's `verify` and `reflection` strategies measurably improve user fidelity:
a supervisor catches responses that hallucinate, leak the whole instruction, or
break persona, and the user regenerates. SCRAPI has no such self-correction —
a bad user turn (e.g., the simulator inventing a detail or revealing the success
criteria) goes straight to the agent and corrupts the eval.

> **Recommendation:** Add a pluggable `user_strategy` and implement at least a
> `verify` pass (a cheap true/false supervisor) around `_next_user_utterance`.

### 4.4 Weak anti-hallucination guardrail — **medium**
τ-bench explicitly forbids inventing data the user wasn't given ("don't make up
an order id, just say you do not remember"). SCRAPI's `response_guide` typically
*hands the user all the credentials*, and the prompt has no rule against
fabricating values the agent asks for but the script didn't supply. This lets
the simulated user invent IDs/details, which makes the agent look like it
authenticated/served a user it shouldn't have.

> **Recommendation:** Add an explicit rule to `LLM_USER_PROMPT`: only use facts
> present in the active step's `response_guide` / injected variables; otherwise
> say you don't have/remember them.

### 4.5 Over-scripted, less natural user — **medium**
The SCRAPI prompt prescribes near-robotic behavior: a fixed two-turn
acknowledgment ("Okay, I'll try that" → "That worked!"), templated escalation
sentences, and the explicit instruction to *not* paraphrase static utterances.
τ-bench instead pushes for naturalness and persona fidelity ("use your own
words", "as natural as possible", "stick to the personalities"). Over-scripting
narrows the distribution of user behavior and can make agents pass evals they'd
fail against real, varied users.

> **Recommendation:** Add naturalness/paraphrase and persona-consistency
> guidance, and consider making the two-turn acknowledgment optional rather
> than mandatory.

### 4.6 Per-step `max_turns` is only soft-enforced — **medium**
The only hard cap in code is the global `_MAX_TURNS = 30`
(`_check_conversation_status`). A step's `max_turns` from YAML is merely
*shown to the LLM* in the prompt and relied upon to self-honor — unreliable.
τ-bench enforces step limits in the environment loop.

> **Recommendation:** Track turns per active step in code and enforce the
> step's `max_turns` deterministically (drive the escalation/transition rather
> than asking the model to remember to).

### 4.7 "Completed" conflates success and failure — **low/medium**
On loop detection or max-turns, the prompt marks the step `Completed` (with a
"Step failed…" justification). Reporting then counts the step as completed for
pass/fail unless the justification text is parsed. Status should distinguish
*succeeded* from *terminated-as-failed*.

> **Recommendation:** Add a `FAILED`/`ABANDONED` status distinct from
> `COMPLETED`.

### 4.8 Single-model / single-vendor judge (self-preference risk) — **low**
The user simulator and the expectations judge are both Gemini. Using the same
model family to drive and grade conversations invites self-preference bias.
τ-bench's litellm layer makes it trivial to use a different model for the user
vs. the agent vs. the judge.

> **Recommendation:** Allow the judge (and optionally the user) model to be
> configured independently from the agent.

### 4.9 No cost tracking — **low**
SCRAPI reports turns and wall-clock but not token/$ cost. τ-bench accumulates
per-run cost, which matters when sweeping many simulations.

---

## 5. Where SCRAPI is *ahead* of τ-bench (for balance)

- **Voice/telephony realism:** native DTMF (`dtmf:`), silence/no-input
  (`event: user_inactive`), welcome events, and an `audio` modality that
  exercises TTS/STT pipelines. τ-bench is text-only.
- **Deterministic mocking:** `session_parameters` / `inject_variables` mock
  tool/variable state for fast, isolated runs.
- **Granular multi-step reporting:** per-step goal/criteria/justification makes
  it clear *which* part of a flow broke and why — more diagnostic than a single
  task-level reward.
- **Exact-input edge cases:** `static_utterance` for testing specific phrasings.
- **Built-in stuck-loop → human-escalation** behavior.
- **Golden-dataset export** for turning a passing simulation into a
  deterministic regression test.
- **Operational maturity:** parallelism, retries/backoff, rich reports.

---

## 6. Prioritized recommendations

1. **Decouple acting from grading** (4.1) and **add programmatic state/tool
   assertions to sims** (4.2) — these two give SCRAPI the objective-rigor that
   is τ-bench's core advantage.
2. **Add a `verify` (and later `reflection`) user strategy** (4.3) to catch bad
   user turns before they reach the agent.
3. **Harden the user prompt:** explicit anti-hallucination rule (4.4),
   naturalness/persona fidelity (4.5).
4. **Enforce per-step `max_turns` in code** (4.6) and **add a distinct
   `FAILED` status** (4.7).
5. **Make judge/user models configurable** (4.8) and **track token cost**
   (4.9).

---

## 7. Sources

- This repo: `src/cxas_scrapi/evals/simulation_evals.py`,
  `src/cxas_scrapi/prompts/llm_user_prompts.py`,
  `docs/guides/evaluation/local-simulations.md`,
  `examples/evals/simulation_eval_example.yaml`.
- τ-bench: [`sierra-research/tau-bench`](https://github.com/sierra-research/tau-bench)
  (`tau_bench/envs/user.py`), and the paper
  [τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains](https://arxiv.org/pdf/2406.12045).
</content>
</invoke>
