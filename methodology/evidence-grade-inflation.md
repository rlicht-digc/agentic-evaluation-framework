# Evidence-Grade Inflation

The most dangerous failure mode in extended agentic reasoning, and how to prevent it.

## What It Is

Evidence-grade inflation is the gradual, unintentional upgrading of a result's evidence quality through repeated citation, without any new evidence being added.

It is the agentic equivalent of the "telephone game" -- each time a result is referenced, it gets slightly more certain, slightly more established, slightly more load-bearing. After enough references, a speculative numerical coincidence becomes a foundational result that the entire project depends on.

## Why It Happens

### 1. Context Window Frequency Bias

The more often a result appears in the agent's context window, the more "established" it feels. This is a fundamental property of language models: frequency correlates with reliability in training data. A fact mentioned 50 times across a session feels more reliable than one mentioned once, even if all 50 mentions trace back to a single unverified source.

### 2. Citation as Implicit Endorsement

Every time a computation references a prior result, it implicitly endorses it. The agent doesn't distinguish between "I'm using this value because it was proven" and "I'm using this value because the previous computation used it." Both look the same in the dependency graph.

### 3. Caveat Truncation

The original result might say: "Parameter X = 1.23, but this is a numerical coincidence with no theoretical support (grade: SUGGESTIVE)." By the tenth citation, the caveats have been dropped: "Using X = 1.23 (from Computation 12)." The grade SUGGESTIVE has been silently upgraded to ESTABLISHED.

### 4. Confirmation Bias in Multi-Step Reasoning

When a result is used as input to a new computation that produces a reasonable output, the agent interprets the reasonable output as evidence for the input. This is circular reasoning: "X must be right because the computation that used X gave a sensible answer."

## The Drift Pattern

Here is the typical lifecycle of an inflated result:

```
Computation 12:  "We find X = 1.23. This is an interesting numerical
                  coincidence (SUGGESTIVE)."

Computation 18:  "Using X = 1.23 (Computation 12), we derive Y = 4.56."
                  [implicit upgrade: X is now an input, not a coincidence]

Computation 25:  "The X-Y relationship (Computations 12, 18) is consistent
                  with framework Z."
                  [implicit upgrade: X is now part of a framework]

Computation 35:  "Based on the established X-Y-Z framework..."
                  [explicit upgrade: SUGGESTIVE -> ESTABLISHED, no new evidence]

Computation 42:  "Since X = 1.23 is well-established, we can use it to
                  predict W..."
                  [X is now load-bearing infrastructure]
```

At no point in this chain was X ever validated. No new measurement confirmed it. No independent derivation produced it. It was cited 30 times, and that repetition was treated as evidence.

## A Real Example (Anonymized)

In our research program, a certain mathematical identification was discovered early on. Call it "the matching": a certain function in the analysis could be rewritten to resemble a well-known function from statistical mechanics.

The matching was correctly classified as SUGGESTIVE when first found. Over the next 30 computations, it became central to the program's theoretical interpretation. Dozens of computations were framed as "testing implications of the matching" or "extending the matching to new domains."

A full project audit, conducted after 82 computations, found:

1. **The matching had never been upgraded past SUGGESTIVE** -- no new evidence had been added since the initial discovery
2. **An overlooked factor of ~3 made the matching quantitatively wrong** -- the two quantities were off by a constant that had been silently dropped
3. **The entire theoretical interpretation was based on this unvalidated matching** -- it was the foundation, not a detail
4. **All 25+ computations testing "implications of the matching" were investigating implications of something that wasn't true**

The audit classified this as a "CODE BLACK" -- the empirical results were all fine, but the theoretical interpretation built on top of them was unfounded. Three questions that had been framed as consequences of the matching had to be reframed as independent open problems.

**Compute wasted**: Approximately 30-40 computation-sessions (each 2-8 hours of agent time) explored implications of a result that was never properly validated.

## How to Prevent It

### 1. Explicit Grade Tracking

Use the [Result Classification](../skills/result-classification/SKILL.md) skill to assign and track evidence grades for every result. The grade is a first-class attribute, not a comment that can be dropped.

### 2. Grade Upgrade Protocol

A result's grade can ONLY be upgraded if:
- New, independent evidence supports it (not just more citations of the same evidence)
- The upgrade is explicit: "Upgrading X from SUGGESTIVE to PARTIAL because Computation 45 provided an independent derivation conditional on assumption A"
- The upgrade is recorded in the project memory

### 3. Periodic Audits

Every 20-30 computations, conduct a "grade audit":
- For each result currently classified as ESTABLISHED or higher, verify the evidence independently
- For each result classified as SUGGESTIVE that is being used as an input, flag the dependency
- For each quarantined result, verify it's not being referenced

### 4. Provenance Tagging

Every citation of a prior result must include its current grade:

```
Using X = 1.23 (Computation 12, grade: SUGGESTIVE)
```

Not:

```
Using X = 1.23 (Computation 12)
```

The grade tag makes it visible when a SUGGESTIVE result is being used as if it were ESTABLISHED.

### 5. Session Bootstrap

Use the [Session Bootstrap](../skills/session-bootstrap/SKILL.md) skill to load the current grade assignments at the start of every session. This prevents the agent from inheriting inflated grades from stale context.

## The Fundamental Insight

Evidence-grade inflation is not a bug in the agent's behavior -- it's an emergent property of extended context windows. It happens because:

1. Language models use frequency as a proxy for reliability (reasonable in training data, dangerous in extended sessions)
2. Multi-step reasoning creates dependency chains that obscure provenance
3. There is no built-in mechanism to distinguish "cited 50 times from one source" from "independently verified 50 times"

The skills in this framework address this by making evidence grades explicit, enforcing provenance tracking, and requiring periodic independent audits. The goal is not to prevent the agent from using uncertain results -- it's to ensure the uncertainty is visible and tracked throughout the session.
