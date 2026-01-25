# Primordia - Vertical Slices

This document breaks down the Primordia game into vertical slices that can be implemented incrementally. Each slice delivers end-to-end functionality across the full stack (Phoenix, React, PostgreSQL, GenServers).

---

## Slice 1: Project Setup & Static World View

**Goal:** Set up the Elixir/Phoenix project, create the world, and display a static map.

### Backend (Phoenix)

**Project Setup:**
- Create new Phoenix project: `mix phx.new primordia --database postgres`
- Configure PostgreSQL connection in `config/dev.exs`
- Set up CORS for React frontend (add `corsica` plug)
- Create basic folder structure following Phoenix conventions

**Database (Ecto):**
- Migration: `tiles` table with `x`, `y`, `terrain_type` fields
- Add composite index on `(x, y)` for spatial queries
- Seed script to generate 100x100 tile grid with random terrain

**Modules:**
```
lib/primordia/
├── game/
│   ├── tile.ex           # Ecto schema
│   └── world.ex          # Context module for tile queries
```

**REST API:**
- `GET /api/tiles` - Returns all tiles (paginated or chunked)
- TileController with JSON view

### Frontend (React)

**Project Setup:**
- Create Vite + React + TypeScript project
- Install dependencies: `phoenix`, `pixi.js`, `zustand`
- Configure proxy to Phoenix backend

**Components:**
- `GameMap.tsx` - Pixi.js canvas rendering 100x100 grid
- `Tile.tsx` - Individual tile sprite with terrain-based coloring
- Basic color scheme: Plains (green), Mountains (gray), Water (blue), Forest (dark green)

**Acceptance Criteria:**
- Phoenix server starts and connects to PostgreSQL
- Running `mix run priv/repo/seeds.exs` creates 100x100 world
- React app displays rendered map with different terrain colors
- No authentication yet - map is publicly visible

---

## Slice 2: User Authentication

**Goal:** Players can register, log in, and receive a token for authenticated requests.

### Backend (Phoenix)

**Dependencies:**
- Add `bcrypt_elixir` for password hashing
- Add `guardian` for JWT token generation

**Database (Ecto):**
- Migration: `users` table with `id`, `email`, `username`, `password_hash`
- Unique constraint on email and username

**Modules:**
```
lib/primordia/
├── accounts/
│   ├── user.ex           # Ecto schema with password hashing
│   └── accounts.ex       # Context: register_user, authenticate_user
lib/primordia_web/
├── controllers/
│   ├── auth_controller.ex    # register, login endpoints
│   └── auth_json.ex          # JSON views
├── guardian.ex               # Guardian configuration
└── plugs/
    └── auth_pipeline.ex      # JWT verification plug
```

**REST API:**
- `POST /api/auth/register` - Create account, return token
- `POST /api/auth/login` - Authenticate, return token
- `GET /api/auth/me` - Return current user (protected)

### Frontend (React)

**Components:**
- `LoginForm.tsx` - Email/password form
- `RegisterForm.tsx` - Registration form
- `AuthContext.tsx` - Store JWT token, provide auth state
- Protected route wrapper

**State:**
- Zustand store for auth: `{ token, user, login(), logout() }`

**Acceptance Criteria:**
- User can register with email/username/password
- User can log in and receive JWT token
- Token is stored in localStorage
- Protected routes redirect to login if not authenticated

---

## Slice 3: Player State & Initial Settler

**Goal:** New players spawn with resources and one Settler unit visible on the map.

### Backend (Phoenix)

**Database (Ecto):**
- Migration: `players` table with `user_id`, `gold`, `food`, `science`
- Migration: `units` table with `id`, `player_id`, `unit_type`, `x`, `y`, `health`
- Player created automatically when user first accesses game

**Modules:**
```
lib/primordia/
├── game/
│   ├── player.ex         # Ecto schema
│   ├── unit.ex           # Ecto schema
│   └── game.ex           # Context: get_or_create_player, spawn_settler
```

**Logic:**
- On first game access, create Player record linked to User
- Spawn Settler at random valid land tile
- Initial resources: gold=100, food=50, science=0

**REST API:**
- `GET /api/game/state` - Returns player resources and all owned units
- `GET /api/units` - Returns player's units with positions

### Frontend (React)

**Components:**
- `UnitLayer.tsx` - Pixi.js layer rendering units on map
- `ResourcePanel.tsx` - Display Gold, Food, Science
- `UnitSprite.tsx` - Settler icon at coordinates

**State:**
- Zustand game store: `{ player, units, fetchGameState() }`

**Acceptance Criteria:**
- New authenticated player automatically receives Player record
- Player has one Settler unit at random position
- Map shows Settler icon at correct coordinates
- Resource panel displays current resources

---

## Slice 4: Phoenix Channels & Real-time Connection

**Goal:** Establish WebSocket connection for real-time game updates.

### Backend (Phoenix)

**Modules:**
```
lib/primordia_web/
├── channels/
│   ├── user_socket.ex        # Socket authentication
│   ├── game_channel.ex       # game:player:{id} topic
│   └── presence.ex           # Track connected players (optional)
```

**Channel Events:**
- `join` - Authenticate and return current game state
- `game_state` (outgoing) - Full state pushed on join

**Socket Authentication:**
- Verify JWT token in `connect/3`
- Store `player_id` in socket assigns

### Frontend (React)

**Services:**
- `socket.ts` - Phoenix Socket connection with token auth
- `gameChannel.ts` - Join `game:player:{id}`, handle events

**Hooks:**
- `useGameChannel()` - Connect on mount, handle events, update store

**Acceptance Criteria:**
- React app connects to Phoenix Channel on game load
- Connection authenticated with JWT token
- Server pushes initial game state on join
- Console logs show successful connection

---

## Slice 5: City Founding (Synchronous via Channel)

**Goal:** Player can found a city using their Settler via Channel message.

### Backend (Phoenix)

**Database (Ecto):**
- Migration: `cities` table with `id`, `player_id`, `name`, `x`, `y`, `population`, `food_stores`

**Modules:**
```
lib/primordia/
├── game/
│   ├── city.ex           # Ecto schema
│   └── game.ex           # found_city/3 function
```

**Channel Events (game_channel.ex):**
```elixir
def handle_in("found_city", %{"settler_id" => id, "name" => name}, socket) do
  case Game.found_city(socket.assigns.player_id, id, name) do
    {:ok, city} ->
      broadcast!(socket, "city_founded", CityJSON.show(city))
      {:reply, {:ok, %{city_id: city.id}}, socket}
    {:error, reason} ->
      {:reply, {:error, %{reason: reason}}, socket}
  end
end
```

**Logic:**
- Validate player owns settler at location
- Delete settler unit
- Create city at settler's coordinates
- Broadcast `city_founded` event

### Frontend (React)

**Components:**
- `CitySprite.tsx` - City icon rendering
- Unit selection state - click Settler to select
- "Found City" button when Settler selected
- Name input modal

**Channel Integration:**
- Send `found_city` event with settler_id and name
- Handle `city_founded` event to update local state

**Acceptance Criteria:**
- Clicking Settler shows "Found City" action
- Entering name and confirming sends Channel message
- Settler disappears, City appears at location
- Other connected clients see city appear (via broadcast)

---

## Slice 6: GenServer Architecture - Game Supervisor

**Goal:** Introduce GenServer-based state management with supervision tree.

### Backend (Phoenix)

**Modules:**
```
lib/primordia/
├── game/
│   ├── game_supervisor.ex    # DynamicSupervisor for game processes
│   ├── player_server.ex      # GenServer per player
│   └── registry.ex           # Process registry helpers
```

**GameSupervisor:**
```elixir
defmodule Primordia.Game.GameSupervisor do
  use DynamicSupervisor

  def start_link(init_arg) do
    DynamicSupervisor.start_link(__MODULE__, init_arg, name: __MODULE__)
  end

  def start_player_server(player_id) do
    DynamicSupervisor.start_child(__MODULE__, {PlayerServer, player_id})
  end
end
```

**PlayerServer:**
```elixir
defmodule Primordia.Game.PlayerServer do
  use GenServer

  def start_link(player_id) do
    GenServer.start_link(__MODULE__, player_id, name: via_tuple(player_id))
  end

  def init(player_id) do
    # Load state from database
    player = Game.get_player!(player_id)
    units = Game.list_units(player_id)
    cities = Game.list_cities(player_id)
    {:ok, %{player: player, units: units, cities: cities}}
  end

  # Handle calls for game actions...
end
```

**Application Startup:**
- Add GameSupervisor to application supervision tree
- Start PlayerServer when player joins Channel

**Acceptance Criteria:**
- GameSupervisor starts with application
- Joining Channel starts/finds PlayerServer for that player
- PlayerServer loads state from database on init
- Process stays alive between Channel reconnects

---

## Slice 7: Global Tick - Resource Generation

**Goal:** Every 10 seconds, all cities generate resources automatically.

### Backend (Phoenix)

**Modules:**
```
lib/primordia/
├── game/
│   ├── tick_server.ex        # GenServer for global tick
│   └── resource_calculator.ex # Pure functions for resource math
```

**TickServer:**
```elixir
defmodule Primordia.Game.TickServer do
  use GenServer

  @tick_interval 10_000  # 10 seconds

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def init(state) do
    schedule_tick()
    {:ok, state}
  end

  def handle_info(:tick, state) do
    # Get all active PlayerServers and send tick message
    for {player_id, pid} <- Registry.lookup(Primordia.PlayerRegistry, :players) do
      GenServer.cast(pid, :process_tick)
    end
    schedule_tick()
    {:noreply, %{state | tick_count: state.tick_count + 1}}
  end

  defp schedule_tick, do: Process.send_after(self(), :tick, @tick_interval)
end
```

**PlayerServer tick handling:**
```elixir
def handle_cast(:process_tick, state) do
  {resources, new_state} = ResourceCalculator.calculate_tick(state)

  # Persist to database
  Game.update_player_resources(state.player.id, resources)

  # Broadcast to connected client
  PrimordiaWeb.Endpoint.broadcast!("game:player:#{state.player.id}", "resources_tick", resources)

  {:noreply, new_state}
end
```

### Frontend (React)

**Updates:**
- Handle `resources_tick` event in game channel
- Animate resource counter changes
- Show tick indicator (optional pulse effect)

**Acceptance Criteria:**
- Every 10 seconds, player resources increase
- Resources update even if player is idle
- Frontend reflects updates in real-time via Channel
- Tick continues running when player disconnects (server-side)

---

## Slice 8: Building Production Queue (Async with Process.send_after)

**Goal:** Cities can queue buildings that complete after real-world time.

### Backend (Phoenix)

**Database (Ecto):**
- Migration: Add `production_item`, `production_started_at`, `production_completes_at` to cities
- Migration: Add `buildings` JSON array field to cities

**Modules:**
```
lib/primordia/
├── game/
│   ├── city_server.ex        # GenServer per city
│   ├── building.ex           # Building definitions (Granary, etc.)
│   └── production.ex         # Production queue logic
```

**CityServer:**
```elixir
defmodule Primordia.Game.CityServer do
  use GenServer

  def start_link({city_id, player_id}) do
    GenServer.start_link(__MODULE__, {city_id, player_id}, name: via_tuple(city_id))
  end

  def init({city_id, player_id}) do
    city = Game.get_city!(city_id)
    state = %{city: city, player_id: player_id}

    # Resume any in-progress production
    state = maybe_schedule_production_completion(state)
    {:ok, state}
  end

  def handle_call({:start_production, building_type}, _from, state) do
    case Production.start_building(state.city, building_type) do
      {:ok, city, duration_ms} ->
        # Schedule completion message
        Process.send_after(self(), {:complete_production, building_type}, duration_ms)
        {:reply, {:ok, city}, %{state | city: city}}
      {:error, reason} ->
        {:reply, {:error, reason}, state}
    end
  end

  def handle_info({:complete_production, building_type}, state) do
    {:ok, city} = Production.complete_building(state.city, building_type)

    # Persist and broadcast
    Game.update_city(city)
    broadcast_to_player(state.player_id, "production_complete", %{
      city_id: city.id,
      building: building_type
    })

    {:noreply, %{state | city: city}}
  end
end
```

**Channel Integration:**
```elixir
def handle_in("start_production", %{"city_id" => city_id, "building" => building}, socket) do
  case CityServer.start_production(city_id, building) do
    {:ok, city} ->
      {:reply, {:ok, %{completes_at: city.production_completes_at}}, socket}
    {:error, reason} ->
      {:reply, {:error, %{reason: reason}}, socket}
  end
end
```

### Frontend (React)

**Components:**
- `CityPanel.tsx` - Show city details when selected
- `ProductionQueue.tsx` - Display current production with countdown
- `BuildingMenu.tsx` - List available buildings with costs/durations

**State:**
- Track `production_completes_at` timestamp
- Display countdown timer

**Acceptance Criteria:**
- Player can select city and choose "Build Granary"
- Production timer shows countdown (e.g., 2 minutes)
- Player can disconnect and reconnect
- Building completes at exact scheduled time
- `production_complete` event pushed to client
- Building effects apply (e.g., Granary increases food generation)

---

## Slice 9: Unit Movement (Async with Scheduled Arrivals)

**Goal:** Units move tile-by-tile over real time, with position updates broadcast.

### Backend (Phoenix)

**Database (Ecto):**
- Add to units: `destination_x`, `destination_y`, `arrival_time`, `path` (JSON array)

**Modules:**
```
lib/primordia/
├── game/
│   ├── unit_server.ex        # GenServer per unit
│   ├── pathfinding.ex        # A* or simple pathfinding
│   └── movement.ex           # Movement calculations
```

**UnitServer:**
```elixir
defmodule Primordia.Game.UnitServer do
  use GenServer

  def handle_call({:move_to, dest_x, dest_y}, _from, state) do
    unit = state.unit
    path = Pathfinding.find_path({unit.x, unit.y}, {dest_x, dest_y})

    case path do
      [] ->
        {:reply, {:error, :no_path}, state}
      path ->
        # Calculate arrival time based on path length and unit speed
        arrival_time = Movement.calculate_arrival(path, unit.movement_speed)

        # Schedule arrival
        Process.send_after(self(), :arrive, arrival_time)

        # Schedule intermediate position updates for animation
        schedule_position_updates(path, unit.movement_speed)

        updated_unit = %{unit |
          destination_x: dest_x,
          destination_y: dest_y,
          path: path,
          arrival_time: DateTime.add(DateTime.utc_now(), arrival_time, :millisecond)
        }

        Game.update_unit(updated_unit)
        broadcast_movement_started(state.player_id, updated_unit)

        {:reply, {:ok, updated_unit}, %{state | unit: updated_unit}}
    end
  end

  def handle_info({:position_update, {x, y}}, state) do
    unit = %{state.unit | x: x, y: y}
    broadcast_position(state.player_id, unit)
    {:noreply, %{state | unit: unit}}
  end

  def handle_info(:arrive, state) do
    unit = %{state.unit |
      x: state.unit.destination_x,
      y: state.unit.destination_y,
      destination_x: nil,
      destination_y: nil,
      path: nil,
      arrival_time: nil
    }

    Game.update_unit(unit)
    broadcast_arrived(state.player_id, unit)

    # Check for combat at destination
    check_combat_at_location(unit)

    {:noreply, %{state | unit: unit}}
  end
end
```

### Frontend (React)

**Components:**
- Click unit to select, click tile to set destination
- `MovementPath.tsx` - Show path preview line
- Animate unit sprite along path
- Show ETA tooltip

**Channel Events:**
- Send `move_unit` with unit_id and destination
- Handle `unit_moving`, `unit_position`, `unit_arrived`

**Acceptance Criteria:**
- Player can order unit to move to destination
- Unit moves tile-by-tile with visible animation
- Movement continues even if player disconnects
- Unit arrives at exact calculated time
- Other players see unit movement in real-time

---

## Slice 10: Fog of War

**Goal:** Tiles are hidden until revealed by player's units or cities.

### Backend (Phoenix)

**Database (Ecto):**
- Migration: `tile_visibility` table with `player_id`, `x`, `y`, `revealed_at`
- Index on `(player_id, x, y)`

**Alternative - ETS for Performance:**
```elixir
defmodule Primordia.Game.FogOfWar do
  use GenServer

  def init(_) do
    # Create ETS table for fast visibility lookups
    table = :ets.new(:fog_of_war, [:set, :public, read_concurrency: true])
    {:ok, %{table: table}}
  end

  def is_visible?(player_id, x, y) do
    :ets.lookup(:fog_of_war, {player_id, x, y}) != []
  end

  def reveal_tiles(player_id, tiles) do
    entries = Enum.map(tiles, fn {x, y} -> {{player_id, x, y}, true} end)
    :ets.insert(:fog_of_war, entries)

    # Also persist to database for recovery
    FogOfWarPersistence.save_revealed(player_id, tiles)
  end
end
```

**Vision Calculation:**
```elixir
defmodule Primordia.Game.Vision do
  @unit_vision_radius 2
  @city_vision_radius 3

  def tiles_in_vision({x, y}, radius) do
    for dx <- -radius..radius,
        dy <- -radius..radius,
        dx * dx + dy * dy <= radius * radius,
        do: {x + dx, y + dy}
  end
end
```

**Integration:**
- Reveal tiles when city founded
- Reveal tiles as unit moves
- Filter tile data in API responses to only visible tiles

### Frontend (React)

**Components:**
- `FogLayer.tsx` - Overlay rendering fog on unrevealed tiles
- Fog as dark overlay with alpha
- Revealed tiles stay revealed (no re-fogging)

**Acceptance Criteria:**
- New player only sees tiles around starting Settler
- Founding city reveals surrounding tiles
- Moving unit progressively reveals tiles
- API only returns visible tile data
- Revealed tiles persist across sessions

---

## Slice 11: Combat System

**Goal:** When hostile units meet on a tile, combat resolves automatically.

### Backend (Phoenix)

**Database (Ecto):**
- Migration: `combat_logs` table for battle history
- Add `health`, `attack`, `defense` fields to units

**Modules:**
```
lib/primordia/
├── game/
│   ├── combat.ex             # Combat resolution logic
│   └── combat_server.ex      # Handles combat events
```

**Combat Resolution:**
```elixir
defmodule Primordia.Game.Combat do
  def resolve(attacker, defender, terrain) do
    terrain_bonus = terrain_defense_bonus(terrain)

    attacker_roll = attacker.attack * :rand.uniform()
    defender_roll = defender.defense * terrain_bonus * :rand.uniform()

    damage_to_defender = max(0, round(attacker_roll - defender_roll * 0.5))
    damage_to_attacker = max(0, round(defender_roll - attacker_roll * 0.5))

    %{
      attacker_damage: damage_to_attacker,
      defender_damage: damage_to_defender,
      attacker_survives: attacker.health > damage_to_attacker,
      defender_survives: defender.health > damage_to_defender
    }
  end
end
```

**UnitServer Combat Check:**
```elixir
def check_combat_at_location(unit) do
  case Game.find_enemy_unit_at(unit.x, unit.y, unit.player_id) do
    nil -> :ok
    enemy ->
      result = Combat.resolve(unit, enemy, Game.get_terrain(unit.x, unit.y))

      # Update units based on result
      apply_combat_result(unit, enemy, result)

      # Broadcast to both players
      broadcast_combat(unit.player_id, enemy.player_id, result)
  end
end
```

### Frontend (React)

**Components:**
- `CombatAnimation.tsx` - Flash/effect when combat occurs
- `CombatLog.tsx` - Show recent battles
- Health bars on units

**Channel Events:**
- Handle `combat_result` with attacker, defender, damage, outcome

**Acceptance Criteria:**
- When unit arrives at tile with enemy, combat triggers
- Combat result calculated based on stats and terrain
- Damaged units show reduced health
- Destroyed units removed from map
- Both players notified of combat result

---

## Slice 12: City Growth & Population

**Goal:** Cities accumulate food and grow population over time.

### Backend (Phoenix)

**CityServer Enhancement:**
```elixir
def handle_cast(:process_tick, state) do
  city = state.city
  food_generated = calculate_food_generation(city)

  new_food_stores = city.food_stores + food_generated
  food_for_growth = city.population * 10

  {new_population, new_food_stores} =
    if new_food_stores >= food_for_growth do
      {city.population + 1, new_food_stores - food_for_growth}
    else
      {city.population, new_food_stores}
    end

  updated_city = %{city |
    population: new_population,
    food_stores: new_food_stores
  }

  if new_population > city.population do
    broadcast_to_player(state.player_id, "city_grew", %{
      city_id: city.id,
      new_population: new_population
    })
  end

  {:noreply, %{state | city: updated_city}}
end
```

### Frontend (React)

**Components:**
- `CityPanel.tsx` - Show population, food stores, growth progress bar
- Growth notification when city grows

**Acceptance Criteria:**
- Cities accumulate food each tick
- When food threshold reached, population increases
- Growth progress visible in UI
- Growth notification pushed to client

---

## Slice 13: Technology Research

**Goal:** Players can research technologies that unlock new buildings/units.

### Backend (Phoenix)

**Database (Ecto):**
- Migration: Add `tech_tree` (JSON map) to players
- Migration: `technologies` table with definitions (or hardcoded module)

**Modules:**
```
lib/primordia/
├── game/
│   ├── technology.ex         # Tech definitions and prerequisites
│   └── research.ex           # Research logic
```

**PlayerServer Research:**
```elixir
def handle_call({:start_research, tech_name}, _from, state) do
  case Research.can_research?(state.player, tech_name) do
    {:ok, cost, duration_ms} ->
      if state.player.science >= cost do
        player = %{state.player | science: state.player.science - cost}
        Process.send_after(self(), {:complete_research, tech_name}, duration_ms)

        {:reply, {:ok, %{completes_at: ...}}, %{state | player: player}}
      else
        {:reply, {:error, :insufficient_science}, state}
      end
    {:error, reason} ->
      {:reply, {:error, reason}, state}
  end
end

def handle_info({:complete_research, tech_name}, state) do
  player = Research.complete(state.player, tech_name)
  Game.update_player(player)

  broadcast_to_player(player.id, "tech_researched", %{tech: tech_name})

  {:noreply, %{state | player: player}}
end
```

### Frontend (React)

**Components:**
- `TechTree.tsx` - Visual tech tree with nodes
- `ResearchPanel.tsx` - Current research progress
- Locked/unlocked indicators

**Acceptance Criteria:**
- Player can view tech tree
- Selecting tech starts research (consumes science)
- Research completes after duration
- Completed tech unlocks new options
- Prerequisites enforced

---

## Slice 14: Process Recovery & Fault Tolerance

**Goal:** Ensure game state survives server restarts and process crashes.

### Backend (Phoenix)

**State Persistence Strategy:**
```elixir
defmodule Primordia.Game.PlayerServer do
  # Periodic state snapshot
  def init(player_id) do
    state = load_state_from_db(player_id)
    schedule_snapshot()
    {:ok, state}
  end

  def handle_info(:snapshot, state) do
    persist_state(state)
    schedule_snapshot()
    {:noreply, state}
  end

  defp schedule_snapshot do
    Process.send_after(self(), :snapshot, 30_000)  # Every 30s
  end

  # Recover scheduled actions on restart
  defp load_state_from_db(player_id) do
    state = # ... load from DB

    # Re-schedule any pending productions
    for city <- state.cities, city.production_completes_at do
      remaining_ms = DateTime.diff(city.production_completes_at, DateTime.utc_now(), :millisecond)
      if remaining_ms > 0 do
        Process.send_after(self(), {:complete_production, city.id}, remaining_ms)
      else
        # Complete immediately if past due
        send(self(), {:complete_production, city.id})
      end
    end

    state
  end
end
```

**Supervision Configuration:**
```elixir
# In application.ex
children = [
  Primordia.Repo,
  {Phoenix.PubSub, name: Primordia.PubSub},
  PrimordiaWeb.Endpoint,
  {Registry, keys: :unique, name: Primordia.GameRegistry},
  {DynamicSupervisor, name: Primordia.GameSupervisor, strategy: :one_for_one}
]
```

**Acceptance Criteria:**
- Server can restart without losing game state
- In-progress productions resume correctly after restart
- Moving units continue to their destinations
- Player reconnect receives accurate current state

---

## Slice 15: Concurrency Testing & Race Conditions

**Goal:** Verify system handles concurrent actions correctly.

### Backend (Phoenix)

**Test Scenarios:**
```elixir
defmodule Primordia.ConcurrencyTest do
  use ExUnit.Case

  test "two players moving to same tile - first arrival wins" do
    # Setup two players with units near same tile
    # Send simultaneous move commands
    # Verify first to arrive occupies tile
    # Verify second triggers combat
  end

  test "rapid commands to same city are serialized" do
    # Send 100 production commands rapidly
    # Verify all processed in order
    # Verify no race conditions
  end

  test "player reconnect during unit movement" do
    # Start unit movement
    # Kill PlayerServer
    # Reconnect
    # Verify movement completes correctly
  end
end
```

**Load Testing:**
- Use `benchee` for performance benchmarks
- Test with simulated multiple concurrent players
- Monitor GenServer message queue lengths

**Acceptance Criteria:**
- No race conditions under concurrent load
- First-come-first-served integrity maintained
- No duplicate events or lost updates
- Performance metrics within targets

---

## Implementation Order

### Phase 1: Foundation (Slices 1-4)
1. **Slice 1:** Project setup, database, static world
2. **Slice 2:** Authentication
3. **Slice 3:** Player state and units
4. **Slice 4:** Phoenix Channels real-time connection

### Phase 2: Core Gameplay (Slices 5-9)
5. **Slice 5:** City founding
6. **Slice 6:** GenServer architecture
7. **Slice 7:** Global tick
8. **Slice 8:** Building production
9. **Slice 9:** Unit movement

### Phase 3: Strategic Depth (Slices 10-13)
10. **Slice 10:** Fog of war
11. **Slice 11:** Combat
12. **Slice 12:** City growth
13. **Slice 13:** Technology research

### Phase 4: Production Hardening (Slices 14-15)
14. **Slice 14:** Fault tolerance
15. **Slice 15:** Concurrency testing

---

## Development Tips

### Elixir/Phoenix Specifics

```bash
# Create project
mix phx.new primordia --database postgres

# Generate context and schema
mix phx.gen.context Game Player players user_id:references:users gold:integer

# Generate Channel
mix phx.gen.channel Game

# Run with IEx for debugging
iex -S mix phx.server

# Observer for process inspection
:observer.start()
```

### Testing GenServers

```elixir
# In test setup
{:ok, pid} = PlayerServer.start_link(player_id)

# Make synchronous call
result = GenServer.call(pid, {:found_city, settler_id, "Rome"})

# Assert on result
assert {:ok, city} = result
assert city.name == "Rome"
```

### Debugging Tips

- Use `IO.inspect(value, label: "debug")` liberally
- `:sys.get_state(pid)` to inspect GenServer state
- LiveDashboard at `/dev/dashboard` for real-time monitoring
- `Process.whereis(name)` to find registered processes

Each slice is independently deployable and testable, building toward a complete real-time strategy game that showcases Elixir/OTP's strengths.
