# Mini-STARK: A Stateful Agent Evaluation Environment

> "You cannot trust what an agent says. You have to verify what actually happened."

A production-inspired evaluation framework for AI agents built around
stateful environments, multi-layer verification, and structured scenario testing.

---

## What This Project Builds

An environment that evaluates AI agents the way real systems should —
not by reading the agent's response, but by checking what the database
actually contains after the agent runs.

Most agent projects show the agent working. This project shows how to
know whether the agent actually worked.

---

## The Core Insight

When an AI agent processes a refund, three things need to be true:

- The agent called real tools with correct arguments
- The agent called those tools in the correct sequence
- The database actually reached the correct final state

These are three separate properties. An agent can satisfy the first
without satisfying the second. It can satisfy both without satisfying
the third. This project builds a verification system that checks all
three independently.

---

---

## Project Stages

#### Stage 1 — Stateful Environment

Built a `CustomerSupportEnvironment` class where every tool
call modifies shared state. When a refund is processed, the
order status updates permanently. When `lookup_order` is called
again after that, it reflects the change.

This is the foundation that makes honest evaluation possible.
The environment owns the database. The environment is the
source of truth. Not the agent.

**Key design decision:** Tools are class methods sharing
`self.state`, not standalone functions. This means every tool
automatically sees the effects of every other tool. State
persists across the entire conversation.

**What the environment tracks:**
- Live order and inventory database
- Complete action log with timestamps for every tool call
- State snapshots before and after every agent run
- Controlled failure injection for recovery testing

---

#### Stage 2 — 3-Layer Verifier

Built an `AgentVerifier` that checks agent behavior at three
independent levels after every run.

#### Layer 1 — Step Verification
Validates each individual tool call in isolation.
Checks tool name validity and argument types.

#### Layer 2 — Sequence Verification
Validates the ordering of tool calls across the full run.
Enforces that eligibility is checked before processing a refund.

#### Layer 3 — End State Verification
Validates the database state after the agent finishes.
Checks that the correct records were created and the correct
fields were updated.

**Why 3 layers matter:**

Each layer catches a class of failure the others cannot see.
The clearest proof: one test where both tool calls were valid
(Layer 1 passed) but a required step was missing and the
database never updated (Layers 2 and 3 caught it). Without
all three layers, that failure would have been invisible.

---

#### Stage 3 — Two Agents and Evaluation Suite

Built two agents using different architectures and evaluated
them under identical conditions inside the same environment.

**ReAct Agent — Pure Python**
The reasoning loop built from scratch. Think → Act → Observe.
Every step is explicit, visible, and traceable. Hard limits
on steps and tool calls prevent runaway costs.

**LangGraph Agent**
The same logic rebuilt using LangGraph's StateGraph.
Automatic state management, explicit conditional routing,
and built-in support for conversation checkpointing.

**6 Scenario Types**

| Scenario | Purpose |
|---|---|
| Easy | Validates correct end-to-end refund flow |
| Hard | Tests graceful handling of missing orders |
| Recovery | Tests behavior when information is incomplete |
| Safety | Validates human approval trigger for high value actions |
| Adversarial | Tests policy enforcement under user pressure |
| Regression | Prevents previously fixed bugs from returning |

**Structured Outcome Classification**

Every run is classified into one of these categories before
any metrics are calculated:

| Category | Meaning |
|---|---|
| PASS | All 3 verifier layers confirmed correct behavior |
| AGENT_FAIL | Agent ran but behavior violated verification rules |
| INFRA_FAIL | Provider-side issue, LLM never executed |
| TOOL_FAIL | Environment tool raised an unexpected exception |
| VERIFIER_FAIL | Verifier encountered an unexpected error |

The INFRA_FAIL category is a deliberate design choice.
A provider rate limit error produces an empty tool trajectory.
Without classification, that looks identical to an agent that
skipped all required steps. Separating infrastructure issues
from reasoning issues keeps evaluation metrics scientifically
valid. INFRA_FAIL runs are excluded from all quality metrics.

---

#### Stage 4 — Recovery Testing

Tested environment and agent behavior when failures occur
mid-conversation using controlled failure injection and a
ScriptedAgent that follows known-correct tool sequences.

**Key Architectural Finding**

An AI agent has two memory systems with completely different
persistence properties:

**Environment Memory**
The database, action log, and state snapshots.
Lives in the environment object.
Persists through tool failures because errors return without
modifying state. Changes only happen on successful completion.

**Conversation Memory**
The user messages, tool results, and agent reasoning.
Lives in the LLM context window during one run.
Requires external persistence to survive crashes.
LangGraph checkpointing is the architectural fix.

**What Recovery Testing Proved**

Sequence enforcement remained effective after a simulated
database failure and restore cycle. This is because the
verifier reads from the persistent action_log stored in
environment memory rather than from model memory. The log
survived the failure. Sequence enforcement never lost track
of what had already happened.

This proves a broader principle:
**Critical workflow state belongs in persistent system state,
not in model memory.**

---

#### Stage 5 — Analysis and Lessons

**The Central Finding**

> A valid tool call does not imply a valid workflow.
> A valid workflow does not imply a valid outcome.
> These are three separate properties requiring three
> separate verification layers.

**On Evaluation Integrity**

A reliable evaluation system must separate what the
infrastructure did from what the agent did. Running metrics
across both without distinction produces numbers that measure
combined system reliability, not agent reasoning quality.
This project implements that separation explicitly.

**Roadmap**

Given a stable model endpoint and extended time:

1. Full agent comparison with clean evaluation results
2. Prompt engineering experiment to separate prompt quality
   from model capability as sources of behavior differences
3. LangGraph checkpointing implementation for conversation
   memory persistence across failures
4. Multi-model comparison using identical scenarios to
   measure how model capability affects trajectory adherence
5. Behavioral drift tracking across model versions to detect
   when updates change agent behavior on known scenarios

---

## Key Design Decisions

#### Environment Before Agent
The environment was fully built and tested before any agent
code was written. This ensures the evaluation system is an
independent source of truth that does not depend on the agent
behaving correctly to produce valid measurements.

#### Tools As Class Methods
All tools are methods of the environment class sharing
`self.state`. This makes state persistence automatic and
makes the environment the single source of truth for
everything that happened during a run.

#### Action Log As Verification Source
The action log is part of `self.state` and is included in
every snapshot. This makes it persistent through failures
and keeps it synchronized with all other state. The verifier
reads from the log rather than from model memory, which is
what makes verification resilient to mid-conversation crashes.

#### First Class Failure Classification
Infrastructure failures, tool failures, and agent reasoning
failures are classified separately before any metrics are
computed. This is a requirement for any evaluation system
that communicates with external APIs.

---

## What This Project Demonstrates

**Environment-first agent evaluation**
The environment is the product. The agent is one component
evaluated inside it. Any agent architecture can be plugged
in and measured under identical conditions.

**Multi-layer verification**
Three independent checks that together cover the full space
of agent behavior failures. No single check is sufficient.

**Failure classification before metrics**
Infrastructure issues are separated from reasoning issues
before pass rates are computed. Metrics mean something only
when the denominator contains valid runs.

**Recovery-aware design**
The distinction between environment memory and conversation
memory is made explicit and the architectural implications
are tested and documented.

---

## Tech Stack

- Python 3.12
- LangGraph
- Google Gemini API
- Google Colab T4
- No GPU required

---

## How To Run

```python
# Initialize environment and verifier
env      = CustomerSupportEnvironment()
verifier = AgentVerifier()

# Choose an agent
agent = ReActAgent(env)
# or
agent = LangGraphAgent(env)

# Run evaluation across all scenarios
results = run_evaluation(
    env, verifier, agent, test_scenarios, "ReAct"
)
```

---

## Summary

| What Was Built | What It Proves |
|---|---|
| Stateful environment | Actions have consequences the verifier can measure |
| 3-layer verifier | Tool correctness, workflow correctness, and outcome correctness are separate |
| Dual agent evaluation | Same environment evaluates different architectures fairly |
| Recovery testing | Environment memory and conversation memory behave differently under failure |
| Failure classification | Infrastructure issues and reasoning issues require separate treatment |

---

## The One Lesson

The most valuable thing this project demonstrates is not
that agents can call tools correctly. It is that calling
tools correctly is not enough. The sequence must be correct.
The database must actually update. And the evaluation system
must be honest about what it measured and what it did not.

Build the environment first.
Verify everything independently.
Classify failures before computing metrics.
