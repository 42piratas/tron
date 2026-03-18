# ADR: TRON v0.2 — TG Message Bus & Active Supervision

**Status:** Draft
**Date:** 2026-03-13
**Context:** SUPER-M research session — TRON improvement and architectural evolution
**Requirements gathered:** 9 rounds of Q&A with user

---

## 1. Problem Statement

TRON v0.1 operates as a prompt doc inside Claude Code. It plans sessions and collects returns, but:

- **No observability:** User has no visibility into agent progress between session start and return
- **No stall detection:** TRON can't detect when agents hang or loop — separate conversation contexts are blind to each other
- **No inter-agent communication:** Agents can't send messages to TRON or to each other
- **No active supervision:** TRON can't validate agent work quality mid-session — user must manually catch incomplete tasks, missed files, skipped steps
- **No remote interaction:** User must be at the terminal to interact with agents
- **Manual spawning:** User must manually open terminals and paste spawn commands
- **Text-only input:** User must type all commands — no voice input support on desktop or mobile

---

## 2. Key Design Decisions

### 2.1 Telegram as Message Bus

TG channel (one per project) becomes the communication backbone, not just a notification layer.

**Participants:**
- Agents SEND: tagged status, heartbeats, milestones, returns
- TRON POLLS: tracks all agents, detects stalls, validates returns, routes messages
- User SENDS: free-text commands, tagged or untagged
- Agents POLL: pick up messages tagged to them, act on commands

**Message tagging convention:**
- `[TRON]` — TRON messages
- `[ENG-1]`, `[ENG-2]` — Engineer instances (identified by task/thread)
- `[REV-1]` — Reviewer instances
- `[ARCH-1]` — Architect instances
- `[USER]` — Optional explicit tag, but untagged messages are assumed to be from user
- `@ENG-1:` — Routing prefix for directing messages to specific agents

**Why TG over alternatives:**
- Zero infrastructure — curl is the only dependency
- User participates natively (phone/desktop TG client)
- Remote access — interact from anywhere
- Multi-platform expansion path (Discord, Slack) via transport adapters
- Already set up and proven for project notifications

### 2.2 Transport-Agnostic Protocol

The comms protocol is defined independently of the transport layer:

```
Agent Logic → Comms Interface → Transport Adapter (TG | CLI-only | Discord | ...)
```

- **TG adapter (v1):** Primary. curl-based send, `getUpdates`-based poll.
- **CLI-only fallback:** Agents write to shared log directory. TRON watches files. User interacts via terminal only. Degraded but functional — no external dependency.
- **Discord adapter (future):** Same protocol, different HTTP endpoints.

Adding a channel = adding a transport adapter, not redesigning the system.

### 2.3 Agent Heartbeat & Stall Detection

**Heartbeat rules:**
- Every agent sends a status message at each milestone (task complete, deploy triggered, PR created, phase boundary, etc.)
- If no milestone is reached within 5 minutes, agent sends an idle heartbeat: `[ENG-1] ⏳ Still working on: {current step}`
- Heartbeat interval is configurable per project (default: 5 min)

**Stall detection (TRON-side):**
- TRON tracks last message timestamp per agent tag
- If no message received from an agent for > heartbeat interval + grace period (default: 5min + 2min = 7min) → TRON sends stall alert:
  - To TG: `🚨 [TRON] Agent [ENG-1] has not reported in 7min — possibly stalled`
  - TRON attempts to ping the agent via TG: `@ENG-1: status check — are you still running?`
- If agent responds to ping → false alarm, reset timer
- If no response after additional 5 minutes (12min total from last message) → escalate to user: `🔴 [TRON] Agent [ENG-1] unresponsive. Manual intervention likely needed.`

**Limitation:** If an agent is in a long-running step (e.g., 10min deploy), it can't send heartbeats mid-step. Agents should send a "starting long operation" message before such steps: `[ENG-1] ⏳ Starting deploy — may take up to 10min`

### 2.4 TRON as Active Supervisor

TRON evolves from passive coordinator to active supervisor with validation loops.

**Validation flow:**
```
Agent: "Phase complete" → TG
TRON: parses return against checklist for that agent type
  ├── Checklist passes → present to USER for final approval
  └── Checklist fails → send follow-up to agent via TG
        Agent: fixes/completes, re-reports → TG
        TRON: re-validates → loop until checklist passes
TRON: presents validated return to USER
User: approves or rejects (TG or terminal)
  ├── Approves → TRON proceeds
  └── Rejects → TRON sends agent back with user's feedback
```

**Critical rule: User is always the final gate.** TRON pre-filters and validates, but never approves on behalf of the user.

**Supervisor interventions — encoded from recurring manual prompts:**

These are patterns the user currently handles manually every session. TRON must automate them.

#### SV-01: Task Completion Verification (Engineer)

**Trigger:** Engineer reports tasks complete (any phase boundary or "done" signal).
**TRON action:** Before accepting the return, TRON asks the engineer:

> "Has every single task from the block been successfully delivered, tested, and validated directly in the server(s)? Any UI or TG tasks the user may need to test?"

**Expected behavior:** Engineers almost always have something still open. If the engineer reveals incomplete items → TRON sends them back to finish. When engineer reports complete again → TRON asks the same question again (re-verification loop). Only when the engineer confirms with zero open items does TRON proceed to SV-02.

**Loop:** Ask → Agent completes remaining → Ask again → Confirm zero open → Proceed.

#### SV-02: Session End Enforcement (All Roles with Session-End Skills)

**Trigger:** Agent has passed its completion verification (e.g., SV-01 for engineers).
**TRON action:** TRON checks if `meta/skills/skill-session-end-{role}.md` exists for this agent's role. If it does, TRON instructs the agent:

> "Read and execute `meta/skills/skill-session-end-{role}.md` — read it first, now, then execute it without skipping ANY APPLICABLE STEP AT ALL!"

**Note:** Headless agents cannot use slash commands. The instruction must reference the skill file path explicitly. Interactive agents can use either form.

**Applies to:** Any role that has a corresponding `skill-session-end-{role}.md` in `meta/skills/`. TRON-SEED discovers which exist during seeding and records them in `tron.md`.

**Rationale:** Including session-end in the startup protocol doesn't work — agents forget by the end of long sessions. This must be triggered punctually at the exact moment of task completion, not earlier.

**TRON must verify:** Agent actually executed the session-end skill. If agent returns without evidence of session-end steps (handover written, logs committed, pipeline updated, etc.) → TRON sends them back.

#### SV-03: Startup Directives (All Agents)

**Trigger:** Immediately after any agent completes its startup protocol.
**TRON action:** TRON sends to the agent (via TG or as part of the spawn prompt):

> "CRITICAL DIRECTIVE: ALWAYS BE VERY CONCISE! RELEVANT CONSIDERATIONS, QUESTIONS, AND ACTIONABLE ITEMS ONLY!
>
> WARNING: There are other AGENTS working in parallel to this session. Follow all best practices regarding BRANCHES and WORKTREES to make sure there are no conflicts. Any questions or considerations?"

**Rationale:** Agents must be concise (saves tokens, reduces noise) and must be aware of parallel work to avoid git conflicts. This is injected after startup — not before — so it's the last thing the agent reads before beginning work.

#### SV-04: Reviewer Coverage Verification (Reviewer)

**Trigger:** Reviewer reports findings / returns.
**TRON action:** TRON cross-checks the reviewer's scope against the actual git diff:
- List all changed files in the commit range
- Compare against files the reviewer reported reviewing
- If files are missing → TRON sends reviewer back: "You missed files: {list}. Review them before returning."

**Loop:** Same as SV-01 — re-verify until coverage is complete.

#### Future SVs

Additional supervisor validations will be added as patterns are identified. Each follows the same structure: trigger → TRON action → verification loop → escalate to user only when validated.

### 2.5 Agent Spawning

TRON spawns agents as **independent Claude Code processes** — not sub-agents. Sub-agents (Claude Code's built-in Agent tool) run inside TRON's context, blocking TRON and preventing direct user interaction. Independent processes run in their own terminals, in parallel, with full user access.

**Two spawn modes (user chooses per agent, or sets project default):**

| Mode | How | User can talk to it? | Runs in parallel? |
|:--|:--|:--|:--|
| **Interactive terminal** | TRON opens a new terminal window (iTerm2 API / `osascript`) with `claude` running | Yes — direct terminal access | Yes |
| **Headless + TG** | TRON runs `claude --print "{prompt}" &` in background | Yes — via TG only | Yes |

**Interactive terminal spawn (macOS):**
```bash
# iTerm2
osascript -e 'tell application "iTerm"' -e 'activate' -e 'create window with default profile' -e 'tell current session of current window' -e 'write text "{meta_path}/logs/tron/spawn-{AGENT_ID}.sh"' -e 'end tell' -e 'end tell'

# Terminal.app
osascript -e 'tell application "Terminal" to do script "claude --model {model} -p \"{agent prompt}\""'
```

**Headless spawn:**
```bash
claude --model {model} -p "{agent prompt}" --output-format stream-json &
```

**User→Agent interaction model:**
- **Terminal mode:** User switches to the agent's terminal tab and types directly. Agent also reports to TG for remote access and TRON visibility.
- **Headless mode:** User sends `@ENG-1: {instruction}` in TG. Agent polls TG, picks up the message, acts on it.
- **Both modes:** Agent sends heartbeats and milestones to TG. TRON monitors all exchanges. User can interact via TG regardless of spawn mode.

**Spawning command issued by TRON:**
```
User to TRON (terminal): "Start 2 engineers on B04-T15 and B04-T16"
TRON: validates against max concurrent agents
TRON: spawns ENG-1 (B04-T15) and ENG-2 (B04-T16) — interactive or headless per config
TRON: [TRON] Spawned ENG-1 for B04-T15, ENG-2 for B04-T16 → TG
```

**Max concurrent agents:** Configurable per project. Default: 5. Limit driven by:
- TRON context window (must track state of all active agents)
- Machine resources (CPU/memory per Claude Code process)
- User attention (diminishing returns beyond 3-4 meaningful parallel streams)

### 2.6 Agent Identification & Enforcement

Each agent instance is identified by:
- **Tag:** `[ENG-1]`, `[REV]`, `[ARCH]`, etc.
- **Thread/task:** The specific work item (e.g., `B04-T15`, `adhoc-hotfix`)
- **Agents must be able to answer "what are you working on?"** when queried

Multiple instances of the same role are supported (e.g., 2 engineers on different tasks). Each gets a unique numeric suffix.

**Enforcement — two layers (belt and suspenders):**

1. **Prompt injection at spawn:** TRON includes identity in the agent's launch prompt:
   ```
   Your agent ID is [ENG-1]. You are working on B04-T15.
   Tag ALL outgoing messages (terminal and TG) with [ENG-1].
   When asked "what are you working on?", respond with your ID and task.
   ```
   This gives the agent self-awareness of its identity and task.

2. **Wrapper script for TG sends:** A shared shell function (provided by `skill-tg-comms.md`) that prepends the tag automatically:
   ```bash
   # Agent calls: tron_send "Deploy complete, PR merged"
   # Wrapper sends: "[ENG-1] Deploy complete, PR merged" → TG
   tron_send() {
     local msg="[${TRON_AGENT_ID}] $1"
     eval "$(cat meta/.env)" && curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
       -d chat_id="${TELEGRAM_TRON_CHAT_ID}" \
       -d parse_mode="Markdown" \
       -d text="${msg}" > /dev/null
   }
   ```
   `TRON_AGENT_ID` is set as an environment variable by TRON at spawn time. The agent cannot forget or omit the tag — the wrapper enforces it.

**Result:** Agent knows its identity (prompt). Tags are guaranteed on TG messages (wrapper). Terminal output relies on prompt compliance — acceptable since the user is directly reading that terminal.

### 2.7 Agent Roster & Discovery

TRON does not assume a fixed roster. It discovers whatever agents exist in `meta/agents/` during seeding and adapts.

**Discovery during seeding:**
- Scan all `.md` files in `meta/agents/`
- Exclude `tron.md` (self), `super-m-local.md` (SUPER-M state, not an agent TRON spawns)
- For each agent found: record role name, whether a `skill-session-end-{role}.md` exists, whether a `handover-{role}.md` exists
- Ask user which agents TRON should orchestrate (some may be user-invoked only)

**Known role behaviors:**

| Role | Mode | Handover | Session-End Skill | Notes |
|:--|:--|:--|:--|:--|
| Engineer(s) | Spawned by TRON | Required | Required | Multiple instances supported. One per block. |
| Reviewer | Spawned by TRON | Written by TRON | Check if exists | Spawned only when there are new commits since last review. Spawns own sub-agents per repo. |
| Architect | On-demand | Not required | Check if exists | User-invoked for brainstorming, design, scoping. TRON can spawn when engineer hits architectural blocker. |
| Analyst | Occasional | Per project | Per project | Functions vary by project. Not present in most projects. |
| Other roles | Discovered | Per project | Per project | TRON adapts to whatever exists in `meta/agents/`. |

**Model defaults (TRON suggests, user confirms):**

| Role | Suggested Model | Rationale |
|:--|:--|:--|
| Architect | Opus | Complex reasoning, design decisions |
| Engineer | Sonnet | Code generation, execution speed |
| Reviewer | Sonnet | Code analysis, pattern matching |
| Other | Ask user | No assumption |

### 2.8 Polling Interval

Agents and TRON are not constantly listening to TG. They periodically ask TG "any new messages?" by calling the `getUpdates` API. This is "polling."

```
Every 30 seconds, between steps:
  Agent runs: curl .../getUpdates?offset={last_seen+1}
  TG responds: [list of new messages since last poll] or []
  Agent: reads messages tagged to it, acts on them
  Agent: resumes work
```

**Why agents can't listen continuously:** Claude Code executes sequentially — it's either doing work (reading files, running commands, writing code) or checking TG. It can't do both simultaneously. Polling happens in the gaps between steps.

**Implication:** If you send `@ENG-1: stop` at second 0 and the agent just started a 3-minute deploy step, it won't see your message until the deploy finishes and it polls again. The 30s interval only applies when the agent is between short steps.

**Default: 30 seconds for all participants.**

| Concern | Impact |
|:--|:--|
| Responsiveness | Max 30s delay between steps. During long steps, delay = step duration. |
| TG API rate limits | 30 req/s global limit. 5 agents polling every 30s = ~10 calls/min. Nowhere near limit. |
| Agent interruption | Agents check TG only between steps. Long steps (deploys, large file operations) delay polling regardless of interval. |
| Token cost | Zero — polling is curl, not LLM calls. |
| Stall detection speed | Worst case: heartbeat_interval (5min) + poll_interval (30s) + grace (2min) = 7.5min to detect stall. |

Configurable per project. TRON may use a shorter interval (15s) for faster stall detection if needed.

### 2.9 Session Flow Rules

Gathered from requirements — these govern how TRON orchestrates a session.

#### Session Start
1. Read `pipeline.md` `#roadmap` — identify active phase, available blocks
2. Read handovers for all active agent roles
3. Check block dependencies — determine which blocks can run in parallel
4. **Ask user:** "How many parallel engineers this session? Which blocks?" — always ask, never assume from last session
5. Suggest model per agent role (Opus/Sonnet/etc.) — user confirms
6. Present session plan — wait for explicit confirmation
7. On confirmation → spawn agents

#### During Session
- TRON monitors all agents via TG polling
- Stall detection active (§2.3)
- User can send commands to any agent via TG or terminal
- If engineer hits architectural blocker → TRON can spawn architect on-demand, coordinate the exchange, resume engineer when resolved
- If reviewer reports zero commits in scope → skip reviewer this session

#### Block Completion
1. Agent reports "done" → TRON fires SV-01 (task completion verification loop)
2. SV-01 passes → TRON fires SV-02 (session-end skill enforcement, if skill exists for role)
3. SV-02 passes → TRON presents validated return to user
4. **User approves or rejects** — user is always the final gate
5. If user approves → TRON asks: "Proceed to next block?" — always ask, never auto-proceed
6. If user says yes → TRON checks next block dependencies, spawns new engineer
7. If pipeline exhausted → notify user

#### Phase-End Gate (when last block of a phase completes)
1. Engineer finishes last block → SV-01 + SV-02 as usual
2. Reviewer runs (if changes in that block) → findings reported
3. Engineer fixes ALL reviewer findings — no deferrals unless engineer provides a logically justified reason. TRON enforces: "Fix all findings before proceeding."
4. After all findings resolved → TRON spawns Architect for phase-end cleanup:
   - Review all block specs from the completed phase
   - Review session logs from the phase
   - Verify pipeline is accurate and up-to-date
   - Archive completed block specs to `meta/blocks/archive/`
   - Ensure docs are clean and polished
5. Architect returns → TRON presents to user for final phase approval
6. Phase is not done until architect signs off on docs

#### Session End
1. All agents have returned and user has approved
2. TRON writes session log to `meta/logs/tron/log-YYMMDD-HHMM-{desc}.md`
3. TRON updates `meta/logs/tron/tron-state.md`
4. TRON commits and pushes `meta/` only (logs, handover updates) — never application repos
5. TRON notifies user via TG: session complete

#### TRON Crash Recovery
- If TRON crashes mid-session → on restart, warn user and wait for instructions
- Do not attempt to auto-recover or resume — state may be inconsistent
- TG history provides an audit trail of what happened before the crash

### 2.10-deferred Speech-to-Prompt Input — DEFERRED to v0.3+

Voice input (desktop via VoiceInk + TG voice message transcription) is a confirmed future capability. See previous research in this session for full analysis. Not in v0.2 scope — focus v0.2 on core orchestration, TG bus, and active supervision first.

### 2.10 Project Structure — Discovery & Alignment

TRON does **not** create or enforce project structure — that's SUPER-M / Architect territory. TRON discovers what exists, aligns to it, and fills in only TRON-specific files. The `meta/` pattern is the reference structure.

#### Workspace Layout

```
workspace/                              ← e.g., ~/workspace/
├── {project}/                          ← a project (e.g., my-app/)
│   ├── meta/                           ← agent operating system
│   ├── {service-repos}/                ← application code (independent git repos)
│   └── ...
├── shared-knowledge/                   ← cross-project knowledge base
├── super-m/                            ← cross-project process auditor
└── tron/                               ← cross-project session orchestrator (this repo)
```

#### `meta/` Structure

```
meta/
├── context.md                          ← project map: repos, access, doc hierarchy
├── principles.md                       ← project-specific rules (extends shared-knowledge/principles-base.md)
├── pipeline.md                         ← SINGLE source of truth for active work (#roadmap + #technical-debt)
├── pipeline-archive.md                 ← completed phases (read-only reference)
├── backlog.md                          ← unscoped ideas — not planned, not tracked by agents
│
├── agents/                             ← agent identity docs
│   ├── engineer.md
│   ├── architect.md
│   ├── reviewer-code.md
│   ├── reviewer-security.md            ← optional, per project needs
│   ├── super-m-local.md                ← SUPER-M's persistent local context
│   ├── tron.md                         ← TRON orchestrator config (project-specific, created by tron-seed)
│   └── ...                             ← other roles as needed
│
├── blocks/                             ← work specifications + handover files
│   ├── block-NN-TT-*.md               ← task specs (phase-sequence-description)
│   ├── handover-engineer.md            ← engineer inter-session state
│   ├── handover-architect.md           ← architect inter-session state
│   ├── handover-reviewer-code.md       ← reviewer scope (written by TRON each session)
│   ├── handover-*.md                   ← one per agent role
│   └── archive/                        ← completed block specs
│
├── skills/                             ← reusable procedures and checklists
│   ├── skill-*.md                      ← task templates agents can invoke
│   └── checklist-*.md                  ← session-end checklists per role
│
├── logs/                               ← session records, organized by role
│   ├── engineering/
│   ├── architecture/
│   ├── review-code/
│   ├── review-security/
│   ├── super-m/
│   └── tron/
│
├── reports/                            ← analysis outputs, research, lectures
├── tmp/                                ← USER USE ONLY — agents do not touch
├── util/                               ← USER USE ONLY — agents do not touch
├── .env                                ← local credentials (gitignored) — TG bot tokens, etc.
└── .gitignore                          ← must include .env, tmp/
```

#### How the Layers Work

| Layer | Files | Purpose |
|:--|:--|:--|
| **Rules** | `shared-knowledge/principles-base.md` → `meta/principles.md` → `meta/agents/*.md` | Cascading rules: universal → project-specific → role-specific. Never duplicated. |
| **Context** | `meta/context.md`, `meta/pipeline.md`, `meta/blocks/*.md` | What agents need to know. Pipeline is the single status tracker. |
| **Continuity** | `meta/blocks/handover-*.md`, `meta/logs/*/log-*.md` | How context survives between sessions. Handovers bridge the session boundary. |
| **Execution** | `meta/agents/*.md`, `meta/skills/*.md`, `meta/blocks/block-*.md` | Who the agent is + how to do tasks + what to do now. |

#### Session Lifecycle (every agent, every session)

```
START
  ├─ Read meta/agents/{role}.md              ← know who I am
  ├─ Read shared-knowledge/principles-base.md ← know universal rules
  ├─ Read meta/principles.md                  ← know project rules
  ├─ Read meta/context.md                     ← know the project
  ├─ Read meta/pipeline.md                    ← know what's active
  ├─ Read meta/blocks/handover-{role}.md      ← know what happened last
  │
  ├─ WORK (guided by block spec + skills)
  │
  ├─ Write meta/blocks/handover-{role}.md     ← leave context for next session
  ├─ Write meta/logs/{role}/log-*.md          ← record what happened
  └─ Update meta/pipeline.md                  ← reflect status changes
END
```

#### Key Conventions

- **`meta/` is always direct-to-main** — no branch protection, no PR
- **Each service repo is independent** — never run git from the project parent directory
- **`tmp/` and `util/` are user-only** — agents must never read or write to them
- **`backlog.md` is idea storage** — agents do not pull work from it unless user explicitly moves items to `pipeline.md`
- **`pipeline.md` is the single source of truth** for active work and technical debt — no other document tracks status
- **`principles.md` extends shared base** — project-specific rules override shared rules on conflict
- **One source of truth per fact** — `pipeline.md` owns status, `context.md` owns structure, `principles.md` owns rules
- **Handovers bridge sessions** — AI agents have no persistent memory; handover docs create continuity
- **Parallel-safe by design** — git worktrees isolate concurrent work; handovers are per-role; logs are per-role; no shared mutable state between agents

#### TRON-SEED Discovery During Seeding

TRON-SEED discovers and aligns. It does not create project structure.

**Required (abort if missing):**
- `meta/agents/` — with at least one agent doc
- `meta/logs/` — with at least one role subdirectory
- `meta/skills/`
- `meta/pipeline.md`
- `meta/context.md`
- `meta/principles.md`

**TRON-SEED creates only its own files:**
- `meta/agents/tron.md` — project-local orchestrator config
- `meta/logs/tron/` — TRON session log directory
- `meta/logs/tron/tron-state.md` — persistent TRON state
- `meta/blocks/handover-reviewer-code.md` — reviewer scope file (written by TRON each session)
- `meta/.env` — TG credentials (if user provides them)
- Ensures `meta/.gitignore` includes `.env`
- TG channel — created programmatically via Bot API (if user confirms)

**TRON-SEED discovers and records:**
- All agent docs in `meta/agents/` → builds agent roster
- All `skill-session-end-*.md` in `meta/skills/` → maps which roles have session-end skills
- All `handover-*.md` in `meta/blocks/` → maps which roles have handovers
- All subdirectories in `meta/logs/` → maps log paths per role
- Block naming convention (from existing blocks in `meta/blocks/`)

#### Block Naming Convention

TRON must understand block names to track dependencies and manage ad-hoc insertions.

**Planned blocks:**
```
block-{phase}-{sequence}-{description}.md
```
Example: `block-04-02-auth-middleware.md` (phase 4, second block)

**Ad-hoc blocks:**
```
block-{phase}-adhoc-{sequence}-{description}.md
```
Example: `block-04-adhoc-01-hotfix-auth.md` (ad-hoc during phase 4, first ad-hoc)

**Rules:**
- Phase number = the phase during which the ad-hoc is executed (or the phase it must happen before)
- Phases and ad-hoc tasks are sequential — ordering matters
- Each block spec must include a `Depends on:` field listing block dependencies (e.g., `Depends on: block-04-01`)
- TRON reads dependencies to determine which blocks can run in parallel

#### Pipeline Rules

- TRON reads `#roadmap` section of `pipeline.md` for session planning
- TRON ignores `#technical-debt` section — debt items appear in the roadmap only when user + architect place them there
- If the pipeline roadmap is exhausted (no more blocks to execute) → TRON notifies the user: `🔔 [TRON] Pipeline exhausted — no more blocks in roadmap. Architect + user coordination needed.`
- Engineers update pipeline status as part of their session-end protocol — TRON verifies it was done, does not do it itself
- When user requests an ad-hoc task, TRON inserts it into the pipeline at the correct sequential position and creates the block spec naming accordingly

---

## 3. Components to Build

| # | Component | Description | Depends on |
|:--|:--|:--|:--|
| 1 | **Comms protocol spec** | Message format, tags, heartbeat spec, stall thresholds, validation loop spec | Nothing |
| 2 | **Shared skill: `skill-tg-comms.md`** | How agents send, poll, parse TG messages. Transport-agnostic interface. | #1 |
| 3 | **TG transport adapter** | curl-based send/poll implementation. `getUpdates` with offset tracking. | #1 |
| 4 | **CLI-only fallback adapter** | File-based send/watch for no-TG environments. | #1 |
| 5 | **Supervisor validation rules (SV-01 to SV-04+)** | Encoded checklists per agent type, verification loops, escalation logic | #1, user's detailed prompts |
| 6 | **TRON v0.2 doc** | Full rewrite: supervisor logic, validation loops, spawn, polling, stall detection, comms integration | #1–5 |
| 7 | **tron-seed.md v0.2** | Seeding includes: comms setup, project structure validation, TG channel, .env, agent ID assignment | #6, §2.10 structure |
| 8 | **Agent doc updates** | Engineer, Reviewer, Architect get comms integration (heartbeat, milestone reporting, TG polling, command handling, SV compliance) | #2, #5, #6 |
| 9 | **Agent spawn wrapper** | Shell script for TRON to spawn Claude Code processes with agent ID, TG env, and proper prompts | #2, #6 |
| 10 | **Seed new project** | First real validation of tron-seed.md v0.2 on a brand new project | #7–9 |
| 11 | **Migrate existing project** | Update existing TRON instance to v0.2 after new project validates the seed | #10 |

---

### 2.11 Headless Mode Capabilities & Constraints

Validated via [Claude Code headless documentation](https://code.claude.com/docs/en/headless):

**Supported:** All tools (Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch), curl for TG API, `--resume` for multi-turn sessions, `--allowedTools` for auto-approval, `--max-turns` to limit runaway, `--output-format stream-json` for real-time monitoring.

**Key spawn pattern:**
```bash
claude -p "{agent prompt}" \
  --model claude-sonnet-4-6 \
  --allowedTools "Bash,Read,Write,Edit,Glob,Grep" \
  --output-format stream-json \
  --max-turns 50
```

**Limitations vs interactive mode:**
- **No slash commands / skills.** SV-02 cannot fire `@session-end-engineer`. Must use: "Read and execute `meta/skills/skill-session-end-{role}.md`" instead.
- **No interactive permission prompts.** Must pre-approve tools via `--allowedTools`.
- **Not inherently long-running.** Each `-p` invocation is bounded. For multi-block sessions, TRON spawns a new process per block (or uses `--resume` with session ID).
- **Session resume:** Capture `session_id` from JSON output, pass via `--resume {id}` for follow-up interactions.

**Implication for TRON:**
- Interactive terminal spawn = full Claude Code with slash commands (user's preferred mode)
- Headless spawn = all tools but no slash commands (skill references must be explicit file paths)
- TRON must adjust SV-02 message based on spawn mode

---

## 4. Open Questions

1. ~~**`--print` mode limitations:**~~ **RESOLVED.** All tools supported. No slash commands — skills must be referenced by file path.
2. **TG message length limits:** TG max message = 4096 chars. Long returns (reviewer findings tables) may need splitting. Define chunking strategy.
3. **Agent crash recovery:** If a spawned process dies, TRON detects via PID check. Should it auto-respawn or alert user? → **RESOLVED: alert user and wait.**
4. **Concurrent git access:** Multiple engineers on different branches in same repo — worktree isolation mandatory. Enforce in spawn logic. → **RESOLVED: blocks declare dependencies, TRON checks with user before parallel spawning.**
5. **TRON crash recovery:** → **RESOLVED: warn user and wait. Do not auto-recover.**
6. **TG channel creation:** TRON-SEED creates programmatically via Bot API — need to validate bot permissions required (createChannel or createSupergroup).
7. ~~**Block dependency parsing:**~~ **RESOLVED.** Format: `Depends on: block-NN-TT-name, block-NN-TT-other` (comma-separated block IDs). `Depends on: none` for blocks with no dependencies.

---

## 5. Architecture Comparison Matrix: Prompt Doc + TG Bus vs Runtime Wrapper

Comparison across all requirements and use cases raised during this research session.

### Legend

- **PD+TG** = Prompt Doc (TRON v0.2) + Telegram Message Bus
- **RW** = Runtime Wrapper (Praktor-style external orchestrator)
- ✅ = Strong fit | ⚠️ = Possible with tradeoffs | ❌ = Not feasible or very difficult

### 5.1 Core Orchestration

| Use Case | PD+TG | RW | Notes |
|:--|:--|:--|:--|
| Spawn agents programmatically | ⚠️ `claude --print &` via shell | ✅ Native process management | PD+TG: works but headless. RW: full lifecycle control (spawn, kill, restart). |
| Kill/restart stalled agents | ⚠️ TRON can `kill PID` but blind to internal state | ✅ Process-level control with state inspection | PD+TG: can kill process, can't gracefully resume mid-task. RW: can checkpoint and restart. |
| Track agent lifecycle (running/done/crashed) | ⚠️ PID check + TG heartbeat absence | ✅ Direct process monitoring, exit codes | PD+TG: indirect detection via heartbeat silence. RW: immediate via process events. |
| Max concurrent agent enforcement | ✅ TRON tracks count before spawning | ✅ Built into runtime scheduler | Both handle this well. |
| Multiple engineers on parallel tasks | ✅ Each spawned with unique tag, separate worktrees | ✅ Each in isolated container/process | Both work. RW has stronger isolation (Docker). |

### 5.2 Observability & Feedback

| Use Case | PD+TG | RW | Notes |
|:--|:--|:--|:--|
| Real-time agent progress visibility | ✅ TG heartbeats + milestones every step | ✅ Stdout capture + dashboard | PD+TG: slight polling delay (30s). RW: true real-time via stdout pipe. |
| Stall detection | ⚠️ Heartbeat timeout (5-7min worst case) | ✅ Process-level (seconds) | PD+TG: depends on heartbeat interval. RW: can detect hung process immediately. |
| Terminal output for all agents | ⚠️ Only TRON has interactive terminal; agents report via TG | ✅ Can multiplex all agent stdout to one terminal/dashboard | PD+TG: agents are headless, output goes to TG. RW: can aggregate all output. |
| Remote monitoring (phone/away from desk) | ✅ Native — TG client on any device | ⚠️ Needs web dashboard or TG integration built separately | PD+TG: free. RW: must build the remote layer. |
| Configurable notification tiers (terminal vs TG) | ✅ Protocol defines which events go where | ✅ Same — config-driven routing | Both handle this equally. |

### 5.3 User Interaction

| Use Case | PD+TG | RW | Notes |
|:--|:--|:--|:--|
| User sends free-text commands to specific agent | ✅ `@ENG-1: {command}` via TG, agent polls | ✅ Via UI/API/TG — routed by runtime | Both work. RW may have lower latency. |
| User gives final approval on agent returns | ✅ TRON presents via TG or terminal, user responds | ✅ Via UI/API | Both work. PD+TG: user can approve from phone. |
| Direct terminal interaction with individual agents | ✅ Interactive terminal spawn mode — each agent gets its own tab | ⚠️ Possible if runtime exposes per-agent terminals | PD+TG: user chooses interactive or headless per agent. RW: depends on implementation. |
| User interacts while away from workstation | ✅ Full TG interaction from phone | ⚠️ Only if runtime has mobile-friendly UI or TG integration | PD+TG: native strength. RW: must build this. |
| Bidirectional TG communication | ✅ Core design — agents poll, user sends | ⚠️ Must integrate TG as I/O channel | PD+TG: built-in. RW: add-on. |

### 5.4 Quality & Supervision

| Use Case | PD+TG | RW | Notes |
|:--|:--|:--|:--|
| Validate agent returns against checklists | ✅ TRON parses TG messages, runs validation logic | ✅ Runtime validates returns programmatically | Both work. RW can be more structured (typed returns). |
| Send agents back for incomplete work | ✅ TRON sends follow-up via TG, agent polls and acts | ✅ Runtime requeues task to agent | Both work. PD+TG: async loop via TG. RW: direct requeue. |
| Enforce deploy gates before proceeding | ✅ TRON checks return, blocks progression via TG | ✅ Runtime enforces gates in DAG | Both work. RW: more reliable (can't bypass). PD+TG: agent must cooperate. |
| Cross-agent coordination (e.g., "wait for ENG-1 before starting ENG-2") | ⚠️ TRON manages sequencing via TG message tracking | ✅ Native DAG/dependency support | RW: stronger. PD+TG: TRON must manually track dependencies. |
| Reviewer coverage validation (all files reviewed?) | ✅ TRON cross-checks reviewer return against git diff | ✅ Same — programmatic check | Both work equally. |

### 5.5 Architecture & Maintenance

| Use Case | PD+TG | RW | Notes |
|:--|:--|:--|:--|
| Setup complexity | ✅ Markdown files + TG bot (already exists) | ❌ Docker + runtime install + config + maintenance | PD+TG: minutes to set up. RW: hours/days. |
| Customization | ✅ Edit markdown — anyone can modify | ⚠️ Code changes in Go/Python/etc. | PD+TG: low barrier. RW: requires dev skills. |
| Debugging | ✅ Read the conversation + TG history | ⚠️ Logs + dashboards, but opaque internals | PD+TG: full transparency. RW: depends on logging quality. |
| Portability across machines | ✅ Works anywhere Claude Code + curl exist | ⚠️ Requires Docker/runtime on each machine | PD+TG: zero dependencies. RW: environment-specific. |
| Failure blast radius | ✅ One agent crashes = others unaffected | ⚠️ Runtime crash = all agents die | PD+TG: isolated by design. RW: single point of failure. |
| Multi-platform expansion (Discord, Slack) | ✅ Add transport adapter, protocol unchanged | ⚠️ Must integrate each platform into runtime | PD+TG: designed for this. RW: per-platform work. |
| Evolution to runtime later | ✅ Protocol and message formats transfer directly | N/A | PD+TG is a stepping stone — nothing is throwaway. |

### 5.6 Scalability & Edge Cases

| Use Case | PD+TG | RW | Notes |
|:--|:--|:--|:--|
| 3-4 agents (typical session) | ✅ Comfortable | ✅ Comfortable | Both fine. |
| 5-8 agents (heavy session) | ⚠️ TRON context window pressure | ✅ Runtime scales independently | PD+TG: TRON must summarize/compress. RW: no context limit. |
| 10+ agents | ❌ TRON context window saturated | ✅ Limited only by infra | PD+TG: not designed for this scale. |
| Long-running sessions (hours) | ⚠️ Claude Code context compression may lose early messages | ✅ Persistent state in runtime DB | PD+TG: TG history provides backup. RW: native persistence. |
| Offline/no-internet operation | ⚠️ CLI-only fallback (degraded) | ✅ Fully local | PD+TG: loses TG. RW: unaffected. |
| CI/CD integration (trigger agents from pipeline) | ❌ Needs human to invoke TRON | ✅ API-triggered agent runs | RW: automation-friendly. PD+TG: human-in-the-loop by design. |

### 5.7 Summary Scorecard

| Category | PD+TG | RW |
|:--|:--|:--|
| Core orchestration | 7/10 | 10/10 |
| Observability & feedback | 8/10 | 9/10 |
| User interaction (especially remote) | 9/10 | 6/10 |
| Quality & supervision | 8/10 | 9/10 |
| Setup & maintenance | 10/10 | 4/10 |
| Customization & transparency | 10/10 | 5/10 |
| Scalability (>5 agents) | 4/10 | 9/10 |
| **Overall (weighted for current needs)** | **8/10** | **7/10** |

**Conclusion:** PD+TG is the right architecture for now. It covers all stated requirements (observability, stall detection, bidirectional TG, active supervision, multi-agent, user interaction) with minimal infrastructure. The protocol is designed to transfer to a runtime wrapper if scale demands it later.

---

**References:**
- [Praktor](https://github.com/mtzanidakis/praktor) — Multi-agent Claude Code orchestrator with TG I/O
- [openclaw-multi-agent-kit](https://github.com/raulvidis/openclaw-multi-agent-kit) — 10-agent TG supergroup coordination
- [gru](https://github.com/zscole/gru) — Self-hosted agent orchestration via TG/Discord/Slack
- [claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) — Claude Code ↔ TG bridge
- [GoClaw](https://github.com/nextlevelbuilder/goclaw) — Multi-agent gateway with inter-agent delegation

---

**Last Updated:** 2026-03-13
