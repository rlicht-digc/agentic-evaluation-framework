# Computation Prompt Template

A battle-hardened template for structuring agent computations with built-in guardrails.

This is a redacted version of the actual prompt template used in a 400+ computation research program. The specific content has been removed; the structure is the contribution.

---

## Template

```markdown
# Computation [N]: [Title]

## Objective

[1-2 sentences. What question does this computation answer?]

## NON-NEGOTIABLE RULES

1. Every numerical value must have a provenance tag:
   COMPUTED (with step reference) | CITED (with source) | ASSUMED (with justification)
2. Do NOT use any value from the quarantine list (loaded at session start)
3. Do NOT import results at a higher grade than their current classification
4. If a step requires an approximation, state it explicitly and bound the error
5. A successful computation is NOT one that confirms the hypothesis.
   A successful computation IS one that honestly answers the question.

## Inputs

| Input | Value | Source | Grade | Tag |
|-------|-------|--------|-------|-----|
| A | [value] | Computation [M], result D3 | ESTABLISHED | CITED |
| B | [value] | Published data, [reference] | PROVEN | CITED |
| C | [value] | Assumed for this computation | N/A | ASSUMED |

### Provenance Notes
- A was last verified in Computation [M] on [date]. Current grade: [grade].
- B is from [specific table/column] in [specific paper].
- C is assumed because [reason]. Sensitivity test in Step 5 varies C by +/- 50%.

## Hard-Fail Criteria (Pre-Registered)

Before starting, define what would KILL, leave INCONCLUSIVE, or SUPPORT:

| Outcome | Criterion | What It Means |
|---------|-----------|---------------|
| **KILL** | Magnitude off by > 2 orders | Mechanism fundamentally doesn't work |
| **KILL** | Wrong sign | Mechanism produces opposite effect |
| **KILL** | Wrong functional form | Mechanism gives linear when need sqrt |
| **INCONCLUSIVE** | Within 1-2 orders, right form | Missing factor possible |
| **SUPPORT** | Within 1 order, right form, no free params | Mechanism works at this level |

## Null Comparison

Every computation must compare against at least one null hypothesis:

- **Null H0**: [The simplest explanation that doesn't invoke the mechanism]
- **How to distinguish**: [What observable would differ between mechanism and null?]
- **Straw-man check**: Is H0 trivially easy to beat? If so, strengthen it.

## Computation Steps

### Step 1: [Description]
[Detailed instructions]

### Step 2: [Description]
[Detailed instructions]

...

### Step N: Adversarial Audit

After completing the computation, run these checks:
1. **Sign check**: Does every sign make physical sense?
2. **Dimensional analysis**: Do all expressions have correct units?
3. **Limiting cases**: Do known limits reproduce known results?
4. **Sensitivity**: How much does the result change if inputs vary by 10%?
5. **Red team**: What's the most likely way this result could be wrong?

## Output Requirements

### Derived Results
For each result, state:
- The result (explicit, with units)
- Its evidence grade (from the taxonomy)
- What would upgrade it
- What would kill it

### Source Map
Every equation maps to: derived here | from Computation N | from [paper]

### Verdict
One of: CONFIRMED | PARTIAL | KILLED | BLOCKED | INCONCLUSIVE

With specific numbers for the failure (if killed) or the match (if confirmed).

### Blocked Steps
Any step that could not be completed, with:
- What specifically is blocked
- What information or capability would unblock it
- Whether the block affects the verdict
```

---

## Why This Structure Works

### Pre-registered kill criteria prevent goalpost-moving

Without pre-registration, the agent will reinterpret any result as "partially successful." With pre-registration, a magnitude gap of 98 orders is a KILL, full stop. The agent can't retroactively decide that 98 orders is "the right direction."

### Null comparison prevents confirmation bias

Without a null, the computation can only confirm. "We found X = 1.23, consistent with the hypothesis" sounds great until you realize the null hypothesis also predicts X = 1.23. The null comparison forces the agent to show that the mechanism adds something beyond the trivial explanation.

### Provenance tags prevent stale-value propagation

Every input is tagged with its source and current grade. If the source is later updated or quarantined, the tag makes it possible to find and update all downstream uses.

### The adversarial audit catches systematic errors

The audit step (especially "red team: what's the most likely way this is wrong?") catches errors that confirmatory analysis misses. In practice, the red-team question has caught:
- Sign errors in intermediate steps
- Off-by-one errors in power counting
- Circular reasoning (the input already assumed the answer)
- Scope errors (result valid in one regime, applied in another)

### "Successful = honest" reframes the incentive

The most important line in the template: *"A successful computation is NOT one that confirms the hypothesis."* This explicitly counteracts the agent's bias toward confirmation. The agent's job is to answer the question honestly, not to make the hypothesis work.
