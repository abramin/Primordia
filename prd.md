# Primordia - Product Requirements Document

This Product Requirements Document (PRD) outlines the vision for **"Primordia,"** a persistent, real-time strategy game built on the Elixir/OTP platform to showcase fault-tolerant, concurrent game server architecture.

---

## 1. Project Overview

**Primordia** is a simplified 4X (eXplore, eXpand, eXploit, eXterminate) web-based strategy game. Unlike traditional turn-based Civilization games, Primordia operates in **continuous real-time**. Actions like building a granary or marching an army to a distant forest happen asynchronously over minutes or hours, persisting even when the player is offline.

### Why Elixir/OTP?

Primordia's requirements align perfectly with Elixir's strengths:

| Requirement | Elixir/OTP Solution |
|-------------|---------------------|
| Continuous real-time evolution | GenServers maintain live state for each game entity |
| Perfect concurrency integrity | Actor model with message queues ensures sequential processing |
| Fault tolerance | OTP Supervisors restart failed processes automatically |
| Real-time updates | Phoenix Channels provide native WebSocket support |
| Horizontal scaling | Distributed Erlang allows clustering across nodes |

### Objectives

* Provide a seamless, low-latency UI for player commands via Phoenix Channels
* Manage a persistent world state that evolves independently of user sessions using GenServers
* Handle high-concurrency interactions (e.g., two players claiming the same tile) with perfect "first-come, first-served" integrity via the Actor model
* Demonstrate fault-tolerant game server architecture using OTP supervision trees

---

## 2. Tech Stack

### Backend

| Component | Technology | Purpose |
|-----------|------------|---------|
| Language | Elixir 1.16+ | Functional, concurrent programming |
| Framework | Phoenix 1.7+ | Web framework, routing, controllers |
| Real-time | Phoenix Channels | WebSocket connections, PubSub |
| Database | Ecto + PostgreSQL | Persistence, migrations, queries |
| State Management | GenServer | In-memory state for game entities |
| Fault Tolerance | OTP Supervisors | Process monitoring and restart |
| Scheduling | `:timer` / GenServer | Global tick, delayed actions |
| Background Jobs | Oban | Persistent job queue (optional) |

### Frontend

| Component | Technology | Purpose |
|-----------|------------|---------|
| Framework | React 18+ / Next.js | Component-based UI |
| State | Zustand or Redux | Client-side state management |
| Real-time | Phoenix JS Client | WebSocket connection to Channels |
| Rendering | Pixi.js or Canvas | 2D tile map rendering |
| Build | Vite | Fast development builds |

### Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| Database | PostgreSQL 15+ | Primary data store |
| Cache | ETS / Mnesia | In-memory caching (built into Erlang) |
| Deployment | Fly.io / Gigalixir | Elixir-optimized hosting |
| Monitoring | Telemetry + LiveDashboard | Observability |

---

## 3. Core Gameplay Mechanics

### A. The World (The Grid)

* **Map:** A coordinate-based grid (e.g., 100x100 tiles)
* **Tile Types:** Plains (Food), Mountains (Production), Water (Trade), and Forests (Mixed)
* **Fog of War:** Tiles are hidden until a player's unit or city border reveals them
* **Implementation:** World state can be loaded into ETS for fast concurrent reads

### B. City Management

* **Settling:** Players start with one "Settler" to found their first city
* **Growth:** Cities consume food to increase population. Higher population allows for more "Specialist" roles (Miners, Farmers, Scientists)
* **Production Queue:** Cities can build one item at a time (Units or Buildings). Each item has a **Production Cost** translated into **Real-World Time**
* **Implementation:** Each city runs as a GenServer, processing commands sequentially

### C. Unit Movement & Combat

* **Travel Time:** Moving from Tile A to Tile B is not instant. Movement speed depends on unit type and terrain
* **Combat:** Occurs automatically when two hostile units occupy the same tile. Results are calculated based on unit stats and defensive bonuses of the terrain
* **Implementation:** Moving units are tracked by a MovementServer that processes arrivals

### D. The Resource Tick

* Resources are not "mined" manually. The empire generates Gold, Food, and Science every **10-second interval (The Global Tick)**
* **Implementation:** A TickServer GenServer broadcasts tick events to all city processes

---

## 4. Functional Requirements

### 4.1 Player Actions (Command Ingestion)

* The system must accept player commands via Phoenix Channels (preferred) or REST API
* Commands are sent as messages to the appropriate GenServer (City, Unit, etc.)
* The GenServer validates and processes commands sequentially, eliminating race conditions
* Responses are pushed back via the same Channel connection

### 4.2 Asynchronous Event Processing

* **Delayed Execution:** Actions with a duration (e.g., "Build Wall - 5 mins") are handled by scheduling a message to the GenServer using `Process.send_after/3`
* **Sequential Integrity:** GenServers process messages one at a time. If two players send commands to the same entity, they are naturally serialized
* **Crash Recovery:** If a GenServer crashes, the Supervisor restarts it, and state is recovered from the database

### 4.3 World Persistence & Evolution

* Game state is persisted to PostgreSQL via Ecto
* GenServers periodically snapshot their state to the database
* On startup, GenServers hydrate their state from the database
* Background processes (Global Tick) update the world even if no players are connected

### 4.4 Real-time Updates

* Phoenix Channels push state changes to connected clients immediately
* Players subscribe to topics: `game:player:{id}`, `game:world:{region}`
* No polling required - all updates are server-pushed

---

## 5. Architecture Overview

### Process Supervision Tree

```
Application
├── Primordia.Repo (Ecto)
├── PrimordiaWeb.Endpoint (Phoenix)
├── Primordia.GameSupervisor (DynamicSupervisor)
│   ├── Primordia.TickServer (GenServer) - Global 10s tick
│   ├── Primordia.WorldServer (GenServer) - Tile state, fog of war
│   └── Primordia.PlayerSupervisor (DynamicSupervisor)
│       ├── Primordia.PlayerServer (GenServer) - Player 1 state
│       │   ├── CityServer (GenServer) - Rome
│       │   ├── CityServer (GenServer) - Alexandria
│       │   └── UnitServer (GenServer) - Scout #1
│       └── Primordia.PlayerServer (GenServer) - Player 2 state
│           └── ...
└── Primordia.PubSub (Phoenix.PubSub)
```

### Message Flow Example: Found City

```
1. Client sends "found_city" via Phoenix Channel
2. Channel handler validates auth, forwards to PlayerServer
3. PlayerServer:
   a. Validates player owns settler at location
   b. Calls CityServer.start_child() to spawn new city process
   c. Sends message to UnitServer to consume settler
   d. Persists changes to database
   e. Broadcasts "city_founded" event via PubSub
4. Channel receives PubSub event, pushes to client
5. Client updates UI
```

---

## 6. User Flows

### Flow 1: Settling and Building

1. **Player** connects via WebSocket and receives initial game state
2. **Player** sends `{"event": "found_city", "payload": {"settler_id": "...", "name": "Rome"}}`
3. **System** (PlayerServer) validates, creates CityServer process, removes settler
4. **System** pushes `city_founded` event to player's channel
5. **Player** sends `{"event": "start_production", "payload": {"city_id": "...", "building": "granary"}}`
6. **System** (CityServer) validates, schedules completion via `Process.send_after(self(), :complete_production, 120_000)`
7. **Player** disconnects
8. **System** (2 mins later) CityServer receives `:complete_production`, updates state, persists to DB
9. **Player** reconnects, receives current state including completed Granary

### Flow 2: Exploration

1. **Player** sends `{"event": "move_unit", "payload": {"unit_id": "...", "to": {"x": 15, "y": 20}}}`
2. **System** (UnitServer) calculates path and arrival time
3. **System** schedules position updates at each tile transition
4. **System** pushes `unit_moving` event with path and ETA
5. **Player** sees animated movement on map
6. **System** (WorldServer) reveals tiles as unit's vision touches them
7. **System** pushes `tiles_revealed` events as unit moves
8. **System** (UnitServer) receives `:arrive` message, updates final position

---

## 7. Non-Functional Requirements

### 7.1 Scalability

* **Vertical:** Single BEAM VM can handle 100,000+ lightweight processes
* **Horizontal:** Distributed Erlang allows clustering Phoenix nodes
* **Initial target:** 1-2 players, architected to scale to thousands
* **Process distribution:** Each player's game entities run in isolated process trees

### 7.2 Fault Tolerance

* **Supervision:** All GenServers are supervised with `:one_for_one` strategy
* **Crash isolation:** One player's crashed process doesn't affect others
* **State recovery:** GenServers restore state from database on restart
* **No message loss:** Critical state changes are persisted before acknowledgment

### 7.3 Observability

* **Telemetry:** Instrument all GenServer calls and Channel events
* **LiveDashboard:** Built-in Phoenix tool for real-time monitoring
* **Metrics to track:**
  * Message queue lengths per GenServer
  * Channel connection count
  * Database query latency
  * Tick processing time

### 7.4 Latency Requirements

| Operation | Target Latency |
|-----------|---------------|
| Channel message round-trip | < 50ms |
| Command validation | < 10ms |
| Database write | < 20ms |
| State broadcast | < 5ms |

---

## 8. Game Data Definitions

### Entities

| Entity | Attributes | Process |
|--------|------------|---------|
| **Player** | id, username, gold, food, science, tech_tree | PlayerServer |
| **City** | id, name, owner_id, x, y, population, food_stores, production_queue | CityServer |
| **Unit** | id, type, owner_id, x, y, destination, arrival_time, health | UnitServer |
| **Tile** | x, y, terrain_type, visible_to (MapSet of player_ids) | WorldServer (ETS) |

### Events (PubSub Topics)

| Event | Topic | Payload |
|-------|-------|---------|
| City Founded | `player:{id}` | `%{city_id, name, x, y}` |
| Building Complete | `player:{id}` | `%{city_id, building_type}` |
| Unit Moved | `player:{id}` | `%{unit_id, x, y}` |
| Tiles Revealed | `player:{id}` | `%{tiles: [{x, y, terrain}]}` |
| Combat Occurred | `player:{id}` | `%{attacker, defender, result}` |
| Resources Updated | `player:{id}` | `%{gold, food, science}` |

---

## 9. API Design

### Phoenix Channels

```elixir
# Join player's game channel
socket.channel("game:player:#{player_id}", %{})

# Incoming events (client -> server)
"found_city"       -> %{settler_id, name}
"start_production" -> %{city_id, item_type}
"move_unit"        -> %{unit_id, to: %{x, y}}
"research_tech"    -> %{tech_name}

# Outgoing events (server -> client)
"game_state"       -> Full current state on join
"city_founded"     -> %{city}
"production_started" -> %{city_id, item, completes_at}
"production_complete" -> %{city_id, item}
"unit_moved"       -> %{unit_id, x, y}
"resources_tick"   -> %{gold, food, science}
"combat_result"    -> %{attacker, defender, winner, damage}
```

### REST API (Secondary)

For operations that don't need real-time:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/auth/register` | Create account |
| POST | `/api/auth/login` | Get auth token |
| GET | `/api/game/state` | Full game state (fallback) |
| GET | `/api/leaderboard` | Player rankings |

---

## 10. Security Considerations

* **Authentication:** Phoenix Token or Guardian JWT for Channel auth
* **Authorization:** Each Channel/GenServer validates player owns the entity
* **Rate Limiting:** Hammer library for command rate limiting
* **Input Validation:** Ecto changesets validate all incoming data
* **Process Isolation:** Players cannot send messages to other players' processes

---

## 11. Future Scope

* **Diplomacy:** Trading resources between players via direct Channel messages
* **AI Barbarians:** Supervised GenServer processes for AI-controlled units
* **Victory Conditions:** GameServer process monitors win conditions
* **Clustering:** Distribute player processes across multiple nodes
* **Replay System:** Event sourcing for game replay functionality

---

## 12. Success Metrics

| Metric | Target |
|--------|--------|
| Server uptime | 99.9% |
| Message processing latency (p99) | < 100ms |
| Concurrent players per node | 1,000+ |
| Zero data loss on server restart | 100% |
| Client reconnection time | < 2s |

---

## Appendix: Why Not Other Stacks?

| Stack | Limitation for Primordia |
|-------|-------------------------|
| **Django/Node.js** | Requires external job queue (Celery/BullMQ) for delayed execution; no native actor model for concurrency |
| **Go** | Excellent concurrency but lacks OTP-style supervision and hot code reloading |
| **Rust** | Performance overkill; longer development time for this use case |

Elixir/OTP provides the best balance of **developer productivity**, **concurrency guarantees**, and **fault tolerance** for a persistent real-time game server.
