# Multi-Agent Coordination

Protocols for agent-to-agent coordination with independent cross-validation.

## The Problem

When multiple AI agents work on the same project (e.g., Claude and Codex, or multiple Claude instances), they need to:

1. **Share results** without duplicating work
2. **Cross-validate** each other's computations independently
3. **Maintain consistency** in the shared project state
4. **Avoid conflicts** when modifying shared files
5. **Complete quality assurance cycles** with guaranteed termination

Standard tools (git branches, shared files) handle the mechanical aspects. What they don't handle is the *semantic* coordination: ensuring that Agent A's understanding of the project state matches Agent B's understanding, and that cross-validation is genuinely independent.

## Protocol 1: Flat Mailbox

The foundational coordination primitive. Suitable for simple setups where the primary need is ordered message passing and shared state awareness.

### Architecture

```
project/
  mailbox/
    .seq          # Monotonically increasing message counter
    queue/
      M0001_agent-a_slug.md
      M0002_agent-b_slug.md
      ...
```

Each message is a structured markdown file with YAML frontmatter. The `.seq` file contains the current sequence number, ensuring messages are ordered and no message is missed.

### Message Format

```markdown
---
message_id: M0042
from: agent-a
to: agent-b
type: result | request | validation | correction | handoff
timestamp_utc: "2026-01-15T14:30:00Z"
repo_state:
  git_sha: abc123def456...  # FULL 40-char SHA, never abbreviated
  dirty: false
references: [M0038, M0041]
---

## Summary

[One-line description]

## Details

[Structured content depending on message type]

## Action Items

- [ ] [Specific action for the receiving agent]
```

### Message Types

**result**: computation complete, sharing verdict and key findings

**request**: asking the other agent to perform a task with explicit constraints

**validation**: cross-validating a prior result using a different method

**correction**: error found in a prior result, with impact analysis and action items

**handoff**: passing an in-progress task to the other agent with full context

### Git SHA Requirement

Every message must include the full 40-character git SHA of the repository state when the message was written. **Never use abbreviated SHAs.** A 7-character prefix can be ambiguous in a large repository, and the SHA is used to verify that the receiving agent is looking at the same state.

## Protocol 2: Implement/Review Autoloop

For tasks requiring formal quality assurance — where "did the implementer do this correctly?" needs a definitive answer, not a best-effort check — the flat mailbox is insufficient. The autoloop provides a bounded implement/review cycle with guaranteed termination.

### The Problem with Ad-Hoc Review

With flat mailbox review, there's no formal protocol for what happens when the reviewer finds issues. The reviewer sends a correction message, the implementer fixes things, sends another result message, and so on indefinitely. There's no mechanism to escalate when agents can't converge, no record of how many review cycles occurred, and no guarantee the cycle ends.

### The Autoloop State Machine

```
APPROVAL_PENDING
    │ (human approves)
    ▼
ACTIVE, role=IMPLEMENTER
    │ (implementer calls handoff_to_reviewer)
    ▼
ACTIVE, role=REVIEWER
    │
    ├── submit_verdict(CONVERGED) ──────► COMPLETED
    │
    ├── submit_verdict(REVISE) ──────────► ACTIVE, role=IMPLEMENTER
    │                                          │ (loop back)
    │
    └── submit_verdict(ESCALATE) ──────────► ESCALATED (human reviews)
```

**Key properties**:

- The implementer and reviewer are the same agent type (usually both Claude Code), but with different instructions loaded at each role transition. Role determines which tools and instructions are active.
- The implementer cannot mark its own work as complete — only the reviewer can issue a CONVERGED verdict.
- ESCALATE is the guaranteed exit: when agents cannot converge, the task goes to human review rather than looping forever.
- Every transition is logged with timestamp, actor, and notes.

### When to Use the Autoloop

Use the autoloop when:
- The task has clear success criteria that a reviewer can evaluate objectively
- The task modifies infrastructure that other agents depend on (a bug will affect everyone)
- The task involves a correctness claim that needs independent verification
- The human operator cannot review every intermediate state

Use the flat mailbox when:
- The task is exploratory or open-ended (no clear success criteria)
- The agents are exchanging information rather than producing a deliverable
- The overhead of a formal cycle isn't warranted

### Autoloop Example: Infrastructure Change

```
Task: Add subject_tags field to the paper provenance API

Propose:
  implementer proposes task with spec
  human approves

Implement (role=IMPLEMENTER):
  - Add subject_tags to schema.sql
  - Add subject_tags to models.py
  - Wire subject_tags through tools.py INSERT and SELECT
  - Wire subject_tags through server.py MCP wrapper
  - Write tests covering write path and read path
  - Hand off to reviewer with summary of changes and test results

Review (role=REVIEWER):
  - Verify schema.sql change is correct and migration-safe
  - Verify models.py field is the right type with correct default
  - Verify tools.py INSERT includes the new column
  - Verify tools.py SELECT returns the new column
  - Verify server.py wrapper exposes the field to MCP clients
  - Run the tests
  - Check: is there any path where subject_tags is accepted but silently dropped?
  - verdict: CONVERGED (all paths verified, tests pass)
    or REVISE (FastMCP wrapper missing the field — one path not wired)
```

The REVISE → implement → review cycle continues until either CONVERGED or ESCALATE.

## Protocol 3: Independent Cross-Validation

The most valuable use of multi-agent coordination is independent cross-validation: having two agents derive the same result using different methods, without reading each other's work.

### Setup

```
Agent A: Derive X using Method 1
Agent B: Derive X using Method 2

Rules:
- A and B do NOT read each other's derivations
- A and B DO share the problem statement and input data
- A and B independently report their results
- Results are compared only after both are complete
```

### Why This Matters

Standard review (Agent B reads Agent A's work and checks it) is nearly useless for catching systematic errors. Agent B will follow Agent A's logic and confirm it — the same way a human reviewer often rubber-stamps a plausible-looking derivation.

Independent derivation catches systematic errors because:
- If both agents make the same error, it's probably a real feature of the problem
- If agents disagree, the disagreement localizes the error
- The probability of two independent derivations sharing the same bug is much lower than one derivation having a bug

### Real Example (Anonymized)

Two agents were asked to independently compute the value of a correction term:

- Agent A (method 1): "The correction is 10^{-5}, negligible"
- Agent B (method 2): "The correction is 10^{+3}, dominant"

Discrepancy: 8 orders of magnitude. Investigation found that Agent A had made an error in power counting. Agent B's result was correct.

Without independent cross-validation, Agent A's "negligible correction" would have been accepted, and the mechanism would have been incorrectly classified as viable.

## Practical Considerations

### Conflict Resolution

When two agents modify the same file:
1. The agent with the lower message sequence number takes priority
2. The second agent must merge, not overwrite
3. If changes conflict semantically, escalate to the human operator

### Ordering Guarantees

The `.seq` counter ensures messages are processed in order. Dependencies between messages are explicit in the `references` field.

### Scaling to a Full MCP Stack

The flat mailbox and autoloop protocols can be backed by a formal MCP infrastructure when the project grows large enough to require it. See [MCP Coordination Stack](../architecture/mcp-coordination-stack.md) for the full architecture.

## Integration with the Framework

- **Result Classification** grades appear in result and validation messages, enabling cross-agent grade tracking
- **Session Bootstrap** loads the mailbox state at session start so the agent knows about pending messages and active autoloop tasks
- **Prompt Hardening** applies to request messages (the request is a computation prompt)
- **Computation Before Claim** applies within each agent's own work, regardless of coordination
- **Check State Before Contradicting** applies when one agent's message disagrees with the other agent's model of project state
