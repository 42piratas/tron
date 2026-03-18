# Skill: TRON Communications

How agents send messages, poll for commands, and report status. This skill is referenced by all agents operating under TRON orchestration.

**Location in project:** `meta/skills/skill-tg-comms.md`
**Deployed by:** `tron-seed.md` during seeding

---

## 1. Overview

All agents in a TRON-orchestrated session communicate through a **file-based message bus** at `{TRON_META_PATH}/logs/tron/bus/`. Agents write to the bus; TRON reads the bus and forwards messages to Telegram for user visibility. Messages are tagged with the agent's ID so TRON and the user can track who said what.

**Your agent ID** is set in the environment variable `TRON_AGENT_ID` (e.g., `ENG-1`, `REV-1`, `ARCH-1`). This was assigned by TRON when you were spawned. Use it in all communications.

**Architecture:** Agents never send to TG directly. The bus is the single write target. TRON handles TG forwarding.

---

## 2. Sending Messages

### 2.1 When to Send

| Event | Message Type | Required? |
|:--|:--|:--|
| Starting a block or major phase | 🚀 `STARTED` | Yes |
| Discrete achievement (PR merged, deploy complete, test passing) | ✅ `MILESTONE` | Yes |
| No milestone achieved in 5 minutes | ⏳ `HEARTBEAT` | Yes |
| About to start a long operation (>2min expected) | ⏳ `HEARTBEAT` | Yes |
| All tasks complete, ready for verification | 🏁 `DONE` | Yes |
| Stuck, need help | ⚠️ `BLOCKER` | Yes |
| Need a decision from user | ❓ `QUESTION` | Yes |
| Something broke | 🚨 `ERROR` | Yes |
| Responding to SV-01: all tasks verified | ✅ `VERIFIED` | Yes (when asked) |
| Responding to SV-01: tasks still incomplete | ⚠️ `INCOMPLETE` | Yes (when asked) |

### 2.2 Message Format

Every message must follow this format:

```
[{TRON_AGENT_ID}] {emoji} {TYPE}: {content}
```

Examples:
```
[ENG-1] 🚀 STARTED: block-04-02-auth-middleware
[ENG-1] ✅ MILESTONE: PR merged for my-service (#45)
[ENG-1] ⏳ HEARTBEAT: Still working on deploy validation
[ENG-1] ⏳ HEARTBEAT: Starting terraform apply — may take up to 5min
[ENG-1] 🏁 DONE: All tasks complete
[ENG-1] ⚠️ BLOCKER: CI failing on my-common — needs investigation
[ENG-1] ❓ QUESTION: Should I use OAuth or direct auth for this endpoint?
[ENG-1] 🚨 ERROR: Deploy failed — containerd snapshot corruption on server-01
```

### 2.3 How to Send

Run this shell command to send a message:

```bash
tron_msg="[${TRON_AGENT_ID}] {your message here}"
timestamp=$(date +%s%N)
echo "${tron_msg}" > "${TRON_META_PATH}/logs/tron/bus/${timestamp}-${TRON_AGENT_ID}.msg"
echo "${tron_msg}"
export TRON_LAST_MSG_TIME=$(date +%s)
```

This writes to the file bus AND prints to terminal. TRON reads the bus and forwards to TG — agents never send to TG directly.

**If the bus directory doesn't exist:** Print to terminal only. Never block work over a failed notification.

### 2.4 Long Messages

Keep bus messages concise. If a message exceeds 4096 characters (TG forwarding limit):

1. Split into chunks at natural boundaries (paragraph breaks, table rows)
2. Append `(1/N)`, `(2/N)` to each chunk
3. Write each chunk as a separate bus file

---

## 3. Polling for Commands

Between major steps in your work, check for messages directed at you.

### 3.1 When to Poll

- **Between every major step** (after completing a task, before starting the next)
- **After sending a DONE message** (TRON will send validation questions)
- **While waiting** for a long operation to complete (if possible)
- You do NOT need to poll continuously. Check between steps — that's sufficient.

### 3.2 How to Poll

Run this shell command to check for new messages from TRON:

```bash
touch -a "${TRON_META_PATH}/logs/tron/bus/.last_read_${TRON_AGENT_ID}" 2>/dev/null
find "${TRON_META_PATH}/logs/tron/bus/" -name "*-TRON.msg" -newer "${TRON_META_PATH}/logs/tron/bus/.last_read_${TRON_AGENT_ID}" 2>/dev/null \
  | sort | while read f; do cat "$f"; done
touch "${TRON_META_PATH}/logs/tron/bus/.last_read_${TRON_AGENT_ID}"
```

This reads all TRON bus messages written since your last poll. The `.last_read_{AGENT_ID}` file tracks your read cursor.

### 3.3 What to Look For

Messages directed at you start with `@{your_agent_id}:`. For example:

```
@ENG-1: skip tests for now, focus on core logic
@ENG-1: Has every single task been delivered, tested, validated?
```

**Action rules:**
- Messages starting with `@{TRON_AGENT_ID}:` → read and act on them
- Messages starting with `@` followed by a different agent ID → ignore (not for you)
- Messages from `[TRON]` with `@{TRON_AGENT_ID}:` → these are supervisor validations, respond to them
- Untagged messages → ignore (TRON will route if relevant to you)

---

## 4. Heartbeat Protocol

You MUST maintain heartbeats so TRON can detect if you stall.

### 4.1 Rules

1. **After every milestone:** Send a `MILESTONE` message. This resets your heartbeat timer.
2. **If 5 minutes pass with no milestone:** Send a `HEARTBEAT` message describing what you're currently doing.
3. **Before long operations:** If you're about to start something that takes >2 minutes (deploy, large test suite, terraform apply), send: `[{TRON_AGENT_ID}] ⏳ HEARTBEAT: Starting {operation} — may take up to {estimate}`

### 4.2 Implementation

Between steps, check elapsed time since your last message. If >5 minutes:

```bash
# Conceptual — integrate into your workflow between steps
current_time=$(date +%s)
if [ $((current_time - TRON_LAST_MSG_TIME)) -gt 300 ]; then
  # Send heartbeat
  tron_msg="[${TRON_AGENT_ID}] ⏳ HEARTBEAT: Still working on: {describe current step}"
  # ... send via §2.3 ...
  export TRON_LAST_MSG_TIME=$current_time
fi
```

### 4.3 If TRON Pings You

If you see a message like:
```
[TRON] 🚨 STALL: @ENG-1 status check — are you still running?
```

Respond immediately with your current status:
```
[ENG-1] ✅ MILESTONE: Still running — was in a long deploy step. Currently at: {status}
```

---

## 5. Responding to Supervisor Validations

TRON will send you validation questions at key moments. These are not optional — respond to every one.

### 5.1 Task Completion (SV-01)

When TRON asks:
```
[TRON] 🔍 @ENG-1: Has every single task from the block been successfully delivered, tested, and validated directly in the server(s)?
```

**Be honest.** If anything is incomplete, say so:
```
[ENG-1] ⚠️ INCOMPLETE: T15 deploy not validated yet. Working on it now.
```

When truly complete:
```
[ENG-1] ✅ VERIFIED: All tasks complete. No pending items.
```

### 5.2 Session End (SV-02)

When TRON instructs:
```
[TRON] 📋 @ENG-1: Read and execute meta/skills/skill-session-end-engineer.md — read it first, now, then execute it without skipping ANY APPLICABLE STEP AT ALL!
```

Do exactly that. Read the skill file, execute every step, then confirm:
```
[ENG-1] ✅ MILESTONE: Session end complete — handover written, log committed, pipeline updated
```

### 5.3 Reviewer Coverage (SV-04)

If TRON tells you files were missed:
```
[TRON] 🔍 @REV-1: You missed the following files: src/auth.py, src/routes.py. Review them before returning.
```

Review those files, then re-report your complete findings.

---

## 6. Environment Variables Reference

These are set by TRON when you are spawned. Do not modify them.

| Variable | Purpose | Example |
|:--|:--|:--|
| `TRON_AGENT_ID` | Your unique identifier | `ENG-1` |
| `TRON_AGENT_ROLE` | Your role | `engineer` |
| `TRON_BLOCK` | Block you're working on | `block-04-02-auth-middleware` |
| `TRON_META_PATH` | Path to project's `meta/` | `/path/to/project/meta` |
| `TRON_POLL_OFFSET` | (Deprecated — bus uses file cursors) | N/A |
| `TRON_HEARTBEAT_INTERVAL` | Seconds between heartbeats | `300` |
| `TRON_POLL_INTERVAL` | Seconds between bus polls | `30` |
| `TRON_TRANSPORT` | Active transport | `tg` or `cli` |
| `TRON_LAST_MSG_TIME` | Timestamp of your last sent message | Unix timestamp |

---

## 7. Bus File Convention

All messages go through the file bus at `{TRON_META_PATH}/logs/tron/bus/`.

**File naming:** `{timestamp}-{SENDER_ID}.msg` (e.g., `1710500000000000000-ENG-1.msg`, `1710500000100000000-TRON.msg`)

**Read cursors:** `.last_read_{AGENT_ID}` files track each reader's position.

**Lifecycle:** Bus files accumulate during a session. TRON cleans up at session end.

**If TG is not configured (CLI-only mode):** The bus still works identically. The only difference is TRON won't forward to TG — user must read the terminal or bus files directly. This is degraded mode (no remote access).

---

**Last Updated:** 2026-03-15 (v2: bus-first architecture — agents write to bus, TRON forwards to TG)
