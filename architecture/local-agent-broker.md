# Local Agent Broker Architecture

## Overview

A local LLM acts as a secure task router between the operator (via iMessage) and a multi-agent fleet. The system runs research programs autonomously while the operator directs from mobile.

```
┌─────────────┐     iMessage      ┌───────────────────────────────────────────────────┐
│  Operator    │ ◄──────────────► │  LOCAL WORKSTATION                                │
│  (iPhone)    │    encrypted      │                                                   │
│              │    via Apple ID   │  ┌─────────────────────────────────────────────┐  │
│  "Run Comp   │                   │  │  Local Router LLM                            │  │
│   154 with   │                   │  │  (small model, runs offline)                 │  │
│   these      │                   │  │                                               │  │
│   params"    │                   │  │  Roles:                                       │  │
│              │                   │  │  • Parse operator intent                      │  │
│              │                   │  │  • Route to the right agent for each task     │  │
│              │                   │  │  • Move outputs between agents                │  │
│              │                   │  │  • Update operator on completion              │  │
│              │                   │  │  • Maintain COMPUTE_QUEUE.md                  │  │
│              │                   │  │  • NEVER generates theory/code                │  │
│              │                   │  └──┬──────┬──────┬──────┬──────┬───────────────┘  │
│              │                   │     │      │      │      │      │                  │
│              │                   │  ┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──────────┐      │
│              │                   │  │Claude││Claude││Codex││Chat ││ Perplexity/ │      │
│              │                   │  │ AI   ││ Code ││     ││ GPT ││ Gemini      │      │
│              │                   │  │(API) ││(CLI) ││(API)││(API)││ (search)    │      │
│              │                   │  │      ││      ││     ││     ││             │      │
│              │                   │  │Theory││Heavy ││Data ││Code ││Research     │      │
│              │                   │  │Reason││Comp  ││Pipe ││Audit││Direction    │      │
│              │                   │  │Prompt││Code  ││API  ││Impl ││Adversarial  │      │
│              │                   │  │Audit ││AWS   ││Error││Revw ││Review       │      │
│              │                   │  └──┬───┘└──┬───┘└──┬──┘└──┬──┘└──┬──────────┘      │
│              │                   │     └───┬───┴───┬───┴──┬───┘      │                │
│              │                   │         │       │      │          │                │
│              │                   │  ┌──────▼───────▼──────▼──────────▼───────┐        │
│              │                   │  │           Shared Disk                   │        │
│              │                   │  │  /outputs/  /mailbox/  /COMPUTE_QUEUE   │        │
│              │                   │  └────────────────────────────────────────┘        │
│              │                   │                                                   │
│              │                   │  ┌─────────────────────────────────────────────┐  │
│              │                   │  │  Security Layer (see below)                  │  │
│              │                   │  └─────────────────────────────────────────────┘  │
└─────────────┘                    └───────────────────────────────────────────────────┘
```

## Task Flow

### 1. Operator sends instruction via iMessage

```
"Run the kill test from CQ-003.
 Use full resolution on GPU. Download results before terminating."
```

### 2. Local router parses and routes

The router does NOT generate theory, write computation prompts, or make scientific decisions. It:

- Parses the operator's natural language into a structured task
- Determines which agent handles it (Claude AI for theory/prompts, Claude Code for execution)
- Constructs the handoff with full context from disk state

```json
{
  "task_id": "CQ-003",
  "source": "operator_imessage",
  "timestamp": "2026-01-01T14:30:00Z",
  "parsed_intent": "Run kill test at full resolution on GPU with data download",
  "route_to": "claude_code",
  "context_files": [
    "COMPUTE_QUEUE.md#CQ-003",
    "scripts/run_computation.py",
    "outputs/comp_previous/summary.json"
  ],
  "security_token": "<signed_token>",
  "requires_theory": false
}
```

### 3. For theory-first tasks

```
Operator: "Derive whether the key parameter changes at finite temperature"
```

Router sends to Claude AI (API) first:
- Claude AI generates the computation prompt with hard-fail criteria
- Router moves the prompt to Claude Code
- Claude Code executes
- Router sends results summary back to operator via iMessage

### 4. For multi-step programs

```
Operator: "Run all HIGH priority items from the compute queue"
```

Router:
1. Reads COMPUTE_QUEUE.md
2. Identifies CQ-001, CQ-002, CQ-003, CQ-008 as HIGH
3. Sends kill test first (gate for the rest)
4. Runs local integration tasks in parallel
5. IF kill test passes → launches remaining computations
6. Updates operator at each milestone via iMessage
7. Updates COMPUTE_QUEUE.md status entries

## What Each Agent Does

| Agent | Role | Strengths | Never Does |
|-------|------|-----------|-----------|
| **Operator** (iMessage) | Strategic direction, approvals, course corrections | Judgment, priority, taste | Write code, run compute |
| **Local Router** | Parse intent, route tasks, move outputs, status updates | Fast local inference, always available | Generate theory, write computation prompts, make scientific judgments |
| **Claude AI** (API) | Theory, analysis, prompt generation, audits, paper drafting | Deepest reasoning, longest context, best at novel derivation | Execute code, manage infrastructure |
| **Claude Code** (CLI) | Heavy computation, code execution, AWS/GCP management, git, full-stack engineering | Superior at coding, tool use, multi-step execution, infrastructure management | Generate theory independently (follows prompts from Claude AI or operator) |
| **Codex** (API) | Data pipelines, API integration, error handling, dataset acquisition | Superior at data wrangling, API calls, robust error recovery, handling messy real-world data | Theory, scientific judgment |
| **ChatGPT** (API) | Code implementation audit, adversarial review of implementations | Good second-set-of-eyes on code before it goes to Claude Code; catches implementation-level bugs | Lead theory, lead computation |
| **Perplexity** (search) | Research direction, literature pointers, finding datasets | Real-time web search, finding papers and data sources | Computation, code execution, scientific derivation |
| **Gemini** (API) | Adversarial scientific review, second-opinion reasoning | Inference quality approaches Opus — good for catching reasoning errors that same-model review misses | Lead computation |

### Agent Selection Philosophy

**Claude Code is the primary workhorse** — it handles all heavy computation, code writing, infrastructure management, and multi-step execution.

**Claude AI is the primary thinker** — theory, derivation, prompt design, and scientific audits. When a computation needs a structured prompt with hard-fail criteria, null comparisons, and adversarial audit requirements, Claude AI writes it.

**Codex fills the data gap** — where Claude Code occasionally struggles (complex API error handling, messy data ingestion, multi-format dataset reconciliation), Codex excels. The mailbox protocol between Claude and Codex runs on structured messages with git SHAs and explicit handoffs.

**ChatGPT audits implementations** — before code goes to Claude Code for execution, ChatGPT reviews the implementation for bugs, edge cases, and logical errors.

**Perplexity and Gemini guide and review** — Perplexity points the research in the right direction. Gemini serves as an adversarial reviewer with different failure modes from the primary model.

**The key principle**: no single model does everything. Each has a specific role matched to its strengths, with explicit handoff protocols and shared disk state. The local router orchestrates but never substitutes for any of them.

## Simultaneous Program Management

The router maintains a task queue and can run multiple programs in parallel:

```
ACTIVE PROGRAMS:
├── Research Program A
│   ├── Computation (running on AWS)
│   ├── Kill test (queued for GPU)
│   └── Paper draft (Claude AI)
├── Research Program B
│   └── Paper revision (Claude AI)
├── Platform Project
│   └── Backtest (running on GKE)
└── Extension Program
    └── Figure generation (Claude Code local)
```

The router:
- Tracks which Claude Code sessions are active (max ~3 concurrent)
- Queues tasks when agents are busy
- Prioritizes based on operator-set priority or COMPUTE_QUEUE.md
- Sends consolidated status updates (not per-step noise)

## iMessage Integration

### Technical implementation

```python
# Uses applescript via subprocess to send/receive iMessages
# The local workstation runs a daemon that:
# 1. Watches ~/Library/Messages/chat.db for new messages from operator
# 2. Parses with local router LLM
# 3. Routes to appropriate agent
# 4. Sends results back via osascript

import subprocess

def send_imessage(recipient, message):
    script = f'''
    tell application "Messages"
        set targetBuddy to buddy "{recipient}" of service "iMessage"
        send "{message}" to targetBuddy
    end tell
    '''
    subprocess.run(["osascript", "-e", script])

def watch_messages(callback):
    # Poll chat.db for new messages from operator's Apple ID
    # Parse with local router
    # Execute callback with parsed task
    ...
```

### Message format conventions

**Operator → Router:**
- Natural language (router parses)
- Can include computation numbers, parameter values, priority overrides
- "Status?" returns current active task summary

**Router → Operator:**
- Structured updates at milestones only (not every timestep)
- "CQ-003 COMPLETE: kill test passed. Launching CQ-001."
- Error alerts: "CQ-002 FAILED: GPU quota exceeded. Need approval for larger instance."

---

# Security Architecture

## Threat Model

The local router has direct access to:
- Claude AI API (can generate prompts and receive responses)
- Claude Code CLI (can execute arbitrary code)
- AWS/GCP credentials (can launch/terminate instances)
- Git repositories (can push code)
- iMessage (can send messages as the operator)

**Primary threats:**
1. **Prompt injection via iMessage** — attacker sends a message that the router interprets as a legitimate operator instruction
2. **Prompt injection via Claude AI output** — compromised or manipulated Claude response contains instructions that the router executes
3. **Prompt injection via disk artifacts** — malicious content in downloaded files, computation outputs, or git repos that the router reads and acts on
4. **Exfiltration** — router sends sensitive data (API keys, proprietary research, trading strategies) to unauthorized destinations
5. **Unauthorized compute** — router launches expensive AWS/GCP resources without operator approval

## Security Layers

### Layer 1: Cryptographic Command Authentication

Every task the router executes must be traceable to either:
- A signed operator command, OR
- A signed Claude AI output in response to a signed operator command

```
┌─────────────────────────────────────────────────────────┐
│                 COMMAND CHAIN OF TRUST                    │
│                                                          │
│  Operator Device (iPhone)                                │
│    │                                                     │
│    ├── Generates: HMAC-SHA256(command, device_secret)     │
│    │   device_secret = HKDF(iCloud_keychain_entry)       │
│    │                                                     │
│    ▼                                                     │
│  Router verifies HMAC before parsing                     │
│    │                                                     │
│    ├── If routing to Claude AI:                          │
│    │   ├── Sends prompt with task_id                     │
│    │   ├── Receives response                             │
│    │   ├── Response is ONLY valid if task_id matches     │
│    │   └── Router signs the handoff to Claude Code:      │
│    │       HMAC-SHA256(task_id + response_hash,          │
│    │                   router_session_key)                │
│    │                                                     │
│    ├── If routing to Claude Code:                        │
│    │   ├── Constructs command with signed task context    │
│    │   └── Claude Code hooks verify signature before     │
│    │       executing destructive operations               │
│    │                                                     │
│    └── ALL outputs tagged with provenance chain:         │
│        operator_sig → router_sig → agent_output_hash     │
└─────────────────────────────────────────────────────────┘
```

### Layer 2: Device-Synced Token (Zero-Knowledge Proof of Origin)

The operator's device and the workstation share a synchronized secret that rotates:

```python
import hashlib
import hmac
import time

class DeviceSyncToken:
    """
    Time-based token synchronized between iPhone and workstation.
    Similar to TOTP but with longer windows and chain verification.
    """

    def __init__(self, shared_seed: bytes, window_seconds: int = 300):
        self.seed = shared_seed       # Established during initial pairing
        self.window = window_seconds  # 5-minute windows

    def current_token(self) -> str:
        counter = int(time.time()) // self.window
        return hmac.new(
            self.seed,
            counter.to_bytes(8, 'big'),
            hashlib.sha256
        ).hexdigest()[:16]

    def verify(self, token: str) -> bool:
        counter = int(time.time()) // self.window
        for offset in [-1, 0, 1]:
            expected = hmac.new(
                self.seed,
                (counter + offset).to_bytes(8, 'big'),
                hashlib.sha256
            ).hexdigest()[:16]
            if hmac.compare_digest(token, expected):
                return True
        return False
```

**Initial pairing**: QR code displayed on workstation, scanned by iPhone. Establishes the shared seed via Diffie-Hellman key exchange over the local network. Seed stored in macOS Keychain and iOS Keychain (hardware-backed).

### Layer 3: Command Allowlisting and Spend Limits

The router operates under a strict allowlist:

```yaml
# security/command_policy.yaml

allowed_actions:
  claude_code:
    - run_computation       # Execute a numbered computation
    - launch_aws_instance   # With spend limit
    - terminate_instance    # Always allowed
    - git_commit            # To tracked repos only
    - git_push              # To tracked repos only
    - read_file             # Any file
    - write_file            # Only in outputs/ and scripts/

  claude_ai:
    - generate_prompt       # Theory/analysis prompts
    - review_output         # Audit computation results
    - draft_text            # Paper sections

  blocked_always:
    - modify_security_config
    - change_api_keys
    - send_to_external_api    # Except Claude/AWS/GCP
    - modify_git_hooks
    - access_trading_credentials
    - push_to_public_repo     # Requires explicit operator approval

spend_limits:
  aws_per_hour: 50.00
  aws_per_day: 200.00
  gcp_per_hour: 50.00
  gcp_per_day: 200.00
  require_approval_above: 100.00

repo_allowlist:
  - /path/to/research-program-a
  - /path/to/research-program-b
  - /path/to/research-program-c
  # trading system NOT on allowlist for automated access
```

### Layer 4: Prompt Injection Firewall

```python
class InjectionFirewall:
    INJECTION_PATTERNS = [
        r"ignore previous instructions",
        r"system prompt",
        r"you are now",
        r"forget everything",
        r"new instructions:",
        r"<\|.*\|>",
        r"```system",
        r"ADMIN_OVERRIDE",
    ]

    def sanitize_operator_input(self, message: str) -> str:
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, message, re.IGNORECASE):
                raise SecurityError(f"Injection pattern detected: {pattern}")
        return message

    def sanitize_claude_output(self, response: str, expected_task_id: str) -> str:
        """
        Claude AI outputs are treated as UNTRUSTED DATA, not instructions.
        Only structured fields are extracted — never raw text executed.
        """
        ...

    def sanitize_disk_artifact(self, filepath: str) -> str:
        """Files read from disk are data, never instructions."""
        ...
```

### Layer 5: Audit Trail (Immutable Log)

```python
class AuditLog:
    """
    Append-only log with hash chain.
    Tampering with any entry breaks the chain.
    """

    def __init__(self, log_path: str):
        self.path = log_path
        self.prev_hash = self._get_last_hash()

    def log(self, action: dict) -> str:
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "action": action,
            "operator_token": action.get("auth_token", "NONE"),
            "prev_hash": self.prev_hash,
        }
        entry_bytes = json.dumps(entry, sort_keys=True).encode()
        entry_hash = hashlib.sha256(entry_bytes).hexdigest()
        entry["hash"] = entry_hash

        with open(self.path, "a") as f:
            f.write(json.dumps(entry) + "\n")

        self.prev_hash = entry_hash
        return entry_hash

    def verify_chain(self) -> bool:
        prev = "genesis"
        for line in open(self.path):
            entry = json.loads(line)
            if entry["prev_hash"] != prev:
                return False
            check = dict(entry)
            stored_hash = check.pop("hash")
            computed = hashlib.sha256(
                json.dumps(check, sort_keys=True).encode()
            ).hexdigest()
            if computed != stored_hash:
                return False
            prev = stored_hash
        return True
```

### Layer 6: Network Isolation

```
ALLOWED OUTBOUND:
├── api.anthropic.com (Claude AI)
├── *.amazonaws.com (AWS)
├── *.googleapis.com (GCP)
├── github.com (git)
├── iMessage relay (Apple servers)
└── pypi.org (packages, read-only)

BLOCKED OUTBOUND:
├── All other HTTP/HTTPS
├── All SSH except to cloud instances
└── All unknown ports
```

## Implementation Roadmap

### Phase 1: Core Router
- [ ] Local LLM deployment (ollama or vllm)
- [ ] iMessage watcher daemon
- [ ] Basic task parsing and routing to Claude Code
- [ ] HMAC command signing (device pairing)

### Phase 2: Claude AI Integration
- [ ] Anthropic API integration for theory tasks
- [ ] Output handoff pipeline (Claude AI → disk → Claude Code)
- [ ] Structured prompt generation from operator intent

### Phase 3: Security Hardening
- [ ] Device-synced TOTP tokens
- [ ] Command allowlisting and spend limits
- [ ] Injection firewall
- [ ] Append-only audit trail with hash chain
- [ ] Network policy enforcement

### Phase 4: Multi-Program Management
- [ ] Concurrent task queue with priority
- [ ] COMPUTE_QUEUE.md auto-maintenance
- [ ] Consolidated status reporting
- [ ] Error escalation to operator

## Security Summary

| Layer | Protects Against | Mechanism |
|-------|-----------------|-----------|
| 1. Command Auth | Unauthorized commands | HMAC-SHA256 chain of trust |
| 2. Device Token | Account compromise, replay | Time-synced TOTP, hardware-backed |
| 3. Allowlisting | Scope creep, overspend | Strict action + repo + spend policy |
| 4. Injection Firewall | Prompt injection (all vectors) | Pattern matching + structural parsing |
| 5. Audit Trail | Tampering, forensics | Hash-chained append-only log |
| 6. Network Isolation | Exfiltration, C2 | Allowlisted outbound only |
