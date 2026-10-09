# MirrorPCE Night Chain — Unattended Overnight Batch Scheduling for Claude

**Version:** v2.1 (2026-10-09)

Schedule a chain of Claude sessions that run overnight without supervision.
All windows are created up front; each one reads a single **schedule file**, pushes every work lane one step forward, and records its status so the next window knows where to pick up.

Flow: time budget → write the schedule file → prepare the environment → create all triggers at once + schedule watch checks → P1 dry run → N windows advance lane by lane → watch duty → final-window report → recovery.

---

## When to use

- You need Claude to work through multiple tasks overnight while you are away
- You have several **independent work lanes** (different working directories / repos) that can advance in parallel, each with its own ordered steps
- You want one approval session, then the whole chain runs hands-free

## Core concepts

1. **Assign work at planning time**: What each window dispatches is written into the schedule file before the chain starts. Night windows write their own task specs and hand them to the worker; they skip the heavier daytime review gates.
2. **Batch pre-creation, staggered**: Create ALL triggers in one session, staggered in time. While the chain is running, never create, edit, or delete triggers (each change asks for a confirmation that nobody is there to give). Do not use `send_later` for retries.
3. **Trigger window = fresh session**: Each trigger starts a brand-new Claude session with zero memory of prior windows. Context travels through the schedule file (plus persistent memory, if you use it).
4. **Lanes run in parallel**: Split work into lanes by working directory. Each window pushes every lane one step; lanes run in parallel, steps inside a lane run in order. If a lane's previous step has not finished, this window skips that lane and still advances the others.
5. **Pending decisions don't stop the chain**: Anything that needs the user's decision goes into the schedule file's `## Pending decisions` section (lane | issue | evidence | orchestrator's view | whether the default was applied). The chain keeps going; the final window puts the list at the top of its report. Implementation-level trade-offs the orchestrator decides itself and writes down its reasoning.
6. **Mid-run additions use a timer, not a trigger**: To add a step or an extra check while the chain runs, use a reminder/timer tool that posts into the on-duty main session (no confirmation needed). A timer cannot start a new session — only the pre-created triggers can.
7. **Worker on-the-spot adjustments** (if you delegate to a worker agent): The worker may make small fixes within the task's own write scope that don't touch design or acceptance criteria — add a timeout to a stuck test, fix a mistyped path or parameter, re-run a measurement, add a missing closing line — and must list each one in a "On-the-spot adjustments" section of its report. Anything beyond that (design change, another write scope, changed acceptance criteria) becomes a stop point. State this rule in every task spec.
8. **Window gap = estimated finish time + 30 minutes** (see Phase 0).
9. **Only schedule well-defined work**: Paths, branches, and reference documents are all verified before they go into the schedule file. Undefined work is not scheduled — better to schedule less than to have windows spin idle.

## Phase 0: Time budget

1. **Ask the user**: "How many hours is this chain allowed to run?" Default: **6–7 hours**.
2. **Gap per window** = the longest single task in that window + 30 min; typically 70–90 min.
3. **P1 only verifies** (dispatches nothing); N1 can follow 10–30 min later.
4. **If tasks don't fit**: Drop lowest-priority tasks. Never create empty windows.

### Timing benchmarks (from real-world runs)

| Item | Typical duration |
|---|---|
| Window itself (open → dispatch → close) | Middle windows 6–13 min; ~15 min when the orchestrator writes a spec document |
| Build task (worker writes code) | 9–40 min |
| External review / exploration draft | 12–20 min |
| On-device probe | 12–45 min |
| Black-box review pass | 5–10 min |
| Full test suite | ~14 min |

**Example**: P1 + N1–N4 + N5 (final) ≈ **6.5 hours** in one night.

## Phase 1: Plan — the schedule file (interactive session, user present)

The **schedule file** is the single source of truth (e.g. `<notes-dir>/night/night_YYYYMMDD.md`). Trigger prompts only say "open → read the schedule file → follow the general rules and this window's section"; to change the plan, edit the file, never the triggers.

Structure:
1. **General rules** — apply to every window (startup check, closing order, worker rules)
2. **Lane table** — lane / working directory / step order
3. **Per-window sections** — for each task: reference documents, decisions already made, acceptance criteria, time budget
4. **Trigger list** — name and fire time of each window
5. **`## Pending decisions`**
6. **`## Window status`** — one line per event (`N2 started HH:MM`, `N2 done HH:MM`, `N3 delayed 1`)

Each task entry must include verified file paths, the exact task spec, and success criteria.

## Phase 2: Prepare environment (before creating triggers, main session)

- Verify every lane's working directory and branch; confirm the file-access MCP tool can reach all of them (fix allowed-directory settings now, not during the night).
- No unfinished in-flight tasks in any repo; devices online if a lane needs real hardware.
- If you use a worker agent, make sure its identity/instructions file is current (model, role, "this file overrides other instruction files in the working directory", one-shot session rules, on-the-spot adjustment rules).
- **When you create the triggers, also schedule 2–3 watch checks** between windows (e.g. after N1, before N3, before N5), each stating what to check.

## Phase 3: Window open and close (unattended, lighter than daytime)

### 3.1 Open

1. Load the file/orchestration tools and read the schedule file. If no tool can reach it, the bridge is down — end the session.
2. **Check window status** (completion is judged from the status log, not from the clock):
   - Previous window shows `done` → append `<this window> started HH:MM` and work.
   - Previous window shows only `started` → wait in-window (e.g. two `sleep 450` = 15 min), check again, record `delayed N`. After 4 delays, treat the previous window as lost and proceed — but do not touch its in-flight tasks.
   - Previous window has no `started` at all → it never ran; do its section's work as well.
3. Read only what this window needs: unfinished handoff items, new worker reports, whether each lane's sessions are still alive, any decisions the on-duty session wrote into the schedule file. **Only P1 and the final window run the full open checklist**; middle windows stay light.

### 3.2 Close — middle windows

1. Append a "Window result:" block to this window's section, one line per lane (dispatched / collected / stuck).
2. Write what the next window must pick up into the handoff notes; one line each to the dev log and progress board, if you keep them.
3. **Last**, write `<this window> done HH:MM` to the window status. This line is the next window's only signal — forgetting it makes the next window wait for nothing.

### 3.3 Close — final window

Full close checklist, plus a report at `~/Downloads/night-report_YYYYMMDD.md`:
- **Top**: "Pending decisions for the user"
- **Then**: one line per lane per task — task ID | result | commit | stop point | next step
- **Last**: problems with the chain itself

## Phase 4: Task-spec rules (write these into every task)

- A first line naming the task type (probe / exploration / build / fix / continue / review), the repo and branch on one line, and a time budget copied from real numbers of similar tasks.
- Read-only tasks say so in the first paragraph ("read-only, change no files").
- The report file name matches the task label; the last line is the completion-notification instruction.
- State the worker's on-the-spot adjustment scope and require an "On-the-spot adjustments" report section.
- **Long commands (full test suite, device measurement, builds) must say "run to completion in this session / poll until finished before closing"**. A worker that answers "it's running in the background, I'll report when it lands" ends its session and the report is lost.
- If a task ends unfinished but its outputs are all present, dispatch a **continue** task that only writes the report — don't re-run the tests.

## Phase 5: Watch duty (main session)

- After all triggers are created, the main session stays on duty. When a watch-check reminder fires: look at window status, pending decisions, and stuck tasks in each lane. If something is wrong, **only edit the schedule file** (the next window follows it); never touch the triggers.
- When a worker completion notice arrives, read only its conclusion and stop points; write any decision into the relevant window section. Don't dispatch tasks for a lane yourself (except a continue task to finish a report). If you need another look, add a timer, not a trigger.
- The on-duty session has a tool-call budget too; once it is used up, reply only "handed to the next window" and stop opening files.

## Phase 6: Trigger creation

### Parameter template

```python
create_trigger(
    name="night-MMDD-Nx",
    prompt=PROMPT_TEMPLATE,          # see below
    run_once_at="<RFC3339 UTC>",     # convert local time to UTC!
    requires_local_device=True,       # if the chain needs device files
    folders=["<lane dir 1>", "<lane dir 2>", "<notes dir>"],
    initiation="human_request"
)
```

**Timezone reminder**: All `run_once_at` values are UTC. Convert your local time first.

### Prompt template

Keep the prompt short and stable — the schedule file carries the details:

```
You are the orchestrator, woken by the overnight schedule. This window = "{Nx}".
The user is away and nobody can confirm anything.
1. Load the file/orchestration tools.
2. Read the schedule file at <path>. Follow ALL of its "General rules"
   (check window status first) and the "{Nx}" section. Do nothing that
   the file does not say.
3. If no tool can reach the file, the bridge is down — end the session.
Do not create, edit, or delete scheduled tasks; do not use send_later.
Record pending decisions as the general rules say and keep going.
Close the window by writing "{Nx} done HH:MM" to the window status — last.
```

## Phase 7: Recovery (user returns)

1. Read the report's "Pending decisions" and decide them one at a time.
2. Pick up each lane's "next step" from the report during the day (merge to main, second review round, etc.).
3. Retrospective: write lessons into this skill's lessons table or the worker's instructions file — **a gate is more reliable than a memory**.

## Customize for your setup

This skill is the generic core. You can extend it with your own:

- Lessons table — add rows from your own runs, each with a gate.
- Dispatch script — route tasks to your own workers.
- Notification channel — alert yourself on finish or stall.
- Window counter or budget control — cap calls or spend per window.
- Schedule-file template — your preferred starting format.

## Lessons learned

| # | Lesson | Defense |
|---|---|---|
| 1 | `device_bash` doesn't work in trigger windows | Use a file-access MCP tool with native device paths |
| 2 | Paths written from memory are wrong | Verify every path before it enters the schedule file |
| 3 | A worker that puts a long command in the background ends its session and loses the report | "Run to completion in this session" rule in every task spec + worker instructions |
| 4 | Instruction files in the working directory (e.g. `AGENTS.md`, `CLAUDE.md`) get auto-loaded by the worker and conflict with its role | Worker instructions file states "this file overrides" |
| 5 | Retrieval tools may skip rows of long tables | Locate first, then read just those lines |
| 6 | Running the full open checklist in every middle window is too heavy | Light open for middle windows; full open only in P1 and the final window |
| 7 | The on-duty session runs out of tool calls | Past the budget, reply one line and stop |
| 8 | Adding a trigger mid-run asks for a confirmation nobody can give | Mid-run additions use a timer |
| 9 | Old approach: `send_later` retries and per-window scheduling | Retired: delay with in-window sleep; create all triggers at once |
| 10 | `device_commit_files` may report success but not write | Verify writes; use the MCP tool's write function instead |

## Appendix: Schedule file format (example)

```markdown
# Night schedule — 2026-10-08

## General rules
- Startup: check window status (see skill Phase 3.1)
- Every task spec carries the on-the-spot adjustment rule
- Close: window result → handoff → "Nx done HH:MM" last

## Lanes
| Lane | Working dir | Steps |
|---|---|---|
| A | <repo-a> (branch feature-x) | A1 build → A2 review → A3 fix |
| B | <repo-b> | B1 probe → B2 build |

## P1
Verify only: tools reachable, lane dirs readable, worker dry-run.

## N1
- A1 build: refs <spec doc>; acceptance: tests pass; budget 40 min
- B1 probe: read-only; budget 20 min
Window result:
- A: dispatched A1
- B: collected B1, stop point recorded under Pending decisions

## Pending decisions
| Lane | Issue | Evidence | View | Default applied? |
|---|---|---|---|---|
| B | API shape for X | report B1 §3 | prefer option 2 | yes |

## Window status
P1 started 22:00
P1 done 22:12
N1 started 22:30
N1 done 22:41
```

---

**License**: MIT
**Author**: PCEStudio — https://github.com/PCEStudio
**Origin**: Battle-tested during overnight development runs on the Mirror PCE project.
