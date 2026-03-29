# Adversarial Audit Example

How a zero-assumption re-derivation audit catches errors that confirmatory analysis misses.

## What Is an Adversarial Audit?

An adversarial audit is a computation where the agent is instructed to:

1. **Trust nothing** -- Do not assume any prior result is correct
2. **Re-derive from scratch** -- Start from first principles and re-derive key results
3. **Classify every claim** -- Assign an evidence grade to every assertion
4. **Find the weakest link** -- Identify what would break if any single assumption failed

This is the opposite of a normal computation, which builds *on top of* prior results. The audit tears everything down and checks the foundation.

## When to Run an Audit

- After every 20-30 computations (periodic health check)
- When a surprising result appears (could be caused by an upstream error)
- When a result seems "too good to be true" (often is)
- Before publishing or sharing any result externally
- When you realize you can't trace a key number back to its source

## Audit Structure

```markdown
# Adversarial Audit [N]

## Scope
[Which computations and results are being audited]

## Method
1. For each result in scope:
   a. State the result explicitly
   b. Trace it to its original derivation
   c. Re-derive it independently (no peeking at the original)
   d. Compare: do the independent derivation and original agree?
   e. Classify: PROVEN | ESTABLISHED | PARTIAL | SUGGESTIVE | FAILED | QUARANTINED

2. For each dependency chain:
   a. List every link: R1 -> R2 -> R3 -> ... -> R_final
   b. Check: does every link use its input at the correct grade?
   c. Check: has any link been updated since the chain was built?
   d. Identify: what is the weakest link? What grade does it have?

3. For each assumption:
   a. Is it stated explicitly, or is it implicit?
   b. Has it been tested, or just assumed?
   c. What would break if it were wrong?

## Rules
- DO NOT look at the original computation's output while re-deriving
- DO NOT assume any intermediate result is correct
- DO flag every place where the re-derivation disagrees with the original
- DO report the honest verdict, even if it means major results are invalidated
```

## A Real Audit (Anonymized)

### Setup

After 82 computations, a full project audit was conducted. The scope: every theoretical claim in the project.

### What the Audit Found

**Category 1: Results that SURVIVED**
- The core empirical measurements (4 published analyses) -- CLEAN
- 25 theoretical impossibility results ("mechanism X cannot work because Y") -- CLEAN
- All data analysis pipelines -- CLEAN

**Category 2: Results that were RECLASSIFIED**
- A central theoretical identification (algebraic mapping between two functions) was classified as ESTABLISHED by the project. The audit found it should be SUGGESTIVE -- it was a mathematical rewriting with no physical content, and a factor of ~3 had been silently dropped.
- A "derivation" of a key parameter turned out to be an identification, not a derivation. It correctly matched a number but didn't explain *why* that number appeared. Reclassified from PARTIAL to SUGGESTIVE.

**Category 3: Results that were RETRACTED**
- A claimed connection between two domains was found to be tautological -- it followed from the definitions, not from physics. RETRACTED.
- A reported match between computed and observed values had a unit conversion error. The corrected values didn't match. RETRACTED.

**Category 4: Structural problems found**
- The entire theoretical interpretation of the project was built on an unvalidated assumption (the identification from Category 2). Since the identification was SUGGESTIVE not ESTABLISHED, everything downstream was resting on an unvalidated foundation.
- The project had been asking the wrong question for 30+ computations. The question "why does function F appear?" was based on the assumption that F was fundamental. The audit showed F was a convenient rewriting of a simpler expression, and the real question was about a different parameter.

### The Verdict

The audit classified the project state as what the team called "CODE BLACK":
- **Empirical core: INTACT** -- all measurements and data analyses were clean
- **Theoretical interpretation: NOT INTACT** -- built on an unvalidated identification
- **25+ theoretical kills: CLEAN** -- the negative results (mechanisms that don't work) were all valid
- **3 clean restart questions identified** -- the right questions to ask going forward, without the incorrect framing

### Compute Impact

Approximately 30-40 computation-sessions were partially or fully invalidated by the audit. However, the audit also:
- Confirmed that all negative results were valid (saving future re-investigation)
- Identified three clean, well-posed questions to replace the poorly-posed original framing
- Prevented the project from publishing an interpretation that was built on sand

## Key Lessons

### 1. Negative results are more robust than positive results

The audit found that all 25 "kill" results (proofs that a mechanism doesn't work) survived scrutiny. The failures were all in the positive results (claims that something *does* work). This makes sense: a kill result says "this specific computation shows a 98-order-of-magnitude gap," which is easy to verify. A positive result says "these two things match," which requires checking all the steps.

### 2. The most dangerous result is the "obvious" one

The identification that anchored the entire theoretical program was treated as obvious -- "clearly F is just a rewriting of G." Nobody questioned it for 80 computations because it seemed self-evident. The audit's zero-assumption stance forced a re-examination, which found the overlooked factor of 3.

### 3. Audits should be run by the same agent with different instructions

The audit was conducted by the same AI agent that did the original work, but with explicitly different instructions: "trust nothing, re-derive everything, find the weakest link." This is important because:
- The agent has access to all the same tools and context
- The different instructions change its behavior from "confirmatory" to "adversarial"
- No additional infrastructure is needed

### 4. The audit format should be standardized

Using the same audit structure every time makes it possible to compare audits, track which results have been audited and when, and ensure nothing is missed. The structure above (re-derive, classify, trace dependencies, identify weakest link) covers the essential checks.

## Integration with the Framework

- **Result Classification** provides the grades used during audit classification
- **Session Bootstrap** ensures the audit starts with the current state, not stale information
- **Prompt Hardening** is applied to the audit prompt itself (yes, you should harden the audit prompt too)
- **Computation Before Claim** prevents the audit from stating conclusions without verification
