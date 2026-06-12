# Agent Society — 20-Day Hackathon Roadmap

## Context

Hackathon Track 3 ("Agent Society") requires a multi-agent collaboration system demonstrating: task decomposition & role assignment, disagreement/conflict resolution, and a **measurable efficiency gain over a single-agent baseline**. Judging: Technical Depth & QwenCloud usage (30%), Innovation & code quality (30%), Problem Value (25%), Presentation & docs (15%).

**Product vision:** an interactive, game-like 2D world (Stanford Generative Agents style) where the user *builds* a society/organization of AI agents, gives it a complex goal, and watches the agents visibly decompose work, negotiate, resolve conflicts, and deliver — with a live metrics dashboard comparing against a single-agent baseline. The society content itself is a placeholder/template for now; the roadmap builds the engine and the experience around it.

**Constraints & decisions:**

- QwenCloud is mandatory; API access already available.
- Team of 4, Python-heavy (1–2 can do React/TS). Deadline: 20 days (~July 1, 2026).
- **Hybrid agent core:** Qwen-Agent SDK handles each agent's inner loop (DashScope calls, tool-calling, custom skills, MCP plumbing); we build the *society layer* ourselves — that custom layer is the technical-depth story.
- Benchmark task type is undecided → Day 1–2 decision spike (recommendation below).

## Architecture (target)

```
┌────────────────────────── Frontend (React + PixiJS/Phaser) ─────────────────────────┐
│  Society Builder (configure agents/roles/goal)  ·  2D World View (sprites, bubbles) │
│  Task Board / Negotiation Viewer  ·  Metrics Dashboard (society vs baseline)        │
└───────────────────────────────▲ WebSocket (event stream) ───────────────────────────┘
┌───────────────────────────────┴── Backend (Python / FastAPI) ───────────────────────┐
│  SOCIETY LAYER (custom — the innovation):                                           │
│   · Orchestrator + Event Bus (event-sourced: every msg/action = event)              │
│   · Task Decomposer (planner agent → task DAG)                                      │
│   · Role Assignment (contract-net: agents bid on subtasks by capability/load)       │
│   · Negotiation & Conflict Resolution (structured debate + arbiter/vote;            │
│     resource-lock conflicts via priority negotiation)                               │
│   · Shared Memory / Blackboard (society state, artifacts)                           │
│   · Metrics Engine (tokens, wall-clock, success rate, quality) + Baseline Runner    │
│   · Replay Engine (record/replay event logs — demo safety net)                      │
│  AGENT RUNTIME (Qwen-Agent SDK): per-agent loop, qwen-max/plus via DashScope,       │
│   custom skills, MCP tool servers (search, code-exec, filesystem)                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

Key principle: **the 2D world is a visualization of the orchestration event stream, not a physics sim.** Agents "walk to a meeting table" when a negotiation event fires, etc. This keeps the game layer thin and the society swappable (placeholder now, richer scenarios later). The event schema is the contract between teams — defined in week 1, versioned.

## Benchmark task — decision spike (Days 1–2)

**Recommendation:** in-world *organizational* scenario that produces a real artifact — e.g., a "startup company" society given goals like "produce a competitive-analysis report" or "design + spec a landing page." Real artifacts give Problem-Value relevance; in-world framing keeps it controllable and demoable. Metrics vs single agent with identical tools: wall-clock time, total tokens/cost, subtask success rate, artifact quality (LLM-judge rubric + human check), recovery from injected failures. Eval set: 5–8 fixed tasks, ≥3 runs each.

## Roadmap

### Phase 0 — Foundation (Days 1–2)

- Benchmark-task decision spike (above) — whole team, half day.
- Monorepo setup (`backend/` FastAPI + `frontend/` Vite React), lint/CI, .env handling.
- DashScope key validation; Qwen-Agent "hello world" with one tool call.
- Frontend scaffold: render a pre-made tilemap (Kenney/LPC assets) with one movable sprite.
- **Define and freeze v1 event schema** (agent_spawned, message_sent, task_decomposed, bid_placed, conflict_opened, conflict_resolved, artifact_produced, metrics_tick…).

### Phase 1 — Vertical slice (Days 3–7)

- Society layer v1: orchestrator, in-process event bus, 3 hardcoded agents (Planner, 2 Workers), naive decomposition → round-robin assignment.
- WebSocket bridge streaming events to frontend.
- Frontend: agent sprites move on events, speech bubbles for messages, scrolling conversation log panel.
- **M1 (Day 7): end-to-end demo — user types a goal, 3 agents visibly decompose it and chat in the 2D world.** If M1 slips past Day 9, cut scope (see Risks).

### Phase 2 — Differentiators (Days 8–13)

- Contract-net role assignment (capability/load-based bidding) replacing round-robin.
- Conflict resolution: structured debate protocol + arbiter agent; timeout/failure → task reassignment (error handling is an explicit judging point).
- Custom Qwen skills + ≥2 MCP integrations (web search, code execution/filesystem).
- Society Builder UI: pick scenario template, configure agent roles/personas/model tier, set goal.
- Baseline single-agent runner + metrics engine (shared instrumentation).
- Replay engine (persist event log, replay at adjustable speed).
- **M2 (Day 13): feature-complete — a configured society completes a benchmark task with metrics recorded.**

### Phase 3 — Evaluation & polish (Days 14–17)

- Run full eval: tasks × {society, single-agent} × 3 runs; build comparison dashboard (time, cost, success, quality).
- UI polish: negotiation-table animation, task-board overlay, dashboard charts.
- Performance pass: parallel agent calls, response streaming, prompt caching.
- Failure-injection demo (kill an agent mid-task → society reassigns) — strong live-demo moment.
- **M3 (Day 17): demo-ready build + headline numbers.**

### Phase 4 — Presentation (Days 18–20)

- Architecture doc + diagrams, README, setup guide.
- Demo script using replay mode (deterministic) with one live segment; record backup video.
- Rehearsals ×2; buffer for bug fixes. No new features after Day 18.

## Workstreams (team of 4)

| Person | Ownership |
|---|---|
| P1 (backend lead) | Society layer: orchestrator, decomposition, contract-net, conflict protocols |
| P2 (AI/Qwen) | Qwen-Agent integration, custom skills, MCP servers, prompt engineering |
| P3 (web-capable) | Frontend: 2D world, builder UI, dashboard |
| P4 (flex) | Eval harness, baseline runner, metrics, replay; docs/demo from Day 14; helps P3 during Phase 2–3 |

## Judging alignment

- **Technical depth (30%):** custom society layer (contract-net, debate protocol, event-sourced replay) on top of deep Qwen usage (Qwen-Agent, custom skills, MCP, model-tier mixing).
- **Innovation (30%):** event-sourced architecture, swappable scenario templates, failure recovery, replay engine; modular boundaries (agent runtime / society layer / viz are independent).
- **Problem value (25%):** real-artifact tasks (org-as-a-service framing); scenario templates make it productizable/open-sourceable.
- **Presentation (15%):** the 2D world *is* the visualization of key logic; replay mode guarantees a clean demo; architecture docs scheduled, not an afterthought.

## Risks & mitigations

- **2D world scope creep** → premade assets, world = event visualizer only; fallback: drop tilemap to a stylized canvas "boardroom" view (same event schema, so backend unaffected).
- **LLM nondeterminism kills the live demo** → replay mode + recorded video.
- **DashScope rate limits/cost** → request queueing, prompt caching, qwen-plus/turbo for worker agents, cap eval concurrency.
- **M1 slip** → cut Society Builder to a JSON config; cut conflict resolution to a simple arbiter vote.

## Verification / milestone checks

- **M1:** run backend + frontend locally; type a goal; confirm 3 agents move/converse in-world and the event log matches.
- **M2:** run one benchmark task end-to-end via the Builder UI; confirm metrics row written for both society and baseline runs.
- **M3:** eval script produces the comparison table; replay of a recorded run renders identically twice.
- Throughout: backend unit tests for decomposition/bidding/conflict state machines (pure logic, no LLM needed — mock agent responses).
