# Agent Launch Confirmation

Ambiguous phrasing is a question, not a command. Confirm before spawning autonomous agents.

## The Problem

Launching an autonomous agent is an irreversible, potentially expensive action. Once a Codex agent, a cloud job, or a parallel Claude session is running, it will consume compute, generate output, modify files, and interact with external systems — all before you can stop it.

The instinct to act quickly on a user's apparent intent is a virtue in most contexts. In agent-launch contexts, it's a liability. The cost of asking for confirmation is a few seconds. The cost of launching the wrong agent with the wrong parameters on the wrong task can be hours of compute and a confusing result set that takes longer to untangle than the original task.

## The Failure Mode

A user message like "should we send this to Codex?" contains an implicit question mark. The phrasing is asking whether to do something, not instructing the agent to do it. An agent that interprets rhetorical or exploratory questions as commands will launch agents when the user was:

- Thinking aloud about options
- Asking for the agent's opinion on whether to proceed
- Checking whether the agent understands the context before committing
- Just venting ("maybe I should just have Codex redo this")

## Classification: Questions vs Commands

### Clear commands (act immediately)

```
"Send this to Codex."
"Launch the computation on AWS."
"Start a parallel session for the data pipeline."
"Run this in the background."
```

### Questions requiring confirmation

```
"Should we send this to Codex?"
"Would it make sense to run this on the cluster?"
"Maybe Codex could handle this?"
"Do you think this needs a parallel run?"
"Could we offload this?"
```

### Ambiguous (confirm before acting)

```
"This might be a good Codex task."
"Feels like a cloud job."
"I wonder if we should parallelize this."
"This is probably worth a separate session."
```

## The Protocol

When a message could be interpreted as either a question about whether to launch or a command to launch:

### Step 1: Treat it as a question

Default interpretation: the user is asking. This is the safer prior. Launching an unintended agent is worse than asking one unnecessary confirmation question.

### Step 2: Confirm with the key parameters

Before launching, confirm:
- **Which agent**: Codex, parallel Claude, cloud job — which specifically
- **What task**: what will it be given
- **What scope**: what files, what repos, what environment variables
- **What success looks like**: how will you know it's done correctly

```
"Ready to send this to Codex. It would get:
  - Task: [specific task]
  - Files: [specific files]
  - Constraints: [any]
Should I go ahead?"
```

### Step 3: Act on explicit confirmation

Only after the user responds with a clear "yes," "go ahead," "do it," or equivalent.

## When to Skip Confirmation

Confirmation is not needed when:

1. **The command is unambiguous** — explicit imperative form with no question markers
2. **The context is established** — the user has already confirmed a pattern ("whenever I say 'Codex task' just send it") and this session is following that pattern
3. **The action is trivially reversible** — the "agent launch" is something like a local dry run that can be immediately stopped

## Compounding Effect

Each unnecessary agent launch compounds the problem. If Codex receives a task that wasn't intended, it will:
- Make a plan based on the unintended task
- Ask clarifying questions (consuming another round-trip)
- Potentially modify files
- Produce output that now needs to be reconciled with the actual intended work

One unconfirmed launch can add an hour of overhead. In a daily workflow running multiple agents in parallel, the overhead accumulates.

## Integration with the Framework

- This pattern applies to any autonomous delegation: Codex, parallel Claude sessions, background agents, cloud jobs, or any long-running process that will operate without real-time oversight
- The rule is simple: when in doubt, ask first. The user can always say "yes" in two words; recovering from a wrongly launched agent can take hours
