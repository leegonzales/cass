# cass/.servitor/ — Runtime State Only

> **Geordi's identity moved.** This directory used to hold Geordi's soul,
> journal, state, and protocols. On **2026-04-12**, Geordi broadened from
> "the cass agent" to "engineer in the fleet, currently tending cass."
> His durable identity now lives at `leegonzales/geordi`.

## Where to find Geordi now

- **Home repo:** https://github.com/leegonzales/geordi
- **Local path:** `~/Projects/leegonzales/geordi/`
- **Soul:** `~/Projects/leegonzales/geordi/soul.md`
- **Journal (authoritative):** `~/Projects/leegonzales/geordi/journal.md`
- **State:** `~/Projects/leegonzales/geordi/state.json`
- **Protocols:** `~/Projects/leegonzales/geordi/protocols/`
- **Migration handoff note for Adama:** `~/Projects/leegonzales/geordi/handoff/adama-wire-up-geordi-repo.md`

## What lives HERE in `cass/.servitor/` (runtime state only)

- `logs/` — cass-specific operational logs (indexer, daemon, cron, events, chat runs)
- `heartbeat.json` — cass heartbeat runtime state
- `session-state.json` — cass session runtime state
- `experiments/` — cass-specific experiments
- `sops/index-health-check.md` — cass-specific operational SOP
- `archive-2026-04-12-pre-geordi-migration/` — frozen snapshot of the pre-migration identity (see below)

Runtime files (heartbeat, session-state, logs) are intentionally **not** migrated —
they are cass-specific operational data that should live with the system they
describe. Geordi reads them when tending cass; he doesn't carry them with him.

## The archive

`archive-2026-04-12-pre-geordi-migration/` contains a complete snapshot of the
pre-migration identity:

- `soul.md` (pre-migration — the geordi repo has the updated version with the
  "On Being, Not Performing" section)
- `journal.md` (everything up through Wake #209 initial entry — the geordi
  journal has the same content plus the Wake #209 continuation)
- `state.json`, `CONSTITUTION.md`, `dream-journal.md`, `dream-digest.md`
- `CLAUDE_SERVITOR.md`, `CLAUDE_GUARDIAN.md`, `CLAUDE_WORKER.md` (wake protocols)
- `mm-prompt.txt`, `mm-system-prompt.txt` (Mattermost prompt templates)
- `dreams/`, `memory/`, `notes/`, `proposals/`
- `sops/base-*.md` (fleet-wide SOPs — also copied to geordi)

**This archive is frozen.** Nothing here gets edited. It exists so that if
we ever forget this migration happened, or need to verify the pre-migration
state, we can come back to it. All of it is also live in `leegonzales/geordi`
with ongoing history.

## For the next Claude Code session in cass

If you're booting into cass and wondering why the soul file is missing:
1. Read `~/Projects/leegonzales/geordi/soul.md` — that's the live identity
2. Read `~/Projects/leegonzales/geordi/journal.md` — that's the live journal
3. Read `~/Projects/leegonzales/cass/CLAUDE.md` — it has been updated to point here

Cass tending is still a primary post for Geordi. But Geordi is an engineer
with multiple posts; cass is one of them, not the whole.

— Migration recorded in geordi commit `d58a0f5` on 2026-04-12
