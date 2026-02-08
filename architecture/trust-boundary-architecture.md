# Trust Boundary Architecture: Technical Specification

## Overview

This document provides a detailed technical specification for the four-layer trust boundary architecture proposed for AI agent systems.

---

## Layer 1: Cryptographic Authentication (HMAC)

### Purpose
Ensure that every instruction processed by the AI agent provably originated from the user's trusted device.

### Implementation

```
Algorithm: HMAC-SHA256
Key length: 256-bit minimum
Output: 64 hex characters (full) or 16 characters (truncated for display)
```

### Flow

```
User Device                          Agent Gateway
    │                                      │
    ├─ message = "Check my email"          │
    ├─ timestamp = 1707350400              │
    ├─ nonce = random(16)                  │
    ├─ payload = message + timestamp       │
    │         + nonce                      │
    ├─ sig = HMAC-SHA256(key, payload)     │
    │                                      │
    ├──── {message, timestamp, nonce, ─────►
    │      sig} ──────────────────────     │
    │                                      ├─ Verify timestamp (±5min window)
    │                                      ├─ Check nonce not reused
    │                                      ├─ Recompute HMAC
    │                                      ├─ Compare signatures
    │                                      │
    │                                      ├─ ✅ Match → forward to Layer 2
    │                                      └─ ❌ Mismatch → reject, log attempt
```

### Security Properties
- **Replay protection**: Timestamp window + nonce tracking
- **Forgery resistance**: Requires knowledge of shared secret
- **Tamper detection**: Any modification invalidates signature
- **Low overhead**: ~0.1ms computation, 16-64 bytes per message

---

## Layer 2: Intent Parsing + Permission Scoping

### Purpose
Map authenticated user instructions to the minimum set of permissions required for execution.

### Permission Categories

```
PERMISSION_READ_WEB      - Browse web pages (read-only)
PERMISSION_READ_EMAIL    - Read email messages
PERMISSION_WRITE_EMAIL   - Send/reply to emails
PERMISSION_READ_FS       - Read local files
PERMISSION_WRITE_FS      - Create/modify local files
PERMISSION_READ_CALENDAR - View calendar entries
PERMISSION_WRITE_CALENDAR- Create/modify calendar entries
PERMISSION_EXECUTE       - Run system commands
PERMISSION_NETWORK_OUT   - Make outbound network requests
PERMISSION_PAYMENT       - Financial transactions
```

### Intent → Permission Mapping (Examples)

```yaml
intent: "Check weather in Tokyo"
permissions:
  - PERMISSION_READ_WEB
denied:
  - PERMISSION_READ_EMAIL
  - PERMISSION_WRITE_FS
  - PERMISSION_EXECUTE
  - PERMISSION_PAYMENT

intent: "Summarize my unread emails"
permissions:
  - PERMISSION_READ_EMAIL
denied:
  - PERMISSION_WRITE_EMAIL    # read ≠ reply
  - PERMISSION_READ_FS
  - PERMISSION_EXECUTE

intent: "Reply to John's email about the meeting"
permissions:
  - PERMISSION_READ_EMAIL
  - PERMISSION_WRITE_EMAIL    # explicitly needed
requires_confirmation:
  - PERMISSION_WRITE_EMAIL    # confirm before sending
denied:
  - PERMISSION_READ_FS
  - PERMISSION_EXECUTE
```

### Escalation Protocol

When the AI determines it needs additional permissions during execution:

```
1. AI requests permission escalation
2. Gateway pauses execution
3. User is notified with:
   - Original intent
   - Currently granted permissions
   - Requested additional permission
   - Reason for escalation
4. User approves/denies (must be HMAC-signed)
5. Execution resumes or terminates
```

---

## Layer 3: Sandboxed Execution

### Purpose
Enforce permission boundaries at the system level, making it physically impossible for the AI to exceed its granted permissions.

### Implementation Options

```
Option A: Container-based (Docker/Podman)
├─ Spin up isolated container per task
├─ Mount only necessary resources
├─ Network policies restrict outbound connections
├─ File system is read-only except for designated output dirs
└─ Container destroyed after task completion

Option B: Capability-based (Linux capabilities / macOS sandbox)
├─ Drop all capabilities by default
├─ Grant only those matching Layer 2 permissions
├─ Use seccomp profiles to restrict system calls
└─ AppArmor/SELinux policies for fine-grained control

Option C: Process-level isolation
├─ Separate process per task
├─ IPC only through controlled gateway
├─ Resource limits (CPU, memory, network)
└─ Monitoring and kill switch
```

### Permission → Sandbox Mapping

```
PERMISSION_READ_WEB →
  Container with:
  ├─ Outbound HTTPS only
  ├─ No file system access
  ├─ No email service access
  └─ Read-only DNS resolution

PERMISSION_READ_EMAIL →
  Container with:
  ├─ IMAP/API access to email service
  ├─ No outbound network (except email server)
  ├─ No file system write
  └─ Response only through gateway

PERMISSION_WRITE_EMAIL →
  Container with:
  ├─ SMTP/API access to email service
  ├─ Outbound only to user's email server
  ├─ Message held in buffer until user confirms
  └─ No file system access
```

---

## Layer 4: Result Audit

### Purpose
Maintain a complete, tamper-evident log of all AI agent actions for user review and forensic analysis.

### Log Format

```json
{
  "task_id": "uuid-v4",
  "timestamp": "2026-02-07T14:30:00Z",
  "user_intent": "Check weather in Tokyo",
  "intent_signature": "a3f8b2c1...",
  "permissions_granted": ["PERMISSION_READ_WEB"],
  "permissions_denied": ["PERMISSION_READ_EMAIL", "..."],
  "actions": [
    {
      "action_id": 1,
      "type": "web_navigate",
      "target": "https://weather.example.com",
      "timestamp": "2026-02-07T14:30:01Z",
      "result": "success"
    },
    {
      "action_id": 2,
      "type": "web_read",
      "target": "search results for 'Tokyo weather'",
      "timestamp": "2026-02-07T14:30:02Z",
      "result": "success",
      "data_extracted": true
    }
  ],
  "injection_attempts_detected": [],
  "result_summary": "Weather in Tokyo: 12°C, partly cloudy",
  "task_status": "completed",
  "escalations": []
}
```

### Audit Capabilities
- **Real-time monitoring**: User can watch actions as they happen
- **Post-task review**: Complete action history available
- **Anomaly detection**: Flag unusual patterns (e.g., unexpected network requests)
- **Injection logging**: Record any detected prompt injection attempts
- **Rollback support**: Track state changes for potential reversal

---

## Integration Architecture

```
┌──────────────┐     ┌──────────────────────────────────────────┐
│  User Device │     │            Agent Gateway                  │
│              │     │                                          │
│  ┌────────┐  │     │  ┌──────────┐  ┌──────────┐             │
│  │  App   │──┼─────┼─►│  Auth    │─►│  Intent  │             │
│  │(signed)│  │     │  │  Layer 1 │  │  Parser  │             │
│  └────────┘  │     │  └──────────┘  │  Layer 2 │             │
│              │     │                └────┬─────┘             │
│              │     │                     │                    │
│              │     │              ┌──────▼───────┐           │
│              │     │              │  Permission  │           │
│              │     │              │   Scoper     │           │
│              │     │              └──────┬───────┘           │
│              │     │                     │                    │
│              │     │  ┌──────────────────▼─────────────────┐ │
│              │     │  │        Sandbox Orchestrator         │ │
│              │     │  │              Layer 3                │ │
│              │     │  │  ┌─────────┐  ┌──────────────────┐ │ │
│              │     │  │  │   AI    │  │  Tool Execution  │ │ │
│              │     │  │  │  Model  │  │  (browser, etc.) │ │ │
│              │     │  │  └─────────┘  └──────────────────┘ │ │
│              │     │  └────────────────────┬───────────────┘ │
│              │     │                       │                  │
│  ┌────────┐  │     │  ┌───────────────────▼────────────────┐ │
│  │ Review │◄─┼─────┼──│         Audit Log (Layer 4)        │ │
│  │  Feed  │  │     │  └────────────────────────────────────┘ │
│  └────────┘  │     │                                          │
└──────────────┘     └──────────────────────────────────────────┘
```

---

## Open Questions

1. **Permission granularity standard**: Who defines the canonical set of permissions? An industry body? De facto standard from a major player?

2. **Intent ambiguity**: How should the system handle ambiguous intents that could map to multiple permission sets?

3. **Performance overhead**: What's the acceptable latency for the full auth + parse + sandbox pipeline?

4. **Cross-agent trust**: If multiple AI agents collaborate, how do they authenticate each other's actions?

5. **Offline operation**: How does HMAC-based auth work when the user's device is temporarily disconnected?

---

*This specification is a starting point for discussion. Contributions and critiques welcome.*
