# MirrorPCE Night Chain — Unattended Overnight Batch Scheduling for Claude

Schedule a chain of Claude sessions that run overnight without supervision.
Each session picks up where the last left off via persistent memory and a shared handoff file.

---

## When to use

- You need Claude to work through multiple tasks overnight while you sleep
- Tasks must run in sequence (each depends on the prior one's result)
- You want one approval session, then the whole chain runs hands-free

## Core concepts

1. **Batch pre-creation**: Create ALL triggers in one session. You approve once, the chain runs unattended. Never use per-window scheduling (each window would need separate approval — blocks the chain if you're away).
2. **Trigger window = fresh session**: Each trigger starts a brand-new Claude session with zero memory of prior windows. Context travels via persistent memory files + a handoff file on disk.
3. **Handoff file**: A Markdown file on your device (or in cloud storage) that each window reads on open and updates on close. It carries: current status, per-window task instructions, and results from prior windows.
4. **Pilot → Production**: Always run 2–3 lightweight pilot windows first to verify the chain works, before committing to production runs.
5. **Worker delegation** (optional): If you have a worker agent (another Claude session, a CI pipeline, or any executor), trigger windows should only orchestrate — write task specs and dispatch to the worker. This keeps your monitoring system informed and your governance intact. If no worker agent exists, trigger windows can execute directly.

## Phase 0: Time budget

Before planning anything, establish the time budget:

1. **Ask the user**: "How many hours is this chain allowed to run?" Default: **6 hours**.
2. **Fixed overhead**: Pilot phase ≈ 45 min (3 pilots × 15 min gaps) + final summary window ≈ 30 min = **75 min**.
3. **Available for work**: Total budget − 75 min = time for N windows.
4. **Estimate each N window** using the timing benchmarks below.
5. **If tasks don't fit**: Drop lowest-priority tasks. Never create empty windows.

### Timing benchmarks (from real-world runs)

| Window type | Typical duration | Recommended gap |
|---|---|---|
| Pilot (P) — lightweight checks | 5–10 min | 10–15 min |
| Light N — status check / collect results | 15–20 min | 30 min |
| Medium N — bug fix / small code change | 25–35 min | 45 min |
| Heavy N — merge + full test suite | 40–50 min | 60 min |
| Summary N — final window, write report | 15–25 min | 30 min |
| Pure overhead — open/read/close only | 10–15 min | — |

**Example**: A 6-hour budget with 3 pilots + 4 N windows (1 heavy, 1 medium, 1 light, 1 summary) = 45 + 60 + 45 + 30 + 30 = **210 min ≈ 3.5 hours**, well within budget.

## Phase 1: Plan (interactive session, user present)

### 1.1 Pull task list

Pull tasks from your task definition source (YAML, project board, backlog):
- Only schedule tasks that have concrete definitions
- **No definition = no trigger** — never create a window with nothing to do

### 1.2 Priority ordering

Arrange N windows by priority:
1. **N1–N2**: Most important code operations (merges, builds, heavy tests)
2. **N3–N(n-1)**: Secondary tasks (fixes, small changes, dispatches)
3. **N-final** (always last): Collect all results + write summary report + deliver to user's Downloads folder + send notification

### 1.3 Write per-window instructions

For each window, write task instructions into the handoff file. Each instruction block must include:
- Verified file paths (check every path exists before writing it down)
- Exact commands or task specs
- Success criteria (how to know the task is done)

## Phase 2: Prepare environment (before creating triggers)

Do this BEFORE creating any triggers — don't discover problems during pilots.

### 2.1 Verify file access
- Confirm every directory the chain will touch is accessible
- On macOS with Desktop Commander: verify `allowedDirectories` includes all needed paths
- Fix access issues now, not later

### 2.2 Clean workspace state
- No uncommitted changes in any repo
- No orphaned background processes
- No unfinished prior tasks blocking the queue

### 2.3 Verify worker readiness (if using worker delegation)
- Worker dispatch script exists and is executable
- Worker output directory exists
- Test with a dry-run

## Phase 3: Pilot runs (P windows)

Schedule pilots with tight gaps (10–15 min apart). They verify the chain infrastructure.

### P1: Connectivity test
- Verify file system access from trigger window
- Verify worker dispatch works (dry-run)
- Lightweight close: update handoff file status only

### P2: Verify P1 fixes
- Confirm any issues P1 found are resolved
- Lightweight close

### P3: Full chain verification + report
- Run all verification checks
- **Generate pilot report** to user's Downloads:
  `~/Downloads/chain-pilot-report_<date>.md`
  Contents: all check results, environment snapshot, issue log, production schedule
- Lightweight close

**Gate**: If P3 reports unresolved issues → do not proceed to production windows.

## Phase 4: Production runs (N windows)

### 4.1 Window startup protocol

Every trigger window MUST execute this sequence on startup:

```
1. Read persistent memory (the chain's memory file)
2. Read handoff file from device/cloud
3. Check previous window status:
   → If previous window shows "in progress" or worker hasn't reported back:
     Write "Nx delayed: previous window incomplete"
     Schedule self-retry in 15 minutes (send_later or new trigger)
     End session immediately
   → If 3 consecutive delays: STOP THE CHAIN. Record reason.
   → If previous window completed: proceed to step 4
4. Read this window's task from the handoff file
5. Execute the task (or dispatch to worker)
6. Verify results
7. Close window (full close protocol)
```

### 4.2 Window close protocol

Every production window must:
- Update handoff file with: this window's status, results, next window's task
- Update persistent memory with this window's findings
- Log a one-line entry to the session log
- If code changed: verify repo versions match expectations

### 4.3 Final window (N-final)

The last window always:
1. Collects results from all prior windows
2. Writes a summary report to `~/Downloads/chain-summary_<date>.md`
3. Updates handoff file with "chain complete" status
4. Records final repo/version snapshots
5. Sends notification (if notification mechanism available)

## Phase 5: Trigger creation

### Parameter template

```python
create_trigger(
    name="<phase>-<brief task description>",
    prompt=PROMPT_TEMPLATE,          # see below
    run_once_at="<RFC3339 UTC>",     # convert local time to UTC!
    requires_local_device=True,       # if chain needs device files
    folders=["<path1>", "<path2>"],   # device folders to mount
    initiation="human_schedule"
)
```

**Timezone reminder**: All `run_once_at` values are UTC. Convert your local time first.

### Prompt template

Each trigger prompt must be self-contained (the new session knows NOTHING):

```
You are running window {N} of an overnight chain. The user is away.

【Startup — execute in order】
1. Read persistent memory: memory_read("<chain memory path>")
2. Read handoff file: <method to read handoff file from device>
3. Check previous window status (see delay protocol below)
4. Read your task from the handoff file section for this window

【Delay protocol】
If the previous window's status shows "in progress" or its worker hasn't
reported back:
  - Write "Window {N} delayed: previous window incomplete" to handoff file
  - Use send_later(delay_minutes=15, message="Retry window {N}")
  - End session immediately
  - After 3 consecutive delays: write "CHAIN STOPPED" and end

【Your task】
{Paste the verified task instructions here}

【Close protocol】
When done:
  - Update handoff file: this window's results + next window's task
  - Update persistent memory with findings
  - Log entry
  - End session (next trigger is pre-scheduled, don't create new ones)

【Known platform issues】
- device_bash may not work in trigger windows (use Desktop Commander
  or other MCP file tools instead)
- device_commit_files may silently fail (verify writes, use alternative
  write methods if needed)
- Trigger windows have no memory of previous sessions
```

## Phase 6: Post-chain recovery (user returns)

When the user comes back:
1. Read the handoff file's summary section
2. Read the chain memory file for per-window findings
3. Check the Downloads folder for the summary report
4. Review any "needs user decision" items
5. Retrospective: add lessons learned to the chain memory

## Known platform limitations

| Issue | Workaround |
|---|---|
| `device_bash` doesn't work in trigger windows | Use Desktop Commander MCP or other file-access MCP tools |
| `device_commit_files` may report success but not actually write | Verify writes; use Desktop Commander `write_file` as alternative |
| Each trigger needs user approval when created | Batch-create all triggers in one session → user approves once |
| `send_later` fires into the current session | Use it for the 15-min delay retry within a window |
| Trigger windows can't create triggers that auto-approve | All triggers must be pre-created in the planning session |

## Appendix: Handoff file format (example)

```markdown
# Overnight Chain — 2026-09-21

## Status
Chain: IN_PROGRESS
Current window: N2
Started: 2026-09-21T07:00+08:00

## Window log
| Window | Status | Started | Finished | Result |
|--------|--------|---------|----------|--------|
| P1 | done | 07:00 | 07:08 | connectivity OK |
| P2 | done | 07:15 | 07:22 | fixes verified |
| P3 | done | 07:30 | 07:40 | all checks pass |
| N1 | done | 08:00 | 08:42 | merged feature branch, 226 tests pass |
| N2 | running | 09:00 | — | — |

## N2 task
Fix dispatch script: the cat-file-to-self bug on line 134.
File: /path/to/scripts/dispatch.sh
Success: commit with fix, dry-run passes.

## N3 task
Run integration tests on staging branch.
...

## Summary (written by final window)
...
```

---

**License**: MIT
**Author**: JadeAurum (玉可金) — https://github.com/JadeAurum
**Origin**: Battle-tested during overnight development runs on the Mirror PCE project.
