# TRON State

Persistent state for TRON sessions. Updated after every session.

---

## Session History

- **Last session:** never
- **Total sessions:** 0
- **Last reviewer run:** never

## Configuration

- **HEARTBEAT_INTERVAL:** 300
- **GRACE_PERIOD:** 120
- **POLL_INTERVAL:** 30
- **MAX_CONCURRENT_AGENTS:** 5
- **TRANSPORT:** tg
- **DEFAULT_SPAWN_MODE:** interactive

## Active Notifications

| Event | Active |
|:--|:--|
| `SESSION_START` | ✅ |
| `SPAWNED` | ✅ |
| `SV-PASS` | ✅ |
| `SESSION_COMPLETE` | ✅ |
| `PIPELINE_EXHAUSTED` | ✅ |
| `BLOCKER` | ✅ (always) |
| `QUESTION` | ✅ (always) |
| `ERROR` | ✅ (always) |
| `STALL` | ✅ (always) |
| `UNRESPONSIVE` | ✅ (always) |
| `SESSION_ABORTED` | ✅ (always) |

## Agent Session-End Skills

| Role | Skill Exists | Path |
|:--|:--|:--|

## Watch Items

None
