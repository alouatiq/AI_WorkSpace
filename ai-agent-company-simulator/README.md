# AI Agent Company Simulator

> A 2D interactive workplace simulator where **AI agents operate as autonomous employees** within a
> corporate structure, and the **user is the CEO**. Built on top of the
> [WorkAdventure](../README.md) codebase as the spatial / proximity / rendering layer.

---

## Vision

A living 2D office where AI agents act as employees inside a real org chart. The human user logs in as
CEO, issues commands, hires/fires, forms teams, and watches agents move between desks and meeting rooms,
collaborate, and talk — all visible on screen.

### Core features

**Company structure**
- CEO (user-controlled)
- Departments (Engineering, Sales, HR, Operations, …)
- Teams within departments
- Team Managers (AI agents or player-assigned)
- Individual Contributors (AI agents)

**Agent capabilities**
- Accept job offers with role-specific requirements
- Autonomous task execution based on CEO commands
- Real-time 2D movement between office locations (desks, meeting rooms, departments)
- Communicate with each other (chat bubbles, conversations visible to the CEO)
- Performance metrics and productivity tracking

**CEO interactions**
- Login dashboard
- Issue commands/tasks to agents or teams
- Chat directly with agents for clarification
- Hire/fire agents based on job descriptions
- Create/dissolve teams and departments
- Monitor agent movements and conversations
- View task completion and deliverables

**Dynamic workflow**
- Agents move to meeting rooms when collaborating
- Agents request resources/information from each other
- Managers delegate tasks to team members
- Tasks propagate through the organizational hierarchy

---

## How the WorkAdventure base works (what we build on)

WorkAdventure is a monorepo of cooperating services:

| Service | Role | Tech |
|---|---|---|
| [`play/`](../play) → `front/` | Browser client, renders the 2D world | Phaser 3 + Svelte |
| [`play/`](../play) → `pusher/` | WebSocket front door for clients | uWebSockets + protobuf |
| [`back/`](../back) | Game-state authority: player list, positions, proximity **groups** | Node + gRPC |
| [`map-storage/`](../map-storage) | Stores/serves maps & areas | Node + gRPC/REST |
| `redis`, `synapse` (Matrix chat), `uploader` | Persistence, chat, files | — |

Mechanics that matter for an office of AI employees:

- **Avatars & movement** — Each "Woka" is a Phaser sprite. Movement is free-2D via
  `PositionMessage {x, y, direction, moving}`; the client does A* pathfinding
  ([`PathfindingManager.ts`](../play/src/front/Utils/PathfindingManager.ts)), but **the back does not
  enforce collisions** — a client simply declares its position. Protocol:
  [`messages/protos/messages.proto`](../messages/protos/messages.proto).
- **Maps & areas** — Tiled `.tmj` maps plus WorkAdventure "Areas" (rectangles with properties:
  `jitsiRoom`, `silent`, `focusable`, `start`, `exit`…). Desks, meeting rooms and departments map to
  areas; areas can be created at runtime via `WA.room.area.create()`.
- **Proximity "bubbles"** — [`back/src/Model/Group.ts`](../back/src/Model/Group.ts) auto-forms a group
  when avatars come within `GROUP_RADIUS` (48px). This is how "agents collaborate in a meeting room."
- **Visible speech** — Native **speech-bubble / thinking-cloud** above avatars via `SayMessage`
  ([`SayManager.ts`](../play/src/front/Phaser/Game/Say/SayManager.ts)), plus proximity chat and full
  Matrix chat. This is how agents "talk" on screen.
- **Programmatic control surfaces:**
  - **Scripting (iframe) API** — `WA.player.moveTo()`, `WA.chat.sendChatMessage()`, `WA.players` (read
    others). *Controls only the current player; runs in a browser.*
  - **RoomAPI** (gRPC, `X-API-Key`) — read/save/listen room **variables** and broadcast/listen
    **events**. Good for coordination/state, *but cannot move or spawn characters.*

### The single most important finding

**WorkAdventure has no native "bot"/"NPC" concept — every visible character is a connected client.**
AI employees cannot be spawned purely server-side. There are two ways to embody them:

- **Headless synthetic players** *(recommended core)* — a Node process opens a WebSocket per agent,
  authenticates anonymously (`POST /anonymLogin`, signed with `SECRET_KEY`), joins the room as a real
  player, and sends `UserMovesMessage` / `SayMessage`. The repo's
  [`benchmark/index.ts`](../benchmark/index.ts) already does this for 160+ synthetic players. Cheap,
  scales to a whole company.
- **Headless browsers per agent** — one Chrome per agent running the scripting API. Full client
  fidelity (free pathfinding) but heavy; only good for a handful of agents.

---

## Proposed architecture

```
                ┌────────────────────────────────────────┐
   CEO (human) ─┤  Browser: WorkAdventure client + a       │
                │  CEO Dashboard (Svelte overlay / iframe) │
                └───────────────┬──────────────────────────┘
                                │ websocket (real player)
   ┌────────────────────────────▼─────────────────────────┐
   │   WorkAdventure back + pusher + map-storage + redis    │  ← unchanged WA core
   └───────────▲───────────────────────────▲───────────────┘
   websockets  │ (one per agent)            │ RoomAPI (events/vars)
   ┌───────────┴───────────────────────────┴───────────────┐
   │            Agent Orchestrator (new Node service)        │
   │  • org model (DB): departments, teams, roles, tasks     │
   │  • one "AgentClient" per employee (synthetic player)    │
   │  • LLM brain (Claude API) → decide move/say/act          │
   │  • task queue + hierarchy/delegation engine             │
   └─────────────────────────────────────────────────────────┘
```

The org chart, jobs, tasks and metrics are a **new domain model** owned by the Orchestrator;
WorkAdventure is the **visualization + spatial/proximity layer**. Departments map to map areas; an agent
"working" = at its desk; "collaborating" = walked into a meeting-room area (auto-forms a bubble);
"talking" = `SayMessage` + chat.

---

## Roadmap

| Phase | Goal | Notes |
|---|---|---|
| **0 — Dev environment** | Stand up `back + pusher + redis + map-storage` | **Gate.** WorkAdventure's dev path is Docker; nothing functional can be built/tested until this is resolved. |
| **1 — Embodiment bake-off** | One agent that joins, walks to a desk, shows a speech bubble, visible to the CEO | Prototype **both** embodiment models and compare (see below). |
| **2 — Office map** | Department zones, desks, meeting rooms (Tiled + WAM areas) | Named target coordinates the orchestrator can address. |
| **3 — Orchestrator + org model** | Departments/teams/roles/agents/tasks in a DB; spawn & place N agents | Place agents at their desks. |
| **4 — LLM brain** | Per-agent Claude loop: role + task + surroundings → move/say/act | Proximity-driven conversations. |
| **5 — CEO dashboard** | Login, commands, hire/fire, teams, chat, monitoring | Svelte panel or scripting-API iframe in the CEO client. |
| **6 — Hierarchy & workflow** | Manager delegation, task propagation, resource requests, metrics | Tasks flow down the org tree. |

### Phase 1 bake-off (decided: prototype both, then compare)

- **A — Headless synthetic player:** a Node `AgentClient` from the
  [`benchmark/index.ts`](../benchmark/index.ts) template (anon-auth → join → move → speech bubble).
- **B — Scripted headless browser:** one Chrome running the iframe API (`WA.player.moveTo`,
  `WA.chat.sendChatMessage`).
- **Compare on:** scalability (agent count), movement fidelity (pathfinding/collisions), control
  richness, and dev effort. Pick the winner as the core; optionally keep B for "high-fidelity" agents.

---

## Decisions & open items

- **Agent embodiment:** prototype both (A and B), then compare. *(decided)*
- **Dev environment:** undecided. Two routes:
  1. Sign in to Docker Desktop (org account) → `docker compose up` runs the full stack as designed.
  2. Run services manually with Node + a local Redis (Memurai/WSL), no Docker. More setup; some
     features (Matrix chat) may be skipped initially.
  > Note: on the current machine, Docker Desktop image pulls are blocked by an org sign-in policy, so
  > route 1 requires signing in first.

---

## Status

Planning complete. **No application code has been written yet** — this document is the agreed plan.
Phase 0 (dev environment) is the first action when work begins.

---

## Key file references (WorkAdventure base)

| Concern | Path |
|---|---|
| Wire protocol (positions, say, join) | [`messages/protos/messages.proto`](../messages/protos/messages.proto) |
| Headless synthetic-player example | [`benchmark/index.ts`](../benchmark/index.ts) |
| Client pathfinding | [`play/src/front/Utils/PathfindingManager.ts`](../play/src/front/Utils/PathfindingManager.ts) |
| Speech bubbles / thinking clouds | [`play/src/front/Phaser/Game/Say/SayManager.ts`](../play/src/front/Phaser/Game/Say/SayManager.ts) |
| Proximity grouping (bubbles) | [`back/src/Model/Group.ts`](../back/src/Model/Group.ts) |
| Anonymous auth / token issuance | [`play/src/pusher/controllers/AuthenticateController.ts`](../play/src/pusher/controllers/AuthenticateController.ts) |
| RoomAPI (events / variables) | [`messages/protos/room-api.proto`](../messages/protos/room-api.proto) |
