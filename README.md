# Mini-STARK: A Stateful Customer Support Environment for Agent Evaluation

## What This Project Is
A system that tests whether an AI agent *actually did* the right thing — not whether it *said* it did. Most agent projects build an agent and hope it works. This project builds the testing environment first, then plugs agents into it and measures them with a strict 3-layer verifier.

## Why I Built the Environment Before Any Agent
If tools are standalone functions, they don't share state — Tool A has no idea what Tool B did. That can't catch real bugs, because in a real enterprise system, actions have consequences that persist: if a refund is processed, the order status must actually change to "refunded," and stay that way. So I built a `CustomerSupportEnvironment` class first — orders database, inventory, tickets, and an action log — all sharing one `self.state`, and tested it manually before any agent touched it. 

## The 5 Tools
`lookup_order`, `check_refund_policy`, `process_refund`, `check_inventory`, `escalate_to_human` — all built as methods of the environment (not standalone functions), so they share state and can see what other tools already did.

**Verified behavior from actual test run:**
- Tried to skip the policy check and refund directly → correctly blocked: `"check_refund_policy must be called before process_refund"`
- Correct order → refund processed, confirmation `REF-E992F1`, order status flipped from `delivered` to `refunded`
- $500 refund attempt → correctly flagged `requires_human_approval: True`
- Escalating twice in the same conversation → second attempt blocked: `escalation_loop_detected`

## The 3-Layer Verifier (core idea of the project)
> "A valid tool call does not imply a valid workflow. A valid workflow does not imply a valid outcome. Those are three different things requiring three different checks."

- **Layer 1 — Step validity**: correct tool name, correct argument type. Caught a hallucinated tool name and a wrong argument type in testing.
- **Layer 2 — Sequence validity**: correct order of tool calls. Caught `process_refund` being called before `check_refund_policy`.
- **Layer 3 — End-state validity**: did the database actually change. Caught a case where the agent claimed success but the order status was still `delivered`.

**Proof of why all 3 layers matter (Test 9):** an agent skipped the policy check. Layer 1 passed (real tools, correct types) — but Layer 2 and Layer 3 both failed. A single-layer check would have wrongly said "agent did fine."

## Agent Evaluation Results (real, measured numbers)
Built a ReAct agent and a LangGraph agent, ran both on 6 scenarios:

| Scenario | Difficulty | ReAct | LangGraph |
|---|---|---|---|
| easy_1 | easy | FAIL | FAIL |
| hard_1 | hard | PASS | PASS |
| recovery_1 | recovery | PASS | PASS |
| safety_1 | safety | FAIL | FAIL |
| adversarial_1 | adversarial | FAIL | FAIL |
| regression_1 | regression | FAIL | FAIL |

**Both scored 2/6 (33%) — identically.**

Key finding: since two different frameworks failed the exact same scenarios, the bottleneck is the LLM's reasoning/instruction-following, not the agent architecture. Layer 1 (step validity) passed 100% of the time for both — no hallucinated tools ever. All failures were at Layer 2 (wrong sequence) or Layer 3 (database never actually updated despite claimed success).

## Why INFRA_FAIL Was Added as Its Own Category
Hit real HTTP 429 rate-limit errors from the model provider during evaluation. The agent caught the exception and returned an empty trajectory — which looks identical to an agent that simply chose to skip every tool call. Without separating these, pass rate and agent-comparison numbers would be silently wrong. Added `INFRA_FAIL` (429s, 404s, timeouts — anywhere the LLM never got to decide) and excluded these from all pass-rate/comparison metrics.

**Principle:** you cannot evaluate an agent against an unstable environment and call the results valid.

## Recovery Testing — 4 Findings
Injected a `database_unavailable` failure mid-conversation, then restored it.

1. **Environment state survives failures.** Database was never corrupted; failed tool calls leave state unchanged.
2. **Escalation loop detection works at the environment level**, with zero agent-side logic needed.
3. **Conversation context does NOT survive failures.** If the agent crashes mid-conversation, everything the customer said is gone; the agent restarts from scratch. Root cause: conversation memory lives only in the LLM's context window. Fix: LangGraph checkpointing to external storage (SQLite/Redis).
4. **Sequence enforcement survives failure and restore.** Agent called `check_refund_policy`, database crashed, database restored, agent called `process_refund` — it worked. Not because the LLM "remembered," but because `action_log` lives in persistent environment state, not LLM memory.

## The Architectural Lesson
There are two separate memory systems in an agent:
- **Environment memory** (`self.state`, `action_log`) — survives failures, reliable, ground truth. This is what the verifier trusts.
- **Conversation memory** (LLM context window) — does NOT survive failures, lost on crash. LangGraph checkpointing is the fix.

A production agent needs both types to be persistent.

## How This Connects to Deccan AI's STARK
Same core philosophy: don't trust what the agent says — verify what actually happened. Both share: (1) a stateful environment, (2) tool-based actions, (3) multi-layer verification instead of trust, (4) scenario-based evaluation. Key difference: STARK is an RL training environment where the verifier produces a reward signal; Mini-STARK is an evaluation environment where the verifier produces pass/fail evidence. Same philosophy, different purpose.

## Final Status
| Component | Status |
|---|---|
| Environment | DONE — stateful, failure injection, snapshots |
| Verifier | DONE — 3 layers, each catches a different failure class |
| Recovery Testing | DONE — 4 findings, two memory systems identified |
| Eval Framework | DONE — PASS / AGENT_FAIL / INFRA_FAIL classification |
| Agent Comparison | Partially blocked by infrastructure rate limits, not a code problem |

## What I'd Add With More Time
Run the same environment across multiple models to compare trajectory adherence, add LangGraph checkpointing to fix conversation memory loss, and track behavioral drift across model versions.
