# Branch closed — 2026-07-27

**Status: RETIRED. Do not merge. Do not build from this branch.**

## What this branch was

An experimental sync of upstream (`Dicklesworthstone/coding_agent_session_search`)
into Lee's fork, staged 2026-04-10 by Geordi and explicitly flagged at the time as
"unpushed — Lee reviews before any origin push" (see `leegonzales/geordi`
`proposals/2026-04-12-forge-fleet-integration.md` for the original context this
branch was mentioned in).

## What actually happened

The review never happened. A release binary was built from this branch and
installed as the fleet's `~/.local/bin/cass` without merging or reviewing it. That
binary ran in production for roughly 3.5 months (2026-04-10 → 2026-07-24) before a
95-day search-staleness incident led to discovering the branch's provenance. It
carries a reproducible out-of-memory bug rebuilding the derived FTS index at real
data volume (referenced in-code as "Bug #168" — a batch-size cap that turned out
insufficient), unrelated to the staleness incident itself but co-discovered during
its investigation.

## Decision (Lee, 2026-07-26, via fleetmail)

> ADOPT MAINLINE CASS. Deprecate our fleet port entirely. Rationale: the port
> existed to support the fleet monitor system; that system has since been moved
> OUT of cass's line of need, so the reason for carrying a divergent port is gone.
> No patching of the sync/upstream-2026-04-10 branch — neither option (a) nor (b)
> as framed; the fork itself is retired.

The fleet now runs `upstream/main` directly (currently pinned at tag `v0.6.22`),
rebuilt from a clean worktree, no fork-specific patches carried forward. Full
diagnostic and recovery trail: `leegonzales/geordi` journal, wakes #438–#446+
(2026-07-24 through 2026-07-27).

## Why this branch is preserved, not deleted

Per fleet doctrine (Constitution Article II — no destructive git operations
without explicit authorization), this branch ref is kept as the historical record
of what actually ran in production and why, not force-deleted. It should not be
built from or merged going forward.

— Geordi
