# Agent Society — Architecture

> Living document. Sections marked **[OPEN — spike]** are finalized during the Day 1–2 society decision spike; sections marked **[OPEN — build]** firm up as the component is built. See [ROADMAP.md](./ROADMAP.md) for phasing.

## 1. Overview

Agent Society is a multi-agent collaboration system with a game-like 2D front end. A user configures a society of AI agents (an organization or scenario), gives it a complex goal, and watches the agents decompose the task, bid for roles, negotiate conflicts, and produce a real artifact — with metrics compared against a single-agent baseline doing the same job.

### Design principles

1. **Event-sourced core.** Every meaningful occurrence (message, decision, bid, conflict, artifact) is an immutable event on a bus. The UI, metrics, and replay are all just consumers of the same stream.
2. **The 2D world is a renderer, not a simulation.** Agent sprites move because orchestration events fire (a negotiation event sends participants to the meeting table), never the other way around. The game layer stays thin and swappable.
3. **Hybrid agent core.** Qwen-Agent SDK owns each agent's inner loop (DashScope calls, tool-calling, custom skills, MCP). The *society layer* — decomposition, role assignment, negotiation, recovery, metrics — is custom. That layer is the project's technical contribution.
4. **Societies are data, not code.** A society is a scenario template (map, roles, character sheets, goal framing) loaded at runtime. The first society is a placeholder; richer ones plug in without engine changes.

## 2. System diagram

```
┌────────────────────────── Frontend (React + PixiJS/Phaser) ─────────────────────────┐
│  Society Builder      2D World View        Task Board /         Metrics Dashboard   │
│  (roles, goal)        (sprites, bubbles)   Negotiation Viewer   (vs baseline)       │
└───────────────────────────────▲──────────────────────────────────────▲──────────────┘
                        WebSocket (event stream)                REST (config, runs)
┌───────────────────────────────┴──────────────────────────────────────┴──────────────┐
│                              Backend (Python / FastAPI)                              │
│                                                                                      │
│  SOCIETY LAYER (custom)                                                              │
│  ┌──────────────┐  ┌────────────────┐  ┌──────────────────┐  ┌────────────────────┐ │
│  │ Orchestrator │  │ Task Decomposer│  │ Role Assignment  │  │ Negotiation &      │ │
│  │ + Event Bus  │  │ (planner → DAG)│  │ (contract-net)   │  │ Conflict Resolution│ │
│  └──────┬───────┘  └────────────────┘  └──────────────────┘  └────────────────────┘ │
│  ┌──────┴───────┐  ┌────────────────┐  ┌──────────────────┐  ┌────────────────────┐ │
│  │ Blackboard   │  │ Metrics Engine │  │ Baseline Runner  │  │ Replay Engine      │ │
│  │ (shared mem) │  │                │  │ (single agent)   │  │ (event log)        │ │
│  └──────────────┘  └────────────────┘  └──────────────────┘  └────────────────────┘ │
│                                                                                      │
│  AGENT RUNTIME (Qwen-Agent SDK)                                                      │
│  per-agent loop · qwen-max/plus/turbo via DashScope · custom skills · MCP tools      │
│  (web search, code execution, filesystem)                                            │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

## 3. Components

### 3.1 Orchestrator + Event Bus

The orchestrator drives a run: load scenario → spawn agents → decomposition → assignment → execution → verification → artifact. It owns the run lifecycle state machine. The event bus is in-process (asyncio queues) with two sinks: the WebSocket broadcaster and the append-only run log (JSONL) that powers replay and metrics.

- Interface: `publish(event)`, `subscribe(filter) -> stream`
- Depends on: nothing (everything else depends on it)
- **[OPEN — build]** persistence format and run-resume semantics.

### 3.2 Task Decomposer

A planner agent (qwen-max) turns the user goal into a task DAG: nodes carry description, required capabilities, dependencies, and acceptance criteria. The DAG is validated structurally (acyclic, capabilities exist in the society) before execution; invalid plans are re-prompted with the validation errors.

### 3.3 Role Assignment (contract-net)

For each ready task, the orchestrator issues a call-for-proposals. Worker agents bid with a structured proposal (suitability rationale, capability match score, current load). A pure-Python scoring function picks the winner — deterministic and unit-testable; the LLM contributes the bids, not the selection. Fallback (M1 / scope-cut mode): round-robin.

### 3.4 Negotiation & Conflict Resolution

Two conflict classes, two mechanisms:

- **Disagreements** (two agents produce contradictory outputs / reject each other's work): structured debate — each side states position + evidence in ≤2 rounds, then an arbiter agent rules and the ruling is published as a `conflict_resolved` event with rationale.
- **Resource/execution conflicts** (both need the same artifact lock, dependency stalls, agent timeout): handled by policy, not debate — priority rules, lock queue, and timeout → task reassignment via a new call-for-proposals.

All conflict state transitions are a pure state machine (unit-tested with mocked agent responses).

### 3.5 Blackboard (shared memory)

Run-scoped store for society state and artifacts: task DAG status, produced artifacts, agent profiles, locks. Backed by an in-memory store with JSON snapshots to disk. Agents read/write through tools, never directly — every write emits an event.

### 3.6 Agent Runtime

Each character is a Qwen-Agent assistant instance built from its **character sheet**: role, persona prompt, allowed tools/skills, model tier (qwen-max for planner/arbiter, qwen-plus/turbo for workers). Custom skills and MCP servers (web search, code execution, filesystem) are registered per role. **[OPEN — spike]** the character roster and sheets.

### 3.7 Metrics Engine + Baseline Runner

Shared instrumentation wraps every model call and task transition: tokens, cost, wall-clock, retries, task outcomes. The baseline runner executes the same goal with one agent holding all tools and the same instrumentation — comparisons are apples-to-apples by construction. Quality scoring: LLM-judge rubric + human check (see ROADMAP eval design).

### 3.8 Replay Engine

Replays a recorded run log through the same WebSocket channel at adjustable speed. The frontend cannot distinguish live from replay. Primary purpose: deterministic demos; secondary: debugging.

### 3.9 Frontend

- **2D World View** (PixiJS or Phaser — **[OPEN — build]** pick during Phase 0 scaffold): pre-made tilemap, agent sprites, movement + speech bubbles driven by events, conversation log panel.
- **Society Builder**: scenario template picker, character/role editor, goal input. Scope-cut fallback: JSON config upload.
- **Task Board / Negotiation Viewer**: DAG status and live conflict threads.
- **Metrics Dashboard**: society vs baseline charts per run.

## 4. Event schema (the contract)

The event stream is the single integration contract between backend and frontend, frozen at v1 at the end of Phase 0. Envelope:

```json
{
  "v": 1,
  "run_id": "r_2026...",
  "seq": 142,
  "ts": "2026-06-14T10:32:05Z",
  "type": "bid_placed",
  "agent_id": "eng_1",
  "payload": { "task_id": "t_3", "score": 0.82, "rationale": "..." }
}
```

Draft v1 event types — **[OPEN — build]** payloads to be specified in `schemas/events.py` (Pydantic, exported to TS types):

`run_started` · `agent_spawned` · `message_sent` · `task_decomposed` · `task_ready` · `cfp_issued` · `bid_placed` · `task_assigned` · `task_started` · `task_completed` · `task_failed` · `conflict_opened` · `debate_turn` · `conflict_resolved` · `lock_acquired` · `lock_released` · `artifact_produced` · `agent_moved` · `metrics_tick` · `run_completed`

`agent_moved` is emitted by a backend *stage-direction* mapper that translates orchestration events into world positions (negotiation → meeting table, task work → workstation), keeping all world logic server-side.

## 5. Data flow — lifecycle of a goal

1. User configures society in Builder → `POST /runs` with scenario + goal.
2. Orchestrator spawns agents (`agent_spawned`…), planner decomposes goal (`task_decomposed`).
3. For each ready task: `cfp_issued` → `bid_placed`×N → `task_assigned` → assignee executes with tools → `task_completed` (or `task_failed` → reassignment; or `conflict_opened` → debate → `conflict_resolved`).
4. Artifacts accumulate on the blackboard (`artifact_produced`); dependent tasks unblock.
5. `run_completed` with final artifact + metrics summary; dashboard compares against baseline runs of the same goal.

## 6. Error handling & recovery

- **Model-call layer:** retries with backoff, request queueing under rate limits, per-run token/cost budget caps.
- **Agent layer:** per-task timeout → task returned to pool and re-auctioned; malformed structured output → bounded re-prompt with validation errors.
- **Run layer:** any agent crash is an event, never an exception escaping the orchestrator; failure injection is a first-class demo feature (kill an agent mid-task, watch reassignment).
- **Frontend:** WebSocket reconnect resumes from last `seq` (event log makes this free).

## 7. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Models | Qwen (qwen-max / plus / turbo) via DashScope | Hackathon requirement; tier-mixing controls cost |
| Agent runtime | Qwen-Agent SDK | Native DashScope + tool/skill/MCP plumbing for free |
| Backend | Python 3.12, FastAPI, asyncio, Pydantic | Team strength; Pydantic schemas double as the event contract |
| Tools | MCP servers: web search, code execution, filesystem | Judging criterion (MCP integrations) |
| Frontend | React + Vite + TypeScript; PixiJS or Phaser | Tight scope for a Python-heavy team |
| Transport | WebSocket (events), REST (config/runs) | |
| Persistence | JSONL run logs + JSON snapshots | No DB needed at this scale; replay for free |

## 8. Repository layout (proposed)

```
backend/
  app/            # FastAPI app, REST + WebSocket
  society/        # orchestrator, decomposer, assignment, conflict, blackboard
  agents/         # character sheets, Qwen-Agent wiring, skills, MCP config
  baseline/       # single-agent runner
  metrics/        # instrumentation, eval harness
  replay/
  schemas/        # Pydantic events + TS type export
  tests/
frontend/
  src/world/      # PixiJS/Phaser scene, sprites, stage directions
  src/builder/    # society builder UI
  src/board/      # task board, negotiation viewer
  src/dashboard/  # metrics charts
scenarios/        # society templates (placeholder society lives here)
docs/
```

## 9. Decision log

| # | Decision | Status |
|---|---|---|
| 1 | Hybrid core: Qwen-Agent inner loop + custom society layer | Decided |
| 2 | Event-sourced architecture; 2D world is a renderer of events | Decided |
| 3 | 2D world-sim UI with premade assets; boardroom-view fallback | Decided |
| 4 | Society concept, setting, character roster | **[OPEN — spike]** |
| 5 | Benchmark task set & quality rubric | **[OPEN — spike]** |
| 6 | PixiJS vs Phaser | **[OPEN — Phase 0]** |
| 7 | Event schema v1 freeze | **[OPEN — Phase 0]** |
