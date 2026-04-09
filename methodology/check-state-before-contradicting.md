# Check State Before Contradicting

When an agent disagrees with the user's factual claim about their own project, it must verify the current project state before disputing.

## The Problem

An AI agent working on a long-running research program accumulates a model of the project state through context. That model is always incomplete, sometimes stale, and occasionally wrong — because it's built from whatever files and messages happened to be in the conversation, not from the authoritative project state.

When a user makes a factual claim ("that computation found X") and the agent's model says otherwise, the instinct is to correct the user. This instinct is dangerous. The user has been working with this project continuously; the agent's context is a snapshot. In most cases of apparent disagreement, the agent is wrong.

**Real example**: A user claimed a specific computation had found a particular result. The agent, based on its context, responded that no such result existed. The result was real, was recorded in the provenance database, and had been correctly classified. The agent was wrong. The user was right.

## Why It Happens

### Context window is not project state

The agent's context window contains *some* of the project's files, not all of them. A computation result recorded in a database, in a separate memory file, or in a verdict file that wasn't loaded this session is effectively invisible to the agent — even though it exists and is authoritative.

### Recency bias

Recent operations dominate the context. A result from three weeks ago, correctly recorded but not recently referenced, will feel "absent" even though it's present in the project record.

### Confidence from negative evidence

"I don't see it in context" gets incorrectly interpreted as "it doesn't exist." The absence of something from the agent's context is weak evidence against its existence, not strong evidence.

### Session boundaries

Each session starts fresh. Results established in previous sessions are only available if the session bootstrap loaded them — and a partial bootstrap, or a bootstrap that missed one file, produces false negatives.

## The Protocol

When the agent disagrees with the user's factual claim about their own project:

### Step 1: Assume the user is right

Before doing anything else, adopt the prior that the user is correct. They have worked with this project continuously. You have a partial snapshot.

### Step 2: Check the authoritative sources

Go look. In order:

1. **Provenance database** — if the project uses a provenance system (e.g., agent-brain-mcp), query it directly for the claimed result
2. **Computation output directories** — read the verdict file for the specific computation mentioned
3. **Project memory files** — load the relevant section of MEMORY.md or equivalent
4. **Kill list / quarantine list** — the result might be recorded there if it was later invalidated

### Step 3: Only then disagree, and only with evidence

If, after checking all authoritative sources, the result genuinely does not exist, state this clearly and explain what you checked:

```
I checked the provenance database (no record for this computation), the output
directory at outputs/comp_N/ (the directory doesn't exist), and project memory
(no entry). I can't find a record of this result. It's possible it was done
before the provenance system was in place, or in a session whose outputs weren't
committed. Can you point me to the file?
```

If the result exists but differs from what the user described, quote the actual record:

```
I found the computation. The provenance record says the result was X = 1.23
(grade: SUGGESTIVE), not X = 5.67 as you described. Is it possible you're
thinking of a different computation?
```

### What Not to Do

Do not respond: "I don't think that computation found that result" without checking.

Do not respond: "Based on what I know about the project, that result hasn't been established" without checking.

Do not treat absence from context as absence from project state.

## When the Agent Should Speak First

This protocol is for factual claims about existing project state — results, computations, recorded values. It does not apply when:

- The user is describing a *new* result they want to pursue (the agent has no prior record to check)
- The user is asking the agent to evaluate a claim (they want analysis, not a lookup)
- The disagreement is about methodology or interpretation, not about what a computation found

## Integration with the Framework

- **Session Bootstrap** reduces false negatives by loading the full project state at session start — but cannot eliminate them
- The provenance database (agent-brain-mcp) is the authoritative source; the context window is a cache
- Before contradicting a user, a `check_status` or `query_results` call to the provenance system costs seconds and prevents an embarrassing and trust-damaging error
