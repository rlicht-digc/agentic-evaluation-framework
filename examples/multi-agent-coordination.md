# Multi-Agent Coordination

A mailbox protocol for agent-to-agent coordination with independent cross-validation.

## The Problem

When multiple AI agents work on the same project (e.g., Claude and Codex, or multiple Claude instances), they need to:

1. **Share results** without duplicating work
2. **Cross-validate** each other's computations independently
3. **Maintain consistency** in the shared project state
4. **Avoid conflicts** when modifying shared files

Standard tools (git branches, shared files) handle the mechanical aspects. What they don't handle is the *semantic* coordination: ensuring that Agent A's understanding of the project state matches Agent B's understanding, and that cross-validation is genuinely independent.

## The Mailbox Protocol

### Architecture

```
project/
  .mailbox/
    .seq          # Monotonically increasing message counter
    msg_001.md    # First message
    msg_002.md    # Second message
    ...
```

Each message is a structured markdown file that one agent writes and another reads. The `.seq` file contains the current sequence number, ensuring messages are ordered and no message is missed.

### Message Format

```markdown
---
seq: 42
from: agent-A
to: agent-B
type: result | request | validation | correction
timestamp: 2026-03-15T14:30:00Z
repo_state:
  git_sha: abc123def456...  # FULL 40-char SHA, never abbreviated
  branch: main
  clean: true               # No uncommitted changes
depends_on: [38, 41]        # Prior messages this depends on
---

## Subject

[One-line description]

## Body

[Structured content depending on message type]

## Action Items

- [ ] [Specific action for the receiving agent]
- [ ] [Another action]

## State Changes

- [What changed in the project state as a result of this message]
```

### Message Types

#### Result Message

Sent when an agent completes a computation and wants to share the result.

```markdown
---
type: result
---

## Computation 45: [Title]

### Verdict: KILLED

### Key Results
- D1: [result] -- Grade: FAILED -- [specific failure]
- D2: [result] -- Grade: PARTIAL -- [conditions]

### Files Modified
- outputs/computation45/verdict.md (NEW)
- outputs/computation45/report.md (NEW)
- memory/MEMORY.md (UPDATED: added Comp 45 entry)

### Action Items
- [ ] Cross-validate D2 independently (do not read my derivation first)
- [ ] Update project memory to reflect the kill
```

#### Request Message

Sent when an agent needs another agent to perform a computation.

```markdown
---
type: request
---

## Request: Independent Derivation of Parameter X

### Context
I derived X = 1.23 in Computation 44. I need an independent check
using a DIFFERENT method.

### Constraints
- Do NOT read Computation 44's derivation
- Use method B (I used method A)
- Report your value with error estimate
- If your value disagrees with 1.23 by more than 10%, flag it

### Why Independent
Cross-validation is only meaningful if the second derivation is
genuinely independent. Reading the first derivation introduces
anchoring bias.
```

#### Validation Message

Sent when an agent has cross-validated another agent's result.

```markdown
---
type: validation
---

## Cross-Validation of Computation 44, Result D2

### Method Used: [different from original]
### My Result: X = 1.25 +/- 0.03
### Original Result: X = 1.23

### Agreement: YES (within error bars)
### Independence: VERIFIED (used method B, did not read method A derivation)

### Notes
- My error bar is larger because method B has fewer constraints
- Both methods agree on the sign and order of magnitude
- Recommend upgrading D2 from SUGGESTIVE to ESTABLISHED
```

#### Correction Message

Sent when an agent finds an error in a prior computation.

```markdown
---
type: correction
---

## Correction to Computation 44, Step 3

### Error Found
Sign error in the second term of the expansion. Line 47 of
computation44/derivation.py has (-1)^n where it should be (-1)^(n+1).

### Impact
- D1 is unaffected (doesn't use this term)
- D2 changes from X = 1.23 to X = -0.45 (sign flip + magnitude change)
- D2's grade should be downgraded to QUARANTINED until corrected

### Downstream Effects
- Computation 45 used D2 as input -- results invalidated
- Computation 47 cited D2 but didn't use the value -- unaffected

### Action Items
- [ ] Fix the sign error in computation44/derivation.py
- [ ] Re-run Computation 44 with the fix
- [ ] Re-run Computation 45 with corrected input
- [ ] Update project memory: quarantine old D2 value
```

## Independent Cross-Validation Protocol

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

Standard agent review (Agent B reads Agent A's work and checks it) is nearly useless for catching systematic errors. Agent B will follow Agent A's logic, nod along, and confirm it -- the same way a human reviewer often rubber-stamps a plausible-looking derivation.

Independent derivation catches systematic errors because:
- If both agents make the same systematic error, it's probably a real feature of the problem
- If the agents disagree, the disagreement localizes the error
- The probability of two independent derivations having the same bug is much lower than one derivation having a bug

### Real Example (Anonymized)

Two agents were asked to independently compute the value of a correction term:

- Agent A (method 1): "The correction is 10^{-5}, negligible"
- Agent B (method 2): "The correction is 10^{+3}, dominant"

Discrepancy: 8 orders of magnitude. Investigation found that Agent A had made an error in power counting (missed a factor of the fundamental scale raised to the 8th power). Agent B's result was correct.

Without independent cross-validation, Agent A's "negligible correction" would have been accepted, and the mechanism would have been incorrectly classified as viable.

## Practical Considerations

### Git SHA Requirement

Every message must include the full 40-character git SHA of the repository state when the message was written. This ensures:
- The receiving agent can check out the exact state the sending agent was working with
- If files have changed since the message was sent, the discrepancy is visible
- No ambiguity about which version of the code or data was used

**Never use abbreviated SHAs.** A 7-character prefix can be ambiguous in a large repository.

### Ordering Guarantees

The `.seq` counter ensures messages are processed in order. This matters because:
- A correction message must be processed before any message that depends on the corrected result
- A validation message must reference the specific version it validated
- Dependencies between messages are explicit in the `depends_on` field

### Conflict Resolution

When two agents modify the same file:
1. The agent with the lower `.seq` number takes priority
2. The second agent must merge, not overwrite
3. If the changes conflict semantically (not just textually), escalate to the user

## Integration with the Framework

- **Result Classification** grades appear in result messages, enabling cross-agent grade tracking
- **Session Bootstrap** loads the mailbox state at session start, so the agent knows about pending messages
- **Prompt Hardening** applies to request messages (the request itself is a computation prompt)
- **Computation Before Claim** applies within each agent's own work, regardless of coordination
