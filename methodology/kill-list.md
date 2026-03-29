# Kill-List Methodology

How to systematically test and kill hypotheses with specific numerical failures.

## The Problem

AI agents are optimistic. When given a hypothesis to test, the default behavior is to find ways to make it work. A mechanism that is off by 5 orders of magnitude gets described as "the right direction but quantitatively insufficient." A sign error gets described as "requiring further investigation." A fundamental obstruction gets described as "a challenge for future work."

This optimism wastes enormous amounts of compute. In a 400+ computation program, we found that **more than half of the total computation time was spent on mechanisms that should have been killed earlier** but weren't, because the agent kept finding ways to characterize failures as "partial successes."

## The Methodology

### Pre-Register Kill Criteria

Before running any computation, specify in advance:

```
KILL if:
  - Magnitude off by > 2 orders
  - Wrong sign
  - Wrong functional form (e.g., linear when need quadratic)
  - Depends on assumption already killed elsewhere

INCONCLUSIVE if:
  - Magnitude off by 1-2 orders (could be a missing factor)
  - Right form but with free parameters that can't be fixed

SUPPORT if:
  - Within 1 order with correct functional form
  - No free parameters adjusted
  - Independent of previously killed assumptions
```

### Record the Kill with Specifics

When a mechanism fails, record it with exact numbers, not vague descriptions:

**Good kill record:**
```
Mechanism: [name]
Verdict: KILLED
Failures:
  K1: Magnitude gap of 10^{111} (need O(1), got 10^{-111})
  K2: Wrong sign (need positive, got negative)
  K3: Wrong scaling (need sqrt(x), got x^3)
Independent kills: 3 (any one is sufficient)
```

**Bad kill record:**
```
Mechanism: [name]
Verdict: Doesn't quite work, but interesting direction
Notes: The magnitude is off but the structure is suggestive
```

The second version guarantees the mechanism will be revisited and waste more compute.

### Count Independent Kills

A mechanism killed by one failure might be rescued by a correction. A mechanism killed by 6 independent failures is dead. Track them separately:

```
Mechanism A:
  K1: [failure 1] -- FATAL (independently sufficient)
  K2: [failure 2] -- FATAL
  K3: [failure 3] -- FATAL
  K4: [failure 4] -- contributing
  K5: [failure 5] -- contributing

Total: 3 fatal + 2 contributing = DEFINITIVELY DEAD
```

The word "independent" is critical. Five failures that all trace back to the same root cause count as one kill, not five.

### Maintain a Cumulative Kill List

The kill list is a living document that grows over time:

```
## Killed Mechanisms (DO NOT RETRY)

| # | Mechanism | Kills | Fatal | Date | Computation |
|---|-----------|-------|-------|------|-------------|
| 1 | Approach A | 6 | 3 | 2026-01-15 | Comp 42 |
| 2 | Approach B | 4 | 2 | 2026-01-20 | Comp 46 |
| 3 | Approach C | 10 | 3 | 2026-02-01 | Comp 50 |
...
```

### Close Entire Classes, Not Just Instances

When enough individual mechanisms from the same class have been killed, close the entire class:

```
## Closed Program: [Class Name]

Mechanisms tested: 11
Mechanisms killed: 11
Common failure: [the structural reason this entire class fails]
Date closed: 2026-02-15
Final computation: Comp 52

DO NOT attempt new instances of this class unless the common failure
is specifically addressed.
```

This prevents the agent from proposing "a new variant of approach A" that falls into the same class and will fail for the same structural reason.

## Real Example (Anonymized)

A research program tested whether a certain microscopic mechanism could produce an observed macroscopic effect. Over 15 computations, 11 specific variants were tested:

| Variant | Verdict | Key Failure |
|---------|---------|-------------|
| Direct calculation | KILLED | 111 orders of magnitude too small |
| With amplification A | KILLED | Wrong sign |
| With amplification B | KILLED | Tautological (assumed the answer) |
| Modified framework 1 | KILLED | Same 111-order gap (different route) |
| Modified framework 2 | KILLED | Wrong functional form (cubic, need sqrt) |
| Quantum correction | KILLED | 162 orders of magnitude too small |
| Collective version | KILLED | Free parameter (not predictive) |
| Tunneling route | KILLED | Universal obstruction in counting statistics |
| Two-bath variant | KILLED | No second bath exists (4 sub-kills) |
| Spectral route | KILLED | Wrong quantity (produces effect in domain A, need domain B) |
| Algebraic route | KILLED | Uniqueness only (no derivation of form) |

After the 11th kill, the entire class was closed:
```
CLOSED: Microscopic [class name] program
Common failure: The fundamental coupling constant is suppressed
by 10^{111}, and no amplification mechanism exists that can bridge this gap.
11/11 variants tested, all killed independently.
```

**Compute saved by the close**: At least 5-10 more computations that would have tested yet more variants, each costing hours of agent time. The structural diagnosis ("suppressed coupling, no amplification") explains *why* the entire class fails, not just each individual variant.

## How to Implement

### Step 1: Add kill criteria to every computation prompt

Use the [Prompt Hardening](../skills/prompt-hardening/SKILL.md) skill to ensure every computation has pre-registered kill criteria.

### Step 2: Enforce honest verdicts

Use the [Result Classification](../skills/result-classification/SKILL.md) taxonomy. KILLED means KILLED, not "partially successful" or "interesting but quantitatively insufficient."

### Step 3: Maintain the cumulative list

Add every kill to the project memory so that [Session Bootstrap](../skills/session-bootstrap/SKILL.md) loads it at the start of every session.

### Step 4: Review for class closures

Periodically (every 10-20 computations), review the kill list for patterns. If 5+ mechanisms from the same class are all dead, close the class.

## The Counter-Intuitive Insight

**Killing mechanisms efficiently is more valuable than confirming them.** A confirmed mechanism adds one data point. A killed mechanism with a specific failure *removes an entire class of future dead ends*. The kill list is the most valuable output of a long-running research program -- it tells you where NOT to look.

In our program, the kill list (50+ entries) saved more compute than all the positive results combined, because each kill prevented multiple downstream computations that would have built on a false foundation.
