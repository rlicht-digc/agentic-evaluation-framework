# MCP Coordination Stack Architecture

## Overview

The flat mailbox protocol described in [multi-agent-coordination](../examples/multi-agent-coordination.md) solves message passing between agents. It does not solve:

- **State consistency**: both agents need to agree on what is true about the project, not just what messages have been sent
- **Quality assurance cycles**: formal protocols for an implementer to hand off to a reviewer and for the reviewer to return a verdict with termination guarantees
- **Provenance tracking**: recording what was computed, by whom, at what evidence grade, and how results depend on each other
- **Drift prevention**: ensuring that the tool surface and schema the agents use stays consistent with documentation

The MCP coordination stack addresses these gaps with three specialized servers, each responsible for a distinct concern.

## Three-Server Architecture

```
Agent (Claude Code or Codex)
    │
    ├── Science Provenance Server (17 tools)
    │   └── Canonical record of what was computed and what it means
    │       Results, evidence grades, outputs, papers, inter-result links
    │       Read-only from coordination layer
    │
    ├── Agent Message Bus (27 tools)
    │   └── Transport and control plane
    │       Registration, sessions, messaging, intent declaration,
    │       implement/review autoloop cycles, action logging
    │
    └── Shared Brain (8 tools)
        └── Coordination state
            Entities, claims, decisions with visibility-scoped access
            Open questions with structured resolution
```

Each server is a FastMCP stdio process. Agents interact via tool calls only — no direct database access, no shared memory, no filesystem side channels.

## Agent Message Bus

The AMB is the successor to the flat mailbox protocol. It provides everything the mailbox provides — ordered messages, SHA tracking, structured handoffs — plus three new primitives:

### Intent Declaration

Before any mutating action, an agent must declare its intent:

```python
intent_id = amb_log_intent(
    agent_id=my_agent_id,
    intent_type="file_modification",
    description="Updating verdict file for computation N",
    target_ref="outputs/comp_N/verdict.md"
)
# ... perform the action ...
amb_log_action(agent_id=my_agent_id, intent_id=intent_id, ...)
```

This creates an audit trail that links every action to the stated intent that authorized it. Actions without a prior intent declaration are flagged.

### Implement/Review Autoloop

The most significant addition over the flat mailbox. Instead of ad-hoc review, the AMB provides a formal bounded cycle:

```
amb_propose_autoloop_task(title, description, domain)
    → task created in APPROVAL_PENDING state

amb_approve_autoloop_task(task_id)
    → task transitions to ACTIVE, role=IMPLEMENTER

[implementer works on the task]

amb_handoff_to_reviewer(task_id, summary, artifacts)
    → role transitions to REVIEWER
    → implementer can no longer modify the task

[reviewer evaluates the work]

amb_submit_verdict(task_id, verdict, notes)
    where verdict ∈ {CONVERGED, REVISE, ESCALATE}
```

**CONVERGED**: The reviewer accepts the implementation. Task is done.

**REVISE**: The reviewer found issues. Task returns to IMPLEMENTER with the reviewer's notes. The cycle repeats.

**ESCALATE**: The reviewer found issues that cannot be resolved in the current cycle — fundamental disagreement, out-of-scope changes needed, human judgment required. Task is held for human review.

The key constraint: the cycle is *bounded*. It cannot loop forever. ESCALATE provides a guaranteed exit when agents cannot converge on their own.

### Action Logging

All agent actions are logged to an append-only action log with intent provenance. This provides:
- Full audit trail of what each agent did and why
- Ability to reconstruct what state looked like at any point
- Detection of agents acting outside their declared intents

## Shared Brain

Shared Brain holds the coordination state that multiple agents need to agree on. Unlike the message bus (which is about communication) and provenance (which is about scientific record), Shared Brain is about *current truth*:

- **Entities**: named things that exist in the project (computations, papers, constants, datasets)
- **Claims**: atomic facts about entities, with supersession (publishing a new claim retires the old one)
- **Decisions**: proposed → accepted transitions with justification links
- **Open questions**: raised → answered/deferred/contested/invalid with structured resolution

Visibility scoping means an agent can checkpoint its view of domain state before starting a task, ensuring it's working from a consistent snapshot rather than a shifting live state.

## Drift Detection

In a complex MCP stack with 52 tools across 3 servers, the documented tool surface can drift from the actual tool surface. This is caught by `infra_audit.py`:

```bash
python3 agent-bus/infra/infra_audit.py --check
# Diffs live server tool inventory vs snapshot + ARCHITECTURE.md canary
# Returns non-zero exit code if drift detected

python3 agent-bus/infra/infra_audit.py --apply
# Updates ARCHITECTURE.md canary to match live state, re-snapshots
```

A pre-commit hook runs `--check` before any commit touching infra files. If the tool count in ARCHITECTURE.md doesn't match the live server, the commit is blocked.

This prevents the situation where an agent is told the server has N tools (from documentation) when the server actually has N+2 — leading to hallucinated tool calls for tools that don't exist, or missed calls for tools the agent didn't know about.

## Phase Tagging

Infrastructure changes are versioned with git tags and a formal phase ledger:

```
stable/phase-2b-a → f31dbbf
    Phase 1: AMB core (agent registration, messaging, channels, action logging)
    Phase 2A: Shared Brain
    Phase 2B-A: Implement/review autoloop (7 new tools)
```

A CHANGELOG.md is generated from Conventional Commits on each phase tag using git-cliff. A BUG_REGISTER.md tracks every known bug with its resolution and propagation analysis.

This creates rollback points: if a new phase introduces a regression, the previous stable tag is a known-good state.

## Relationship to the Flat Mailbox

The flat mailbox protocol remains valid for simpler setups. The MCP coordination stack is appropriate when:

- Multiple long-running agents are active simultaneously
- Scientific results need to be tracked with formal evidence grades
- Implement/review cycles need termination guarantees
- Audit trails are needed for compliance or debugging
- The project has grown large enough that "what is true about this project" is no longer answerable from a single file

The protocols are compatible: a project can use the flat mailbox for day-to-day coordination and graduate to the full MCP stack as complexity grows.

## Public Implementation

The Science Provenance server is available as an open-source library: [agent-brain-mcp](https://github.com/rlicht-digc/agent-brain-mcp). The Agent Message Bus and Shared Brain are private but documented here as architecture patterns.
