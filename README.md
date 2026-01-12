[![Version](https://img.shields.io/badge/version-0.1.0-blue)](#) [![Python](https://img.shields.io/badge/python-3.10%2B-blue?logo=python)](https://www.python.org/) [![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

# SynthCore

**SynthCore** is the orchestration engine at the heart of the Nexus Client architecture.  
It is the single, authoritative control plane responsible for assembling context, enforcing identity and behavioral invariants, managing token budgets, coordinating memory and mood subsystems, and producing coherent, budget-safe, degradation-resilient LLM responses.

SynthCore is not a chatbot loop.  
It is a **governor**, **scheduler** and **integrity enforcer** for long-horizon AI systems.

---

## Table of Contents

- [What SynthCore Is](#what-synthcore-is)
- [What SynthCore Is Not](#what-synthcore-is-not)
- [Core Responsibilities](#core-responsibilities)
- [Architectural Position](#architectural-position)
- [Subsystem Contracts](#subsystem-contracts)
- [End-to-End Turn Lifecycle](#end-to-end-turn-lifecycle)
- [Token Budget Enforcement](#token-budget-enforcement)
- [Prompt Assembly Model](#prompt-assembly-model)
- [Graceful Degradation](#graceful-degradation)
- [Failure Guarantees](#failure-guarantees)
- [Directory Structure](#directory-structure)
- [Key Design Principles](#key-design-principles)
- [Phase 1 Scope](#phase-1-scope)
- [Phase 2 Forward Compatibility](#phase-2-forward-compatibility)
- [Status](#status)

---

## What SynthCore Is

SynthCore is the **single entry point** for all user-facing interactions in the Nexus Client.

It coordinates four foundational subsystems:

1. **SynthIdentity** – persistent identity and invariant enforcement  
2. **SynthMemory** – episodic and semantic memory retrieval  
3. **SynthMood** – dimensional affective state with decay  
4. **Primary LLM** – the language model that produces the final response  

SynthCore decides:
- *What context is retrieved*
- *How much context is allowed*
- *What is injected into the prompt*
- *What happens when something fails*
- *Whether the response is allowed to leave the system*

Nothing bypasses SynthCore.

---

## What SynthCore Is Not

SynthCore explicitly does **not**:
- Learn identity (Phase 1)
- Perform emotional role-play
- Execute tools or plugins (Phase 2)
- Store long-term data directly
- Decide *what* the assistant believes

It enforces behavior; it does not invent it.

---

## Core Responsibilities

SynthCore is responsible for:

- Loading identity snapshots (with safe fallback)
- Loading and decaying mood state
- Building and enforcing token budgets
- Retrieving and packing memory within budget
- Assembling the final structured prompt
- Calling the primary LLM
- Validating the generated response
- Persisting episodic memory
- Handling all errors via graceful degradation

If any of these steps fail, SynthCore **still returns a response**, unless the LLM itself is unreachable.

---

## Architectural Position

```

User Input
↓
SynthCore (Orchestrator)
├─ SynthIdentity
├─ SynthMood
├─ SynthMemory
├─ TokenBudget
└─ Prompt Assembler
↓
Primary LLM
↓
SynthCore (Post-Checks + Persistence)
↓
User Output

```

SynthCore is both the **first** and **last** component in the turn lifecycle.

---

## Subsystem Contracts

SynthCore interacts with subsystems strictly via contracts.

### Identity Contract
- Scope: `user_id`
- Must return a valid `IdentitySnapshot`
- May fail → fallback to minimal skeleton
- Invariants are checked **after** response generation

### Mood Contract
- Scope: `user_id`
- Must apply time-based decay
- Mood is metadata, never role-play
- May fail → fallback to baseline mood

### Memory Contract
- Scope: `user_id`, session-aware ranking
- Retrieval is budget-aware
- May fail → memory omitted entirely

### LLM Contract
- Must accept structured prompt
- Must respect max token output
- If unreachable → hard failure

---

## End-to-End Turn Lifecycle

For every request, SynthCore executes the following pipeline:

1. Initialize token budget
2. Load identity snapshot (or fallback)
3. Load and decay mood state (or baseline)
4. Retrieve relevant memory (budget-aware)
5. Assemble strict 5-section prompt
6. Call primary LLM
7. Validate response against identity invariants
8. Persist episodic memory
9. Return final response

Each step is time-bound and independently degradable.

---

## Token Budget Enforcement

SynthCore enforces **proactive** token budgeting.

Budgets are allocated *before* prompt assembly, not after.

### Guaranteed Rules
- Budget is never exceeded
- Output tokens are always reserved
- Memory is greedily packed last
- Conversation history is optional
- User input is never dropped

If memory does not fit, it is truncated or omitted.  
If conversation history does not fit, it is skipped.  
If identity or system sections do not fit, the request aborts.

---

## Prompt Assembly Model

SynthCore assembles a **strict, ordered, five-section prompt**:

```

SYSTEM
IDENTITY SNAPSHOT
MOOD STATE (non-narrative)
RELEVANT MEMORY
CURRENT REQUEST

```

This structure is invariant.  
No section is reordered.  
No section is implicit.

This guarantees:
- Deterministic context packing
- Debuggable failures
- Stable behavior across turns

---

## Graceful Degradation

SynthCore is designed to degrade, not crash.

### Supported Degradation Paths

- Memory unavailable → skip memory
- Mood unavailable → baseline mood
- Identity unavailable → minimal skeleton
- Multiple failures → minimal viable prompt
- Partial failures → continue execution

Only one condition aborts execution:
> The primary LLM cannot be reached.

---

## Failure Guarantees

SynthCore guarantees the following:

- ❌ Never exceeds token budget
- ❌ Never violates identity invariants
- ❌ Never leaks cross-user data
- ❌ Never crashes due to subsystem failure
- ✅ Always produces a response if the LLM is reachable

This is enforced by design, not convention.

---

## Directory Structure

```

nexus/
├─ core/
│  ├─ orchestrator.py        # SynthCore main loop
│  ├─ token_budget.py        # Token accounting + allocation
│  └─ prompt_assembler.py   # Structured prompt builder
│
├─ identity/
│  ├─ state.py               # Identity snapshots + invariants
│
├─ mood/
│  ├─ mood.py                # PAD model + decay
│
├─ memory/
│  ├─ retrieval.py           # Episodic retrieval
│  ├─ consolidation.py       # Nightly consolidation
│
└─ tests/
├─ test_token_budget.py
├─ test_identity.py
├─ test_orchestrator.py

```

---

## Key Design Principles

1. **Orchestration over intelligence**  
   Intelligence lives in models. Control lives in SynthCore.

2. **Explicit contracts beat clever heuristics**  
   Every subsystem has clear inputs, outputs, and failure modes.

3. **Budget first, generation second**  
   Tokens are a resource, not an afterthought.

4. **Degradation is a feature**  
   Failure is expected. Cascades are not.

5. **Identity is enforced, not implied**  
   Post-generation validation is mandatory.

---

## Phase 1 Scope

Phase 1 intentionally includes:
- Single-process orchestration
- AsyncIO-based execution
- Static identity kernel
- Time-decay-only mood model
- Episodic memory retrieval
- Nightly consolidation (offline)

Phase 1 intentionally excludes:
- Plugins
- Tool execution
- Identity learning
- Semantic fact retrieval
- Event buses (Kafka)
- Distributed orchestration

This keeps the foundation solid.

---

## Phase 2 Forward Compatibility

SynthCore is explicitly designed to support:

- Plugin orchestration
- Permissioned tool execution
- Event-driven scaling
- Identity learning vectors
- Appraisal-based mood updates
- Multi-agent coordination

Phase 2 extends SynthCore; it does not replace it.

---

## Status

**Phase 1**: Specification complete, ready for implementation  
**Production Readiness**: High  
**Primary Risk**: None unidentified  
**Next Step**: Implement Step 1 (TokenBudget + Prompt Assembly)

---

### Final Note

SynthCore is the part of the Nexus system that prevents everything else from becoming a liar, a goldfish or a runaway expense generator.

Treat it accordingly.
