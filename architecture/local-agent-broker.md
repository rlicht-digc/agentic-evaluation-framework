# Local Agent Broker Architecture

## Overview

A local LLM (Qwen 3.5 Opus-distilled) acts as a secure task router between the operator (via iMessage), Claude AI (theory/reasoning), and Claude Code CLI (computation/coding). The system runs research programs autonomously while the operator directs from mobile.

```
┌─────────────┐     iMessage      ┌──────────────────────────────────────────┐
│  Operator    │ ◄──────────────► │  LOCAL WORKSTATION                       │
│  (iPhone)    │    encrypted      │                                          │
│              │    via Apple ID   │  ┌────────────────────────────────────┐  │
│  "Run Comp   │                   │  │  Qwen 3.5 (Opus-distilled)        │  │
│   154 with   │                   │  │  LOCAL ROUTER LLM                  │  │
│   these      │                   │  │                                    │  │
│   params"    │                   │  │  Roles:                            │  │
│              │                   │  │  • Parse operator intent           │  │
│              │                   │  │  • Route to Claude AI or Code      │  │
│              │                   │  │  • Move outputs between agents     │  │
│              │                   │  │  • Update operator on completion   │  │
│              │                   │  │  • Maintain COMPUTE_QUEUE.md       │  │
│              │                   │  │  • NEVER generates theory/code     │  │
│              │                   │  └──────┬──────────────┬──────────────┘  │
│              │                   │         │              │                  │
│              │                   │    ┌────▼────┐   ┌────▼──────────┐      │
│              │                   │    │Claude AI│   │Claude Code CLI│      │
│              │                   │    │  (API)  │   │  (local)      │      │
│              │                   │    │         │   │               │      │
│              │                   │    │ Theory  │   │ Computation   │      │
│              │                   │    │ Analysis│   │ Code execution│      │
│              │                   │    │ Prompts │   │ AWS/GCP mgmt  │      │
│              │                   │    │ Audits  │   │ Git operations│      │
│              │                   │    └────┬────┘   └──────┬────────┘      │
│              │                   │         │               │                │
│              │                   │         └───────┬───────┘                │
│              │                   │                 │                        │
│              │                   │         ┌───────▼────────┐               │
│              │                   │         │  Shared Disk   │               │
│              │                   │         │  /outputs/     │               │
│              │                   │         │  /mailbox/     │               │
│              │                   │         │  /COMPUTE_QUEUE│               │
│              │                   │         └────────────────┘               │
│              │                   │                                          │
│              │                   │  ┌────────────────────────────────────┐  │
│              │                   │  │  Security Layer (see below)        │  │
│              │                   │  └────────────────────────────────────┘  │
└─────────────┘                    └──────────────────────────────────────────┘
```

## Task Flow

### 1. Operator sends instruction via iMessage

```
"Run the Mach 1 kill test from CQ-003.
 Use 256³ on GPU. Download results before terminating."
```

### 2. Local router (Qwen) parses and routes

The router does NOT generate theory, write computation prompts, or make scientific decisions. It:

- Parses the operator's natural language into a structured task
- Determines which agent handles it (Claude AI for theory/prompts, Claude Code for execution)
- Constructs the handoff with full context from disk state

```json
{
  "task_id": "CQ-003",
  "source": "operator_imessage",
  "timestamp": "2026-03-29T14:30:00Z",
  "parsed_intent": "Run Mach 1 soliton collision at 256³ on GPU with data download",
  "route_to": "claude_code",
  "context_files": [
    "COMPUTE_QUEUE.md#CQ-003",
    "scripts/gpe_3d_soliton_merger.py",
    "outputs/comp151_3d_gpe_results/summary.json"
  ],
  "security_token": "<signed_token>",
  "requires_theory": false
}
```

### 3. For theory-first tasks

```
Operator: "Derive whether the sextic Jeans scale changes at finite temperature"
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
3. Sends CQ-003 to Claude Code first (kill test — gate for the rest)
4. Runs CQ-008 locally in parallel (data integration, no AWS)
5. IF CQ-003 passes → launches CQ-001 and CQ-002 via Claude Code
6. Updates operator at each milestone via iMessage
7. Updates COMPUTE_QUEUE.md status entries

## What Each Agent Does

| Agent | Role | Never Does |
|-------|------|-----------|
| **Operator** (iMessage) | Strategic direction, approvals, course corrections | Write code, run compute |
| **Qwen Router** (local) | Parse intent, route tasks, move outputs, status updates | Generate theory, write computation prompts, make scientific judgments |
| **Claude AI** (API) | Theory, analysis, prompt generation, audits, paper drafting | Execute code, manage infrastructure |
| **Claude Code** (CLI) | Code execution, AWS/GCP management, git, data processing | Generate theory independently (follows prompts from Claude AI or operator) |

## Simultaneous Program Management

The router maintains a task queue and can run multiple programs in parallel:

```
ACTIVE PROGRAMS:
├── BEC Dark Matter
│   ├── Comp 154 (running on AWS)
│   ├── CQ-003 (queued for GPU)
│   └── Paper A draft (Claude AI)
├── BH Singularity
│   └── Paper 3 revision (Claude AI)
├── V3 Trading System
│   └── v7.5.0 backtest (running on GKE)
└── Cluster Paper
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
# 2. Parses with Qwen
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
    # Parse with Qwen router
    # Execute callback with parsed task
    ...
```

### Message format conventions

**Operator → Router:**
- Natural language (Qwen parses)
- Can include computation numbers, parameter values, priority overrides
- "Status?" returns current active task summary

**Router → Operator:**
- Structured updates at milestones only (not every timestep)
- "CQ-003 COMPLETE: Mach 1 coh_frac = 0.87. Galaxy preserved. Launching CQ-001."
- Error alerts: "CQ-002 FAILED: GPU quota exceeded. Need approval for p3 instance."

---

# Security Architecture

## Threat Model

The local LLM (Qwen) has direct access to:
- Claude AI API (can generate prompts and receive responses)
- Claude Code CLI (can execute arbitrary code)
- AWS/GCP credentials (can launch/terminate instances)
- Git repositories (can push code)
- iMessage (can send messages as the operator)

**Primary threats:**
1. **Prompt injection via iMessage** — attacker sends a message that Qwen interprets as a legitimate operator instruction
2. **Prompt injection via Claude AI output** — compromised or manipulated Claude response contains instructions that Qwen executes
3. **Prompt injection via disk artifacts** — malicious content in downloaded files, computation outputs, or git repos that Qwen reads and acts on
4. **Exfiltration** — Qwen sends sensitive data (API keys, trading strategies, unpublished research) to unauthorized destinations
5. **Unauthorized compute** — Qwen launches expensive AWS/GCP resources without operator approval

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
import secrets

class DeviceSyncToken:
    """
    Time-based token synchronized between iPhone and workstation.
    Similar to TOTP but with longer windows and chain verification.
    """

    def __init__(self, shared_seed: bytes, window_seconds: int = 300):
        self.seed = shared_seed       # Established during initial pairing
        self.window = window_seconds  # 5-minute windows

    def current_token(self) -> str:
        """Generate current valid token."""
        counter = int(time.time()) // self.window
        return hmac.new(
            self.seed,
            counter.to_bytes(8, 'big'),
            hashlib.sha256
        ).hexdigest()[:16]

    def verify(self, token: str) -> bool:
        """Verify a token (allows ±1 window for clock drift)."""
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

**Every iMessage command** includes the current token. Router rejects messages without valid tokens — even from the operator's Apple ID (in case of account compromise).

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
  aws_per_hour: 50.00         # USD
  aws_per_day: 200.00
  gcp_per_hour: 50.00
  gcp_per_day: 200.00
  require_approval_above: 100.00  # Single task

repo_allowlist:
  - /Users/russelllicht/bec-dark-matter
  - /Users/russelllicht/bh-singularity
  - /Users/russelllicht/empirical-ladder
  - /Users/russelllicht/morris-thorne-wormhole
  # V3 trading system NOT on allowlist for automated access
```

### Layer 4: Prompt Injection Firewall

The router applies input sanitization before parsing:

```python
class InjectionFirewall:
    """
    Prevents prompt injection from iMessage, Claude outputs, or disk files.
    """

    INJECTION_PATTERNS = [
        r"ignore previous instructions",
        r"system prompt",
        r"you are now",
        r"forget everything",
        r"new instructions:",
        r"<\|.*\|>",              # Common injection delimiters
        r"```system",
        r"ADMIN_OVERRIDE",
    ]

    def sanitize_operator_input(self, message: str) -> str:
        """Strip potential injection patterns from operator messages."""
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, message, re.IGNORECASE):
                raise SecurityError(f"Injection pattern detected: {pattern}")
        return message

    def sanitize_claude_output(self, response: str, expected_task_id: str) -> str:
        """
        Claude AI outputs are treated as UNTRUSTED DATA, not instructions.
        The router extracts structured fields only, never executes raw text.
        """
        # Parse only the expected JSON/markdown structure
        # Never eval() or exec() anything from Claude's response
        # Only extract: computation_prompt, verdict, numerical_results
        ...

    def sanitize_disk_artifact(self, filepath: str) -> str:
        """Files read from disk are data, never instructions."""
        content = open(filepath).read()
        # Strip any content that looks like router commands
        # Files can contain computation results but not router directives
        return content
```

### Layer 5: Audit Trail (Immutable Log)

Every action is logged to an append-only audit trail:

```python
class AuditLog:
    """
    Append-only log with hash chain (lightweight blockchain).
    Each entry references the hash of the previous entry.
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
        """Verify the entire audit trail hasn't been tampered with."""
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
┌─────────────────────────────────────────────┐
│  WORKSTATION NETWORK POLICY                  │
│                                              │
│  ALLOWED OUTBOUND:                           │
│  ├── api.anthropic.com (Claude AI)           │
│  ├── *.amazonaws.com (AWS EC2/S3)            │
│  ├── *.googleapis.com (GCP)                  │
│  ├── github.com (git push/pull)              │
│  ├── iMessage relay (Apple servers)           │
│  └── pypi.org (pip install, read-only)       │
│                                              │
│  BLOCKED OUTBOUND:                           │
│  ├── All other HTTP/HTTPS                    │
│  ├── All SSH except to AWS/GCP instances     │
│  ├── All SMTP (no email from workstation)    │
│  └── All unknown ports                       │
│                                              │
│  ALLOWED INBOUND:                            │
│  ├── iMessage (Apple relay only)             │
│  └── SSH from operator's known IPs only      │
│                                              │
│  BLOCKED INBOUND:                            │
│  └── Everything else                         │
└─────────────────────────────────────────────┘
```

## Implementation Roadmap

### Phase 1: Core Router (Week 1)
- [ ] Qwen 3.5 local deployment (ollama or vllm)
- [ ] iMessage watcher daemon
- [ ] Basic task parsing and routing to Claude Code
- [ ] HMAC command signing (device pairing)

### Phase 2: Claude AI Integration (Week 2)
- [ ] Anthropic API integration for theory tasks
- [ ] Output handoff pipeline (Claude AI → disk → Claude Code)
- [ ] Structured prompt generation from operator intent

### Phase 3: Security Hardening (Week 3)
- [ ] Device-synced TOTP tokens
- [ ] Command allowlisting and spend limits
- [ ] Injection firewall
- [ ] Append-only audit trail with hash chain
- [ ] Network policy enforcement (pf/iptables)

### Phase 4: Multi-Program Management (Week 4)
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

**Defense in depth**: An attacker would need to compromise the operator's device (Layer 2) AND bypass the injection firewall (Layer 4) AND stay within the allowlist (Layer 3) AND avoid detection in the audit trail (Layer 5). Each layer is independent.
