# Computation Before Claim

Enforces the rule that no quantitative claim may be stated without a supporting computation.

## The Problem It Solves

Large language models are confident. When asked a quantitative question, they produce a specific number with an authoritative tone. The number might come from:

1. **Training data** -- The model recalls a value from a paper it was trained on
2. **Reasoning by analogy** -- The model estimates based on similar problems
3. **Confabulation** -- The model generates a plausible-sounding number that has no basis

In standard chat interactions, this is a minor issue -- the user can verify. In extended agentic sessions where the agent is conducting multi-step research, it becomes dangerous:

- The agent states a number in Computation 12
- Computation 15 uses that number as an input
- Computation 20 builds on Computation 15's result
- By Computation 30, the original unverified number is deeply embedded in the dependency graph

The deeper problem: the agent doesn't distinguish between "I computed this" and "I stated this." Both feel equally real in the context window.

## The Rule

**No quantitative claim without a computation.**

If the agent needs a number, it must either:
1. **Compute it** -- write code, run it, get the answer
2. **Look it up** -- read it from a specific file, paper, or database, and cite the source
3. **Declare it assumed** -- explicitly state "I am assuming X = [value] because [reason]" and tag it as an assumption

The agent must NEVER:
- State a numerical result "from reasoning" without computation
- Round or approximate a previously computed value without flagging the approximation
- Combine two computed values without showing the combination

## When It Triggers

Continuously, during any computation or analysis. The enforcement is implicit in the project's culture: every value that appears in a result must have a traceable provenance.

## What It Does

### At the point of any quantitative claim:

```
Before stating "X = [number]":
  1. Was this number computed in the current session?
     -> Cite the computation step
  2. Was it computed in a prior computation?
     -> Cite the computation number and its current grade
  3. Is it from published data?
     -> Cite the source (paper, table, column)
  4. Is it an assumption?
     -> Tag it: "ASSUMED: X = [value] because [reason]"
  5. None of the above?
     -> DO NOT STATE IT. Compute it first.
```

### For derived quantities:

```
If X = f(A, B) where A and B are known:
  1. Show the computation: X = f(A, B) = f([value], [value]) = [result]
  2. Cite sources for A and B
  3. State the formula f and where it comes from
```

## Example: Blocking a Confident Wrong Number

**The setup**: During a computation investigating whether a certain mechanism could produce a specific effect, the agent needed to estimate the magnitude of a correction term.

**What happened without the rule**: The agent stated "The correction is of order 10^{-5}" based on general reasoning about the scales involved. This sounded right. It was used in the next step. The computation concluded the correction was negligible.

**What the rule caught**: When enforcing "computation before claim," the agent was required to actually compute the correction. The actual computation showed the correction was 10^{+111} -- not 10^{-5}. The difference: 116 orders of magnitude. The "negligible correction" was in fact a catastrophic divergence that killed the mechanism entirely.

**Why this matters**: The agent's intuitive estimate was not just wrong -- it was wrong in the direction that confirmed the hypothesis. This is a systematic bias: when an agent *wants* a mechanism to work (because the prompt asks it to test the mechanism), it is more likely to estimate corrections as small.

## Example: Catching Training-Data Contamination

**The setup**: The agent needed a specific constant from a theoretical framework. The constant appears in published papers with a standard value.

**What happened without the rule**: The agent stated the constant from memory. The value was correct *for the standard framework in the literature* but wrong for the user's novel framework, which defines the constant differently.

**What the rule caught**: By requiring an explicit source ("compute it or cite it"), the agent was forced to either derive the constant from the user's definitions or cite a specific source. This revealed the discrepancy: the literature value uses convention A, the user's framework uses convention B, and the two differ by a factor of 2.

## How to Use It

### In CLAUDE.md:

```markdown
## Computation Before Claim (Mandatory)

Every numerical value in a computation must have one of these tags:
- COMPUTED: [step where it was computed]
- CITED: [source file, paper, computation number]
- ASSUMED: [value, reason for assumption]

Do not state numerical results from "reasoning" without computation.
If you need a number, compute it.
```

### In computation templates:

```markdown
## Source Map

Every equation and value used in this computation:

| Item | Value | Source | Tag |
|------|-------|--------|-----|
| A | 1.23e-10 | Computation 45, D3 | CITED |
| B | 3.14 | Computed in Step 3 below | COMPUTED |
| C | 0.5 | Standard assumption, justified in A2 | ASSUMED |
```

### As a runtime check:

When the agent produces a verdict that includes a number, verify:
- Can you trace that number back to a specific line of code?
- Can you re-run that code and get the same number?
- If you change the inputs, does the number change as expected?

If any answer is "no," the number is not computed -- it's confabulated.

## The Asymmetry Problem

This rule is especially important when results are negative (killing a hypothesis). The agent naturally wants to be helpful, which biases it toward finding ways to make things work. The most common manifestation:

- "The correction is small" (without computing it) -- saves the hypothesis
- "The sign works out" (without checking) -- saves the hypothesis
- "The orders of magnitude are consistent" (without counting) -- saves the hypothesis

Computation Before Claim is the antidote: force the agent to actually compute the correction, check the sign, and count the orders of magnitude. The result is often the opposite of what "reasoning" suggested.

## Integration with Other Skills

- **Result Classification** assigns grades to computed results; Computation Before Claim ensures the computation exists in the first place
- **Prompt Hardening** sets up the structure (source maps, assumption lists) that makes this rule enforceable
- **Session Bootstrap** loads prior computations so the agent can cite them rather than re-derive from memory
