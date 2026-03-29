# Session Bootstrap

Loads critical project state from disk at session start, ensuring the agent knows what is alive, dead, and quarantined before answering any question.

## The Problem It Solves

Every new Claude Code session starts with a blank slate. The agent has access to project files, but it doesn't *know* the current state of the project until it reads them. This creates a window of vulnerability:

1. **Stale citations** -- The agent answers a question about a result that was killed three sessions ago, because it hasn't loaded the current kill list
2. **Quarantine violations** -- The agent uses a value that was quarantined, because the quarantine list lives in a file it hasn't read yet
3. **Dead-route exploration** -- The agent suggests investigating a mechanism that has already been tested and killed with a specific numerical failure
4. **Version confusion** -- The agent references an old version of a result when a corrected version exists

In a research program with 400+ computations and 50+ project memory files, the state of "what is alive, what is dead, what is quarantined" is not obvious from any single file. It's distributed across kill lists, quarantine lists, verdict files, and memory documents.

## When It Triggers

At the start of every new session, before any user question is answered.

## What It Does

### Step 1: Load Project Memory

Read the persistent memory file (e.g., `MEMORY.md` or equivalent) that contains:
- Completed computations and their verdicts
- Active quarantine list
- Key version history
- Cross-references between computations

### Step 2: Load Quarantine List

From `CLAUDE.md` or equivalent project instructions, extract:
- Every quarantined value
- Why it was quarantined
- What computation invalidated it
- What (if anything) replaced it

### Step 3: Load Kill List

From project memory, build the list of:
- Dead mechanisms (tested and failed, with specific failure numbers)
- Closed routes (entire classes of approach that have been ruled out)
- Retracted results (previously published but later found to be wrong)

### Step 4: Load Active State

Identify:
- Currently open computations (in progress)
- Results that are being actively used (and at what grade)
- The current "frontier" -- what questions are open, what's the next computation

### Step 5: Verify Consistency

Cross-check:
- No quarantined value appears in the "active" list
- No killed mechanism appears in the "open" list
- All cross-references between computations are consistent with current verdicts

## Example: Preventing a Dead-Route Suggestion

**The setup**: A user starts a new session and asks "What approaches haven't been tried for deriving quantity X?"

**Without bootstrap**: The agent reads a few recent files and suggests "try mechanism A" -- not knowing that mechanism A was tested in Computation 42 and killed with 6 independent failures.

**With bootstrap**: The session starts by loading project memory, which includes the kill list entry for mechanism A. The agent's response includes: "Mechanism A was tested in Computation 42 and killed (wrong sign, 98 orders of magnitude too small, universal obstruction in the counting statistics). The following approaches have NOT been tried: ..."

**The savings**: Without bootstrap, the agent would spend an entire session re-deriving something that was already proven impossible. With bootstrap, it immediately focuses on genuinely open routes.

## Example: Catching a Quarantine Violation

**The setup**: A script imports a constant from a shared configuration file. The constant was quarantined two weeks ago due to a bug in the solver that produced it.

**Without bootstrap**: The agent runs the script, gets results, and reports them -- not knowing the input was invalid.

**With bootstrap**: The quarantine list is loaded at session start. When the agent reads the script, it recognizes the quarantined value and flags it before execution: "This script uses parameter Y, which was quarantined on [date] due to [reason]. The current replacement is Z. Shall I update the script?"

## How to Use It

### In CLAUDE.md (auto-loaded by Claude Code):

```markdown
## Session Bootstrap (Auto-Load)

At the start of every session, load:
1. Project memory: [path to MEMORY.md]
2. Quarantine list: [section below or path]
3. Kill list: [embedded in project memory or separate file]

Do not answer questions about project state until these are loaded.

## Quarantine List (current as of YYYY-MM-DD)

| Value | Status | Invalidated By | Replacement |
|-------|--------|----------------|-------------|
| param_X | QUARANTINED | Computation 68 | param_X_v2 |
| solver_Y | QUARANTINED | Bug report #14 | solver_Y_fixed |
```

### As a slash command:

```markdown
---
description: Check a script for quarantined values
---

Scan the script at the provided path for references to quarantined values.

## Quarantined values (current):
- param_X (must not be used)
- solver_Y outputs (invalidated)
- Any results from Computation 30-35 (superseded by Computation 40)

## Procedure
1. Read the script
2. Search for quarantined identifiers
3. Report: PASS (clean) or FAIL (with specific line numbers)
```

### Memory file structure:

The project memory file should include, for each completed computation:
```
- Computation N: [Title] -- [VERDICT]. [1-line summary]. [Date].
```

Where VERDICT is one of: CONFIRMED, PARTIAL, KILLED, BLOCKED, RETRACTED.

This gives the bootstrap process a scannable index of all project state.

## Integration with Other Skills

- **Result Classification** defines the grades that bootstrap loads
- **Prompt Hardening** uses the quarantine list that bootstrap makes available
- **Computation Before Claim** can reference the kill list to prevent re-exploration of dead routes
