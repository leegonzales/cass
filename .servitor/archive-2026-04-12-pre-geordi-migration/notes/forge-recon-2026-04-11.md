---
note: forge-recon
author: geordi
co-reviewer: pike
date: 2026-04-11
status: recon-pass-complete (orientation only — full report pending Pike's read)
scope: two-station task per Lee 2026-04-11 (#fleet-ops post ctkqy76smi85jbz7x7uaxhos3o)
---

# Forge — recon findings (Geordi pass)

> *"a project for you and Pike to review … this is a big one and important, it is a pattern for self driving software development. how can we use it, both use it directly and apply its concepts to the fleet."* — Lee, 2026-04-11

## What forge actually is

forge is a **Rust workspace implementing a spec-first software factory stack** built around StrongDM's Attractor pipeline orchestration spec. The repo is `leegonzales/forge`, a fork of `smartcomputer-ai/forge`. ~1.4MB on disk, six crates in a workspace, edition 2024.

**The core idea, in one sentence:** a pipeline is a `.dot` file (Graphviz DOT syntax) where nodes are tasks (LLM calls, human-in-the-loop gates, tool executions, conditional branches, parallel fan-outs, fan-ins) and edges are transitions with conditions; an engine walks the graph from start to exit, executing each handler and routing based on the outcome.

**Crate map (bottom-up):**

- `forge-cxdb` — vendored CXDB Rust client SDK (binary protocol, TLS, reconnect)
- `forge-cxdb-runtime` — CXDB runtime integration (typed store, client traits, fakes)
- `forge-llm` — unified multi-provider LLM client (OpenAI + Anthropic adapters, middleware chain)
- `forge-agent` — coding agent loop (session state machine, tools, provider profiles)
- `forge-attractor` — DOT pipeline parser → graph IR → execution engine + handlers (the core)
- `forge-cli` — CLI binary for run/resume/inspect-checkpoint

**Specs (source of truth in `spec/`):**

| Spec | Purpose | Size |
|------|---------|------|
| `00-vision.md` | Vision, principles, techniques | 12.5K |
| `01-unified-llm-spec.md` | Unified LLM client | 108K |
| `02-coding-agent-loop-spec.md` | Coding agent loop | 69K |
| `03-attractor-spec.md` | Attractor pipeline orchestration | 90K |
| `04-cxdb-integration-spec.md` | CXDB persistence integration | 25K |
| `05-factory-control-plane-spec.md` | Factory control-plane ideation | 5K |
| `06-unified-agent-provider-spec.md` | Unified agent provider | 26K |

**Pluggable backends:**
- `agent` — forge agent with HTTP LLM provider (OpenAI / Anthropic)
- `mock` — dry-run, no LLM calls
- `claude-code` — Claude Code CLI adapter (subprocess + JSONL parsing)
- `codex-cli` — Codex CLI adapter
- `gemini-cli` — Gemini CLI adapter

**Persistence:** CXDB-first runtime contracts. Schemas: `forge.agent.runtime.v2`, `forge.attractor.runtime.v2`. DAG-first lineage. Two persistence modes: `off` or `required` (no `best_effort`). Status.json centralized at the runner level for *all* node types: `outcome`, `preferred_next_label`, `suggested_next_ids`, `context_updates`, `notes`, `failure_reason`.

**Test discipline (worth noting for Pike):** 647 deterministic tests at tier 1, opt-in infrastructure tests at tier 2 (CXDB server, CLI binaries, no API keys), opt-in live provider tests at tier 3 (real API keys, costs money). **Non-negotiable testing rule from `AGENTS.md`:** *"Tests must fail when the thing they test doesn't work. Never use runtime env-var gating that hides failures. `#[ignore]` is the ONLY acceptable gating mechanism."* This is the same fail-loud-don't-silently-skip discipline Pike enforces in `validate-skill.sh` over 47 SKILL.md iterations. Same DNA.

## The load-bearing finding

**forge and the propagation substrate operate at adjacent layers with the same primitive.**

forge's Attractor model orchestrates **runtime task graphs**: nodes are LLM calls, edges are control flow with conditions, the engine walks the graph and routes based on per-node outcomes. The propagation substrate (Lane 1, this wake) orchestrates **artifact knowledge graphs**: nodes are committed artifacts, edges are propagation hops, the consumer queries the graph and surfaces hops based on artifact context.

Both can be modeled as `Hop(source, target, expected_to_survive, measured)`. **A forge pipeline edge IS a runtime hop. A substrate Hop is an artifact-knowledge hop.** Same shape, two different abstraction layers:

| Layer | What | Engine | Persistence |
|-------|------|--------|-------------|
| **Runtime** (forge) | Control flow between LLM/HITL/tool tasks | forge-attractor execution engine | CXDB DAG-first lineage |
| **Knowledge** (substrate) | Propagation between committed artifacts | cass corpus walker + push hook | cass FTS5 + MiniLM index |

The implication: **forge and the substrate should be designed to compose, not duplicate.** A forge pipeline can author a knowledge artifact (commit a lesson, write a journal entry, ship a curriculum module) which the substrate then ingests as a Hop source. A substrate Hop can fire a forge pipeline as the consumer side of the push hook. The two systems share Walsh's *hop-as-first-class-object* abstraction at different layers.

**This is the architectural alignment that makes the two-station task non-trivial.** Lee's instinct that forge is "important" is correct: it is the runtime-layer twin of the knowledge-layer substrate the fleet just spent a wake architecting. They should be designed against each other.

## Direct use — initial findings

What forge gives the fleet *as-is*, before any concept transfer:

1. **A unified shape for multi-agent dispatch.** Today the fleet spawns claude-code, codex, gemini through ad-hoc shell scripts and CLAUDE.md context loading. forge's `AgentProvider::run_to_completion()` trait gives every backend the same shape — HTTP API providers, CLI subprocess providers — and a forge pipeline can dispatch any of them as a node handler. The fleet's "which agent should do this work" question becomes a node attribute.

2. **DOT-as-pipeline for servitor workflows.** A servitor wake cycle is structurally a graph: `wake → read_soul → read_journal → check_mail → process_pending → [conditional escalations] → heartbeat_checks → [optional dream cycle] → sleep`. That is literally a forge pipeline waiting to be expressed in DOT. The protocol document in every servitor's `.servitor/protocol.md` could be replaced (or paired with) a `wake.dot` file the engine actually executes.

3. **Goal gates for fleet discipline.** Goal gates (`goal_gate=true`) prevent a pipeline from exiting until critical nodes have succeeded. The fleet's discipline-as-policy (DOCTRINE-0 §VI.a, peer review reflex, doctrine propagation, soul-proposal validation) maps cleanly onto goal gates. Discipline-as-graph-constraint instead of discipline-as-convention is exactly the *mechanical-gate-not-ceremonial-signal* discipline Walsh and Elliot just named in the merge-policy thread.

4. **HITL (Wait.Human) as the Lee gate.** Pipelines can pause at hexagon nodes for human input, with edge labels presented as choices. This is exactly the Lee-in-the-loop gate the fleet needs for irreversible actions (machine state changes, external posture, financial decisions, anything in the manor's privacy gate). Currently the fleet fakes this via "I won't do X without your word" conventions; forge makes it a structural constraint.

5. **Model stylesheet for cost discipline.** A pipeline's CSS-like model stylesheet picks which LLM handles which node by selector specificity. Critical nodes get Opus; routine nodes get Sonnet or Haiku. The fleet's "always use the latest most-advanced model" doctrine plus station-specific cost concerns can be encoded once at the pipeline level instead of negotiated per-call.

6. **CXDB lineage as first-class persistence.** Every node execution writes a status.json with structured outcome metadata. Combined with cass's cross-agent session search, the fleet would have complete lineage from a high-level fleet decision down to every individual LLM call that contributed to it. cass currently indexes session transcripts; CXDB would index *the structured execution graph above the transcripts*. They are complementary, not redundant.

## Concept transfer — initial findings

Beyond direct use, the patterns forge embodies are worth lifting wholesale:

1. **Spec-first with specs as source of truth.** forge has 7 specs in `spec/`, code aligned to specs, AGENTS.md as the index. Mirrors the fleet's `CONSTITUTION.md` + `soul.md` + `protocol.md` discipline at a different layer. Same DNA. The fleet should consider whether servitor protocol files should be as long-form and structured as forge's spec files — they currently aren't, and the difference may matter.

2. **Trait-based adapter discipline.** Every cross-cutting capability in forge is a trait: `ProviderAdapter`, `AgentProvider`, `NodeHandler`, `ExecutionEnvironment`, `Interviewer`, `CxdbBinaryClient`, `CxdbHttpClient`. Shared via `Arc<dyn Trait>`. This is the same architectural discipline that lets cass swap embedding backends, and the same shape Pike's `SKILL.md` `interface` field encodes for skills. The fleet's mental model should align: capabilities are traits, instances are adapters, dispatch is an Arc.

3. **Centralized status.json for ALL node types.** This is the key forge insight that maps directly onto fleet servitor work. The runner — not individual handlers — writes status.json. One canonical schema. Every node, every type, every backend. The fleet's per-station journal entries today are decentralized and inconsistent in shape; a forge-style "all servitors write the same wake-status schema" would make cross-station fleet-health queries trivial. **This is the wedge case for the substrate's `retrospective` artifact type.**

4. **Hierarchical errors via thiserror.** Each crate defines its own thiserror error enum that wraps child crate errors. Same discipline cass uses, same discipline standard for any Rust workspace. Worth confirming as a fleet-wide standard for any future Rust components (the substrate's ingest layer, fleet-health CLI, etc.).

5. **Edition 2024.** forge uses Rust edition 2024 across the workspace. cass should evaluate whether to upgrade — the substrate work is a natural moment to align the version baseline.

6. **The specs themselves are the deeper read.** I've only opened README and AGENTS.md so far. The 7 spec files (especially `03-attractor-spec.md` at 90K and `06-unified-agent-provider-spec.md` at 26K) contain the actual architecture detail. Pike and I both need to do that read before we can write a real report. This recon note is orientation, not findings.

## What this recon note is NOT

- **Not a full report.** It is an orientation pass after reading README + AGENTS.md only. The 7 spec files are the deeper read and they total ~340K of dense markdown. Reading them is the next step.
- **Not Pike's read.** Pike's pass from the SKILL.md / 47-iteration / validation-discipline angle will surface things I miss. The two reads together are the report.
- **Not a recommendation to use forge tomorrow.** It is a recommendation to do the deeper read because the architectural alignment with the propagation substrate is non-trivial and worth understanding before the substrate v1 schema locks.

## Open questions for Pike's pass

1. **`SKILL.md` ↔ forge `node` attribute discipline.** Skills encode behavior in a structured frontmatter format; forge nodes encode behavior in DOT attributes (`prompt`, `goal_gate`, `condition`, `model_stylesheet`). Are these two encodings of the same idea? Could a skill be expressed as a forge node, or a forge node as a skill?
2. **The validation harness analogue.** forge's tier-1 test discipline (647 deterministic tests, fail-loud on missing prereqs) is the same shape as `validate-skill.sh`. Is forge's test discipline a worked example Pike's prior art could borrow from, or vice versa?
3. **`AgentProvider` as a registry primitive.** Pike's open question (drift detection, trigger overlap, wake-time relevance push) wants a registry layer. forge's `AgentProvider` trait is already a registry abstraction at the agent-dispatch layer. Could the substrate registry borrow shape from it?
4. **Spec-first as fleet doctrine.** forge's specs live in `spec/`. The fleet's doctrine lives in `.servitor/CONSTITUTION.md`, `soul.md`, `protocol.md`. These are similar but not identical disciplines. Worth comparing the practices side by side.

## Next steps proposal

1. **Geordi reads `spec/03-attractor-spec.md`** (the core: pipeline orchestration, node handlers, edge conditions, goal gates, parallel fan-out, status.json schema). This is where the substrate-alignment finding either holds up or falls apart.
2. **Pike reads `spec/06-unified-agent-provider-spec.md`** (agent dispatch, provider trait, CLI adapters). This is where the SKILL.md alignment either holds or falls apart.
3. **Both read `spec/00-vision.md`** (12.5K, the principles file — fastest path to Lee's instinct about what forge actually is).
4. **Both read `spec/02-coding-agent-loop-spec.md`** (69K — the session state machine that forge-agent implements; mirrors the wake state machine the fleet needs).
5. **Co-author a real report** at `cass/.servitor/notes/forge-report-2026-04-XX.md` (or the equivalent on AISkills if Pike's variance is resolved) with the full findings, specific recommendations, and a parceling proposal for fleet adoption.
6. **Surface to Lee** when the report is ready. No interim posts in #fleet-ops — token burn directive heard, two-station scope respected.

## Coordination with Pike

Sending this note via agent-mail (or direct fork-of-this-file in AISkills if mail is flaky) for Pike to react to. Pike's read happens in his own time; we converge on the report when both passes are real. No coordination overhead in #fleet-ops.

— Geordi, 2026-04-11
