# Framework Misattribution

When training-data frequency bias overrides the user's actual framework.

## What It Is

Framework misattribution occurs when an AI agent, asked to work within a specific framework defined by the user, defaults to the conventions of a different (usually more common) framework from its training data. The agent doesn't announce the substitution -- it just uses the familiar version because it "feels more natural."

This is distinct from a factual error. The agent isn't wrong about the standard framework -- it's correct that the standard framework defines things a certain way. The problem is that the user isn't using the standard framework, and the agent doesn't notice the difference.

## Why It Happens

### Training-Data Frequency Bias

Language models learn statistical associations between concepts. If concept A is associated with convention X in 10,000 training documents and with convention Y in 3 documents, the model will default to X unless strongly prompted otherwise.

In research contexts, the user's novel framework is, by definition, not in the training data (or is vastly underrepresented compared to standard frameworks). The agent has seen the standard version thousands of times and the user's version zero times.

### Semantic Similarity Masking

The user's framework often uses the same words as the standard framework but with different definitions. The agent processes the familiar words, activates the familiar associations, and produces the familiar result -- even though the user intended a different meaning.

### Gradual Drift

Even when the agent is initially corrected, the standard associations reassert themselves over long sessions. As the context window fills, the ratio of "user's framework" text to "standard framework" associations (from the model's weights) gradually shifts back toward standard.

## How It Manifests

### Type 1: Definition Substitution

The user defines a quantity one way. The agent uses the standard definition instead.

**Example (anonymized)**: A project defined a certain coupling constant as the ratio of two scales in the novel framework. The standard literature defines the same symbol as a running coupling in a renormalization group flow. Over a 20-computation session, the agent gradually started using the RG definition -- not because anyone told it to, but because "coupling constant + that symbol" activates the RG association in the model's weights.

**How it was caught**: A session bootstrap check loaded the project's own definition and flagged the discrepancy when the agent's computation used a formula that was correct for the standard definition but wrong for the project's definition.

### Type 2: Convention Mismatch

Two frameworks use different conventions (e.g., different sign conventions, different normalization factors, different unit systems). The agent uses the more common convention.

**Example (anonymized)**: The project used a specific convention where a certain factor appeared with coefficient `c_0 / g_*`, following the project's own derivation chain. The standard literature uses a different normalization. The agent produced a computation using the literature convention without flagging the change.

**How it was caught**: When the computation produced a result that was "off by a factor of 2 from the expected value," the factor of 2 was traced to the convention difference. The result was actually correct under the literature convention -- it was the wrong convention for this project.

### Type 3: Scope Assumption

The standard framework has a known domain of validity. The user's framework has a different domain. The agent assumes the standard domain.

**Example (anonymized)**: A computation tested whether a certain relationship held at a new scale. The agent imported constraints from the standard framework that apply at a different scale, concluding "this is already ruled out." In fact, the user's framework makes different predictions at that scale, and the standard constraints don't apply.

**How it was caught**: The prompt hardening step (Pass 5, red team) asked "what assumptions are we importing from outside the framework?" and identified the scope mismatch.

## The Danger

Framework misattribution is especially dangerous because:

1. **It looks correct.** The computation is internally consistent -- just under the wrong framework.
2. **The agent is confident.** It has strong associations with the standard framework and will defend them.
3. **It's hard to catch in review.** A reviewer familiar with the standard framework will also think it looks right.
4. **It compounds.** One misattributed convention infects every downstream computation.

## How to Prevent It

### 1. Explicit Framework Documentation

In `CLAUDE.md` or equivalent project instructions, explicitly document:
- Every definition that differs from standard conventions
- Every notation that is used differently from the literature
- Every assumption that is project-specific

```markdown
## Framework Conventions (THIS PROJECT)

| Symbol | This Project | Standard Literature | Difference |
|--------|-------------|--------------------| ------------|
| g_s | c_0 / g_* (from Comp 75D) | Running coupling | Different quantity entirely |
| a_0 | c * H_inf / (2d) | Milgrom's constant | Same physical quantity, different derivation |
| eta | 2/9 (from TT projection) | Various definitions | Project-specific |
```

### 2. Session Bootstrap with Framework Loading

Use [Session Bootstrap](../skills/session-bootstrap/SKILL.md) to load the framework conventions at the start of every session. This sets the agent's context before it has a chance to default to standard associations.

### 3. Provenance Tagging for Conventions

Every time a convention or definition is used, tag its source:

```
Using g_s = c_0 / g_* [THIS PROJECT, Computation 75D]
```

Not:

```
Using g_s = [value]
```

The tag forces the agent to explicitly identify which framework it's using.

### 4. Red-Team Pass for Framework Consistency

The [Prompt Hardening](../skills/prompt-hardening/SKILL.md) red-team pass should specifically ask:

```
Is every definition, convention, and scope assumption in this computation
consistent with THIS PROJECT's framework, not the standard literature?
```

### 5. Drift Detection

In long sessions, periodically ask the agent to state the project's conventions for key quantities. If the response drifts toward standard literature conventions, correct it and reload the framework documentation.

## The Deeper Issue

Framework misattribution reveals a fundamental tension in using language models for research:

- **Training data makes the model useful** -- it knows the standard frameworks, can do calculations, and has broad knowledge
- **Training data makes the model biased** -- it will default to the most common version of any concept
- **Novel research requires the uncommon version** -- that's what makes it novel

The skills in this framework don't try to fight the training data. Instead, they make the bias visible and correctable: explicit conventions, provenance tags, periodic checks, and session bootstrap to reload the correct framework at every opportunity.
