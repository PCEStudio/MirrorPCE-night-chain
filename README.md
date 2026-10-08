# MirrorPCE Night Chain

**Unattended overnight batch scheduling for Claude** — set up a chain of tasks, approve once, sleep, wake up to results.

> Built with Claude's latest capabilities (scheduled tasks, persistent memory, MCP tools — available since mid-2025) and battle-tested on the Mirror PCE project. Works best with strong frontier-class models.

---

## What it does

You have several streams of work that need to move forward overnight — some steps depend on earlier ones, but different repos or working directories can progress side by side. You don't want to stay up babysitting Claude.

**Night Chain** lets you:
1. Plan all tasks in one interactive session and write them into a single **schedule file**
2. Create all scheduled triggers at once, staggered (one approval, done)
3. Walk away — the chain runs itself through the night
4. Wake up to a summary report in your Downloads folder

Each trigger window is a fresh Claude session. Context passes between windows via the **schedule file** on your device (plus persistent memory, if you use it). Each window pushes every **work lane** one step forward. If the previous window hasn't marked itself done, the next window **waits inside the session** (15 minutes at a time, up to 4 times) and then carries on without touching the unfinished work.

## Why this exists

Claude's scheduled tasks are powerful, but they have a catch: **each trigger starts a brand-new session with zero memory**. If you schedule 5 tasks, window 3 has no idea what windows 1 and 2 did.

This skill solves that by giving Claude a complete protocol:
- **Schedule file** is the single source of truth: lanes, per-window tasks, pending decisions, and a window-status log
- **Parallel lanes** keep independent work moving even when one lane is stuck
- **Window-status check** prevents windows from stepping on each other
- **Pending decisions** are logged instead of stopping the chain, and land at the top of the morning report
- **Pilot window** catches environment issues before production runs
- **Time budgeting** ensures your tasks actually fit in the hours you have

No more waking up to find the chain broke at window 2 because of a file access issue that could have been caught in a 5-minute pilot.

## Quick start

### 1. Install the skill

Copy `SKILL.md` to your Claude skills directory:

```
~/.claude/skills/MirrorPCE-night-chain/SKILL.md
```

Or place it in your project's `.claude/skills/` directory for project-scoped use.

### 2. Invoke

Tell Claude:

> "I need to run these tasks overnight: [your task list]. Use the night chain skill."

Or use the slash command if configured:

> `/MirrorPCE-night-chain`

### 3. What happens next

Claude will:
1. Ask how many hours the chain is allowed to run (default: 6 hours)
2. Plan the window schedule with time estimates
3. Write the schedule file (lanes, per-window tasks) with verified paths
4. Create all triggers at once, plus 2–3 watch-check reminders between windows
5. Run one pilot window (P1) that only verifies the setup
6. Run production windows, advancing every lane one step per window
7. Write a report to `~/Downloads/` with pending decisions at the top

You approve the triggers once, then you're free to leave. The planning session can stay open "on duty" and adjust the schedule file if a watch check finds a problem — it never edits the triggers.

## How it works

### The chain structure

```
[You + Claude plan together]
        ↓
   P1                     ← Pilot: verify only, dispatch nothing
        ↓
   N1 → N2 → ... → N-final   ← Production: each window advances every lane
        ↓                        (lane A, lane B, ... run in parallel)
   Report in ~/Downloads/ (pending decisions on top)
```

### The window-status check

Each window decides from the status log in the schedule file, not from the clock:

```
Previous window marked "done"?         → Proceed
Previous window only marked "started"? → Wait 15 min in-session, record "delayed N", check again
4 delays?                              → Treat previous window as lost, proceed (leave its in-flight work alone)
Previous window never started?         → Do its work as well
```

Each window writes its own "done" line **last** — it is the next window's only signal. Inside a window, a lane whose previous step hasn't finished is skipped while the other lanes advance.

### Time budgeting

The skill includes timing benchmarks from real overnight runs. The gap between windows is **the longest task in that window + 30 minutes** — typically 70–90 minutes.

| Item | Typical duration |
|---|---|
| Window itself (open → dispatch → close) | 6–15 min |
| Build task | 9–40 min |
| Review / exploration draft | 12–20 min |
| On-device probe | 12–45 min |
| Full test suite | ~14 min |

A default 6–7 hour budget fits P1 + N1–N5 (about 6.5 hours).

### Worker delegation (optional)

If you have a worker agent (another Claude session, a CI pipeline, or any executor), trigger windows can **orchestrate only** — they write task specs and dispatch to the worker. This is useful for:
- Keeping a monitoring system informed
- Maintaining governance and audit trails
- Separating "what to do" from "how to do it"

Workers may make small on-the-spot fixes within the task's own scope (a timeout for a stuck test, a mistyped path, a re-run) and list them in their report; anything touching design or acceptance criteria becomes a stop point. Long commands must run to completion inside the worker's session, or the report gets lost.

If no worker exists, trigger windows execute tasks directly.

## Known platform limitations

These are real issues we hit during overnight runs, with proven workarounds:

| Issue | Workaround |
|---|---|
| `device_bash` may not work in trigger windows | Use Desktop Commander or other MCP file tools |
| `device_commit_files` may silently fail | Verify writes; use alternative write methods |
| Each trigger needs approval when created | Batch-create all in one session → approve once; never add or edit triggers mid-run |
| Need an extra step mid-run | Use a timer/reminder into the on-duty session, not a new trigger |
| Trigger windows have no memory | Everything passes through the schedule file (+ persistent memory) |

## Requirements

- **Claude with scheduled tasks and persistent memory** — these features became available in mid-2025. Works best with strong frontier-class models (the orchestration logic needs strong instruction-following in fresh sessions with no prior context).
- **Claude Pro, Team, or Enterprise** plan (scheduled tasks require a paid plan)
- **Persistent memory** enabled in your Claude settings
- **MCP file tools** for reading/writing your schedule file (Desktop Commander, Filesystem, or similar)
- A device running the Claude desktop app (for `requires_local_device` triggers) — or adapt the handoff to cloud storage

## File structure

```
MirrorPCE-night-chain/
├── SKILL.md       # The skill file — this is what Claude reads
└── README.md      # You are here
```

## Tips

- **Start small**: Your first chain should be 2–3 simple tasks. Add complexity once you trust the flow.
- **Verify paths**: The #1 cause of chain failures is file paths that don't exist. The skill enforces path verification before trigger creation.
- **Don't skip the pilot**: 15 minutes of pilot testing saves hours of debugging at 3 AM.
- **Log decisions, don't wait**: Anything that needs you goes into "Pending decisions"; the chain keeps going.
- **Check your timezone**: All `run_once_at` values are UTC. A common mistake is forgetting to convert.

## Contributing

Found a bug? Have a better workaround for a platform limitation? PRs welcome.

If you've used Night Chain for your own overnight workflows, I'd love to hear about it — open an issue and share your experience.

## ⭐ If this helped you

If Night Chain saved you a sleepless night, consider giving this repo a star! It helps others discover the skill and motivates continued development.

This skill is built on the latest Claude capabilities — scheduled tasks, persistent memory, and MCP tool integration — to let you hand off real, sequential work and walk away. Stars help us keep it updated as Claude evolves.

## Changelog

- **2026-10-08 — v2 alignment**: single schedule file as source of truth; parallel work lanes; all triggers created at once and never changed mid-run (timers for mid-run additions); completion judged from the window-status log with in-session waiting; pending-decisions log; watch checks and on-duty session; worker on-the-spot adjustments; single pilot window; updated timing benchmarks and lessons.
- **Initial release**: sequential chain with handoff file, 3 pilots, and `send_later` retries.

## License

MIT

## Author

**PCEStudio** — [github.com/PCEStudio](https://github.com/PCEStudio)

Built during overnight development runs on the Mirror PCE project — a personal cognitive exoskeleton that puts AI assistance in everyone's hands.
