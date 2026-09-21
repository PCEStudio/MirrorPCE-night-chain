# MirrorPCE Night Chain

**Unattended overnight batch scheduling for Claude** — set up a chain of tasks, approve once, sleep, wake up to results.

> Built with Claude's latest capabilities (scheduled tasks, persistent memory, MCP tools — available since mid-2025) and battle-tested on the [Mirror PCE](https://github.com/JadeAurum) project. Works best with Opus or Sonnet class models.

---

## What it does

You have a list of tasks that need to run in sequence overnight. Each task depends on the previous one's result. You don't want to stay up babysitting Claude.

**Night Chain** lets you:
1. Plan all tasks in one interactive session
2. Create all scheduled triggers at once (one approval, done)
3. Walk away — the chain runs itself through the night
4. Wake up to a summary report in your Downloads folder

Each trigger window is a fresh Claude session. Context passes between windows via **persistent memory** and a **handoff file** on your device. If a previous task isn't finished yet, the next window automatically **delays 15 minutes and retries** — up to 3 times before stopping the chain safely.

## Why this exists

Claude's scheduled tasks are powerful, but they have a catch: **each trigger starts a brand-new session with zero memory**. If you schedule 5 tasks, window 3 has no idea what windows 1 and 2 did.

This skill solves that by giving Claude a complete protocol:
- **Handoff file** carries status, task instructions, and results between windows
- **Persistent memory** stores findings that survive across sessions
- **Delay protocol** prevents windows from stepping on each other
- **Pilot phase** catches environment issues before production runs
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
3. Run 3 pilot windows to verify everything works
4. Execute production windows in priority order
5. Write a summary report to `~/Downloads/`

You approve the triggers once, then you're free to leave.

## How it works

### The chain structure

```
[You + Claude plan together]
        ↓
   P1 → P2 → P3          ← Pilot: verify connectivity, file access, tools
        ↓
   N1 → N2 → ... → N-final   ← Production: real tasks, priority order
        ↓
   Summary report in ~/Downloads/
```

### The retry mechanism

Each production window checks the previous window's status on startup:

```
Previous window done? → Proceed with this window's task
Previous window still running? → Write "delayed" → Wait 15 min → Retry
3 consecutive delays? → STOP THE CHAIN (record why)
```

This prevents cascading failures. If a heavy task takes longer than expected, the chain waits patiently instead of crashing.

### Time budgeting

The skill includes timing benchmarks from real overnight runs:

| Window type | Typical duration | Recommended gap |
|---|---|---|
| Pilot (P) | 5–10 min | 10–15 min |
| Light task | 15–20 min | 30 min |
| Medium task | 25–35 min | 45 min |
| Heavy task | 40–50 min | 60 min |
| Summary | 15–25 min | 30 min |

A 6-hour budget comfortably fits 3 pilots + 4 production windows.

### Worker delegation (optional)

If you have a worker agent (another Claude session, a CI pipeline, or any executor), trigger windows can **orchestrate only** — they write task specs and dispatch to the worker. This is useful for:
- Keeping a monitoring system informed
- Maintaining governance and audit trails
- Separating "what to do" from "how to do it"

If no worker exists, trigger windows execute tasks directly.

## Known platform limitations

These are real issues we hit during overnight runs, with proven workarounds:

| Issue | Workaround |
|---|---|
| `device_bash` may not work in trigger windows | Use Desktop Commander or other MCP file tools |
| `device_commit_files` may silently fail | Verify writes; use alternative write methods |
| Each trigger needs approval when created | Batch-create all in one session → approve once |
| Trigger windows have no memory | Everything passes through handoff file + persistent memory |

## Requirements

- **Claude with scheduled tasks and persistent memory** — these features became available in mid-2025. Works best with **Opus** or **Sonnet** class models (the orchestration logic needs strong instruction-following in fresh sessions with no prior context).
- **Claude Pro, Team, or Enterprise** plan (scheduled tasks require a paid plan)
- **Persistent memory** enabled in your Claude settings
- **MCP file tools** for reading/writing your handoff file (Desktop Commander, Filesystem, or similar)
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
- **Don't skip pilots**: 15 minutes of pilot testing saves hours of debugging at 3 AM.
- **Check your timezone**: All `run_once_at` values are UTC. A common mistake is forgetting to convert.

## Contributing

Found a bug? Have a better workaround for a platform limitation? PRs welcome.

If you've used Night Chain for your own overnight workflows, I'd love to hear about it — open an issue and share your experience.

## ⭐ If this helped you

If Night Chain saved you a sleepless night, consider giving this repo a star! It helps others discover the skill and motivates continued development.

This skill is built on the latest Claude capabilities — scheduled tasks, persistent memory, and MCP tool integration — to let you hand off real, sequential work and walk away. Stars help us keep it updated as Claude evolves.

## License

MIT

## Author

**JadeAurum (玉可金)** — [github.com/JadeAurum](https://github.com/JadeAurum)

Built during overnight development runs on the Mirror PCE project — a personal cognitive exoskeleton that puts AI assistance in everyone's hands.
