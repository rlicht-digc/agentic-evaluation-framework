# Prompt Hardening

Automatically self-reviews, red-teams, and battle-hardens any computation prompt before execution.

## The Problem It Solves

When an agent generates a computation prompt (instructions for itself or another agent to carry out a multi-step analysis), the prompt often contains:

1. **Unchecked assumptions** -- The prompt assumes a result that hasn't been verified, or uses a value that has been quarantined
2. **Missing null comparisons** -- The computation would confirm a hypothesis without ever testing whether a simpler explanation works
3. **No failure criteria** -- There's no pre-registered definition of what would count as the computation *failing*, so any result can be spun as a success
4. **Contaminated inputs** -- The prompt imports values or conclusions from a prior computation that has since been invalidated
5. **Numerology disguised as derivation** -- The prompt asks the agent to find a match between two quantities without requiring a mechanistic explanation for *why* they should match

These aren't edge cases. In a 400+ computation research program, every single one of these failure modes occurred multiple times, and each one wasted compute and created results that later had to be retracted.

## When It Triggers

Before any computation prompt is executed -- either automatically (when using a computation template) or when the user asks the agent to design a new computation.

## What It Does

The hardening process has five passes:

### Pass 1: Assumption Audit

```
For each claim or value used in the prompt:
  - Is it tagged with a provenance? (Which computation produced it? At what grade?)
  - Is it on the quarantine list?
  - Is its grade sufficient for how it's being used?

Flag: Any value used without provenance
Flag: Any SUGGESTIVE result treated as ESTABLISHED
Flag: Any quarantined value referenced
```

### Pass 2: Null Comparison

```
Does the computation include a comparison against:
  - A simpler model that could explain the same data?
  - A random/shuffled baseline?
  - The trivial explanation ("this is just a coincidence")?

Flag: Any computation that can only confirm, never falsify
```

### Pass 3: Hard-Fail Criteria

```
Does the prompt specify, in advance:
  - What numerical result would KILL the hypothesis?
  - What result would be INCONCLUSIVE?
  - What result would SUPPORT it (and at what evidence grade)?

Flag: Any computation with no pre-registered failure criteria
```

### Pass 4: Contamination Check

```
For each input to the computation:
  - Trace it back to its source
  - Check whether the source has been updated, corrected, or quarantined
  - Check whether the input is being used at the correct version

Flag: Any input from a computation that has been superseded
Flag: Any input whose source was RETRACTED
```

### Pass 5: Red Team

```
Ask: "If I wanted to make this computation produce a false positive,
      what would I exploit?"

Common exploits:
  - Fitting a model with enough free parameters to match anything
  - Comparing to a straw-man null that's easy to beat
  - Using a metric that rewards the hypothesis by construction
  - Cherry-picking the comparison scale or range
```

## Example: Catching a Contaminated Prompt

**The setup**: A computation prompt asked the agent to test whether a certain quantity matched a theoretical prediction. The prompt imported the theoretical prediction from an earlier computation.

**What hardening caught**: Pass 4 (contamination check) discovered that the earlier computation had been superseded. The value being imported was from the original version, not the corrected version. The corrected value didn't match the prediction at all.

**Without hardening**: The computation would have "confirmed" the match, adding another false data point to the evidence chain.

**What happened next**: The prompt was rewritten with the corrected value, explicit provenance tags, and a null comparison against the trivial hypothesis. The rewritten computation found that the match was a coincidence -- the corrected value was off by a factor of 3.

## Example: Missing Hard-Fail Criteria

**The setup**: A prompt asked "test whether mechanism X can produce effect Y."

**What hardening caught**: Pass 3 found no failure criteria. The prompt would accept *any* output of mechanism X as "producing" effect Y, because it didn't specify how close the match needed to be, what units to compare in, or what discrepancy would constitute failure.

**The fix**: Added explicit criteria:
- KILL: if the magnitude is off by more than 2 orders
- KILL: if the sign is wrong
- INCONCLUSIVE: if the magnitude is off by 1-2 orders
- SUPPORT: if within 1 order AND the functional form matches

**Outcome**: The mechanism was off by 98 orders of magnitude. Without failure criteria, the agent might have focused on the fact that it got the right *sign* and called the computation "partially successful."

## How to Use It

### As a pre-flight checklist in computation templates:

```markdown
## Pre-Flight Checklist (complete before starting)

### Assumptions
- [ ] Every input value has a provenance tag: [Source computation, Grade]
- [ ] No quarantined values used
- [ ] No SUGGESTIVE results treated as ESTABLISHED

### Null Comparison
- [ ] Simpler alternative specified: ___
- [ ] Random/trivial baseline specified: ___

### Hard-Fail Criteria
- [ ] KILL if: ___
- [ ] INCONCLUSIVE if: ___
- [ ] SUPPORT if: ___

### Contamination
- [ ] All inputs traced to current versions
- [ ] No retracted sources used

### Red Team
- [ ] Most likely false-positive exploit identified: ___
- [ ] Mitigation for that exploit: ___
```

### As a success-criteria template:

```markdown
## Success Criteria

A successful computation is NOT one that confirms the hypothesis.
A successful computation IS one that honestly answers:
  1. Does the mechanism work? (with specific numerical criteria)
  2. If not, how does it fail? (with the exact discrepancy)
  3. What did we learn? (even from a negative result)
```

## Integration with Other Skills

- **Result Classification** provides the evidence grades that Pass 1 checks against
- **Session Bootstrap** ensures the quarantine list is loaded for Pass 4
- **Computation Before Claim** is the runtime enforcement of what Prompt Hardening sets up at design time
