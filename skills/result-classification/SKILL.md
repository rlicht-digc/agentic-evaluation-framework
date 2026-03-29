# Result Classification

Forces honest, explicit classification of every finding before it can be cited downstream.

## The Problem It Solves

In extended agentic sessions, the evidence grade of a result drifts upward over time without any new evidence being added. A computation produces a "suggestive" numerical match. Ten computations later, it's being treated as "established." Twenty computations later, it's load-bearing infrastructure that other results depend on.

This happens because:
- The agent's context window fills with references to the result, making it *feel* established
- Each new computation that cites the result adds another implicit endorsement
- The original caveats ("this is a coincidence until proven otherwise") get truncated or forgotten
- Nobody goes back to check whether the evidence grade was ever actually upgraded

The result: a house of cards where downstream work depends on upstream findings that were never properly validated.

## The Classification Taxonomy

Every result must be assigned exactly one of these grades:

| Grade | Meaning | Criteria |
|-------|---------|----------|
| **PROVEN** | Mathematically demonstrated | Algebraic identity, formal proof, or direct measurement with controlled systematics |
| **ESTABLISHED** | Strong empirical support | Multiple independent measurements or derivations agree; no known contradictions |
| **PARTIAL** | Conditionally derived | Result follows from a calculation, but conditional on one or more unproven assumptions |
| **SUGGESTIVE** | Interesting but unvalidated | Numerical coincidence, single measurement, or model-dependent extraction |
| **OPEN** | Under investigation | Active computation in progress; result not yet available |
| **FAILED** | Route attempted, did not work | Computation completed, mechanism does not produce the claimed result |
| **QUARANTINED** | Known to be wrong or unreliable | Previously used value that has been invalidated; must not be cited without explicit justification |

### Critical Distinctions

**PARTIAL vs SUGGESTIVE**: A PARTIAL result comes from a genuine calculation that produces the right answer *conditional on* an assumption. A SUGGESTIVE result is a numerical match without a supporting calculation. This distinction matters enormously -- PARTIAL results are one proof step from PROVEN, while SUGGESTIVE results might be coincidences.

**FAILED vs QUARANTINED**: A FAILED result is a route that was tried and didn't work. A QUARANTINED result is a value that was *previously accepted* but has since been invalidated. QUARANTINED is more dangerous because it may still be cached in the agent's context or in downstream files.

## When It Triggers

1. **End of every computation**: Before writing the verdict, classify every derived result
2. **When citing a prior result**: Check its current grade before using it
3. **When a result changes grade**: Propagate the change to all downstream dependents
4. **During audits**: Re-verify that grades match the actual evidence

## What It Does

### At computation completion:

```
For each result R produced by this computation:
  1. State R explicitly (the claim, the number, the relationship)
  2. Assign a grade from the taxonomy
  3. List the evidence supporting that grade
  4. List what would be needed to upgrade the grade
  5. List what would downgrade or kill it
```

### When citing a prior result:

```
Before using result R from Computation N:
  1. Look up R's current grade in the project memory
  2. If QUARANTINED: STOP. Do not use without explicit user authorization.
  3. If FAILED: STOP. This route is dead.
  4. If SUGGESTIVE: Flag in the current computation's assumptions list.
  5. If PARTIAL: Note the unproven assumption that R depends on.
  6. If ESTABLISHED or PROVEN: Proceed, but cite the source computation.
```

## Example: Catching Grade Inflation

**The setup**: Computation 12 found that a certain parameter matched a theoretical prediction to 1%. It was correctly classified as SUGGESTIVE (numerical coincidence, no derivation).

**The drift**: Over the next 40 computations, the result was cited repeatedly. By Computation 50, the agent was treating it as ESTABLISHED -- not because any new evidence had appeared, but because it had been referenced so many times.

**The catch**: A full-project audit (re-examining every result classification) found that the grade had never been legitimately upgraded. The parameter was reclassified back to SUGGESTIVE. Three downstream computations that depended on it being ESTABLISHED had to be re-evaluated.

**The deeper problem found by the audit**: The parameter wasn't just SUGGESTIVE -- it was actually WRONG. A factor of ~3 had been overlooked in the original identification. The correct value didn't match the prediction at all. Without the audit forcing re-examination, this error would have propagated indefinitely.

## How to Use It

### In CLAUDE.md (project-level instructions):

```markdown
## Result Classification (Mandatory)

Every computation verdict MUST classify each result using this taxonomy:
PROVEN | ESTABLISHED | PARTIAL | SUGGESTIVE | OPEN | FAILED | QUARANTINED

Before citing any prior result, check its current grade. If QUARANTINED,
do not use. If SUGGESTIVE, list it in assumptions.
```

### In computation templates:

```markdown
## Derived Results
- D1: [result statement] -- Grade: [GRADE] -- Evidence: [why this grade]
- D2: ...

## Borrowed Results
- B1: [result from Computation N] -- Current grade: [GRADE] -- Used as: [how it's used here]
```

### As a quarantine list (in CLAUDE.md):

```markdown
## Quarantine List

| Value | Status | Reason |
|-------|--------|--------|
| parameter_X | QUARANTINED | Invalidated by Computation 68 |
| solver_Y outputs | QUARANTINED | Bug found in solver |
| model_Z | QUARANTINED | Based on now-disproven assumption |
```

## Integration with Other Skills

- **Session Bootstrap** loads the quarantine list at session start, so the agent never accidentally uses a quarantined value
- **Prompt Hardening** checks that computation prompts don't assume results at a higher grade than warranted
- **Computation Before Claim** prevents the agent from stating a result without the computation that would justify its grade
