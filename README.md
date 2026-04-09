# Agentic Evaluation Framework

Alignment guardrails for extended AI reasoning sessions.

## The Problem

When AI agents work on complex multi-step tasks over extended sessions, systematic failure modes emerge that standard prompt engineering doesn't address:

1. **Evidence-grade inflation** -- Results drift from "interesting" to "suggestive" to "established" to "proven" without new evidence. In a 50-computation session, an early exploratory finding can become a load-bearing assumption 30 computations later.

2. **Stale-value propagation** -- A corrected or quarantined result continues to be cited by downstream computations because the correction didn't propagate through the dependency graph.

3. **Framework misattribution** -- The agent defaults to published literature conventions rather than the novel framework's conventions, because training-data frequency bias makes the published version "feel more natural."

4. **Agent coordination failures** -- Multiple agents working in parallel reach inconsistent conclusions, launch unintended work, or fail to converge on quality assurance cycles.

5. **Scale-launch failures** -- Computation logic bugs discovered after committing to expensive cloud runs that a 5-minute local smoke test would have caught.

These aren't hypothetical. They were discovered through daily operation of a 420+ computation research program across multiple domains, where each failure mode caused real errors that had to be caught and corrected.

## Skills

Four core skills that run as automatic triggers or slash commands during Claude Code sessions:

### [Result Classification](skills/result-classification/SKILL.md)
Forces honest classification of every scientific finding before it gets cited downstream. 7-tier evidence taxonomy (PROVEN through QUARANTINED) with explicit criteria for each level. Prevents the gradual drift from "interesting" to "proven."

### [Prompt Hardening](skills/prompt-hardening/SKILL.md)
Automatically self-reviews, red-teams, and battle-hardens any computation prompt before execution. Catches unchecked assumptions, contaminated inputs, numerology disguised as derivation, and missing guardrails.

### [Session Bootstrap](skills/session-bootstrap/SKILL.md)
Loads critical project state from disk at session start, ensuring the agent knows what is alive, dead, and quarantined before answering any question. Prevents stale or killed results from being cited as current. For projects using a provenance database ([agent-brain-mcp](https://github.com/rlicht-digc/agent-brain-mcp)), bootstrap can query the database directly for authoritative state.

### [Computation Before Claim](skills/computation-before-claim/SKILL.md)
Enforces the rule that no quantitative claim may be stated without a supporting computation. Blocks the pattern where an agent states a number confidently from "reasoning" that was never actually computed.

## Methodology

Seven methodology documents capturing failure modes, their causes, and their solutions:

### Core failure modes
- **[Evidence-Grade Inflation](methodology/evidence-grade-inflation.md)** -- The most dangerous failure mode in extended agentic reasoning, and how to prevent it
- **[Framework Misattribution](methodology/framework-misattribution.md)** -- When training-data frequency bias overrides the user's actual framework
- **[Kill-List Methodology](methodology/kill-list.md)** -- How to systematically test and kill hypotheses with specific numerical failures, preventing wasted compute from AI overconfidence

### Agent behavior
- **[Smoke Test Before Scale](methodology/smoke-test-before-scale.md)** -- Validate computation logic locally before committing to expensive cloud runs. Real cost: ~$48 in wasted compute from three untested bugs caught too late
- **[Check State Before Contradicting](methodology/check-state-before-contradicting.md)** -- When an agent disagrees with the user's factual claim about their own project, verify the provenance record before disputing
- **[Agent Launch Confirmation](methodology/agent-launch-confirmation.md)** -- Ambiguous phrasing ("should we send this to Codex?") is a question, not a command. Confirm before spawning autonomous agents

## Examples

- **[Computation Prompt Template](examples/computation-prompt-template.md)** -- Structured prompt with hard-fail criteria, null comparison, and adversarial audit requirements
- **[Adversarial Audit Example](examples/adversarial-audit-example.md)** -- How a zero-assumption re-derivation audit catches errors that confirmatory analysis misses
- **[Multi-Agent Coordination](examples/multi-agent-coordination.md)** -- Three coordination protocols: flat mailbox, implement/review autoloop with CONVERGED/REVISE/ESCALATE verdicts, and independent cross-validation

## Architecture

- **[Local Agent Broker](architecture/local-agent-broker.md)** -- Secure task router between operator (iMessage) and multi-agent fleet; local LLM as dispatcher with cryptographic command authentication
- **[MCP Coordination Stack](architecture/mcp-coordination-stack.md)** -- Three-server MCP architecture (science provenance + agent message bus + shared brain) with implement/review autoloop cycles and drift detection

## Background

These tools were developed during an ongoing AI-assisted research program in theoretical physics ([preprint](https://osf.io/jshr9/)). The program has executed 420+ computations across multiple scientific domains, with a full adversarial math audit (70 computation directories verified with zero contamination).

The physics is the example domain. The contribution is the evaluation methodology -- which applies to any extended agentic reasoning task where evidence quality, consistency, and honest assessment matter.

## Usage with Claude Code

These skills are designed as Claude Code slash commands and automatic triggers. They live in your project's `.claude/commands/` directory or as instructions in `CLAUDE.md`. See each skill's documentation for integration instructions.

```
.claude/
  commands/
    check-quarantine.md      # Session Bootstrap trigger
    computation-template.md  # Prompt Hardening + Result Classification
  CLAUDE.md                  # Framework rules + quarantine list
```

The skills compose: a new computation triggers Prompt Hardening (pre-flight), runs with Computation Before Claim enforced, and has its outputs classified by Result Classification before they enter the project memory.

## License

MIT
