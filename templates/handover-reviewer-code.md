# Reviewer Handover

Scope definition file for the Code Reviewer agent.
Written by: TRON (at session start, before spawning the reviewer).
Read by: Reviewer (at session start).

This file is overwritten every session by TRON. Do not manually edit.

---

## IF YOU'RE THE REVIEWER:

1. Read the scope below the `=============` delimiter.
2. Use the commit range and focus areas defined there — do not expand scope beyond it.
3. Execute your review procedure. Committed state only — never read working tree files.
4. Return your findings following the format in your agent doc.
5. Communicate status via `meta/skills/skill-tg-comms.md` protocol.

## IF YOU'RE TRON:

Overwrite everything below the `=============` delimiter with the current session's review scope.

=============

**If there's nothing below the delimiter:** TRON has not yet written the scope for this session. Do not proceed — wait for TRON to populate this file.

## Review Scope

- **Commits to review:** {git hash range — from last review to HEAD}
- **Since:** {timestamp from last review log filename: YYMMDD-HHMM}
- **Repos in scope:** {list of repos with commits in range}
- **Focus areas:** {specific concerns from engineer's session — or "standard review"}
- **Last review log:** {path to most recent review log}

## Instructions

- Committed state only (git log range above). Do not read working tree files.
- All findings must be fixed by the engineer. Do not defer without explicit user approval.
- If a finding poses significant risk, downtime, or cost to fix: flag it as an escalation item — the user decides.
