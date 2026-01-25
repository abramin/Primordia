# Primordia Architecture Guidelines

## Introduction

This document defines the architectural patterns and design principles for **Primordia**, a real-time strategy game built on Elixir/OTP. These guidelines leverage functional programming, the Actor model, and OTP supervision to build a maintainable, fault-tolerant, and highly concurrent system.

The key principles guiding our architecture are:

1. **Functional core, imperative shell** - Pure domain logic, side effects at boundaries
2. **Process-per-entity** - Each game entity (player, city, unit) as a supervised GenServer
3. **Message-passing concurrency** - No shared mutable state, all communication via messages
4. **Let it crash** - Embrace failures with supervision and recovery
5. **Event-driven updates** - Phoenix PubSub for real-time client synchronization

---

## Table of Contents

1. [Domain Model Overview](#domain-model-overview)
2. [Layer Responsibilities](#layer-responsibilities)
3. [Process Architecture](#process-architecture)
4. [Domain Logic & Pure Functions](#domain-logic--pure-functions)
5. [State Management with GenServers](#state-management-with-genservers)
6. [Event Broadcasting with PubSub](#event-broadcasting-with-pubsub)
7. [Persistence with Ecto](#persistence-with-ecto)
8. [Phoenix Channels & Real-time](#phoenix-channels--real-time)
9. [Code Organization](#code-organization)
10. [Testing Strategy](#testing-strategy)
11. [Common Patterns and Examples](#common-patterns-and-examples)

---

## Domain Model Overview

### Core Entities

Primordia's domain is organized around four primary entities, each managed by its own process:

1. **Player** - Manages player identity, resources, and technology
2. **City** - Manages city state, population, production queues
3. **Unit** - Manages unit state, movement, and combat
4. **World/Tiles** - Manages the game map, tile properties, and fog of war

### Ubiquitous Language

| Term | Definition |
|------|-----------|
| **Settler** | A unit capable of founding a new city |
| **Production Queue** | An ordered list of items (buildings/units) a city is constructing |
| **Global Tick** | A 10-second interval where resources are generated empire-wide |
| **Fog of War** | Tiles that are hidden from a player until revealed by units or cities |
| **Arrival Time** | The calculated future timestamp when a unit will reach its destination |
| **Combat Resolution** | The automatic calculation of battle outcomes when hostile units occupy the same tile |

---

## Layer Responsibilities

### Domain Layer (`lib/primordia/game/`)

**Purpose:** Contains pure business logic as stateless functions operating on data structures.

**Responsibilities:**
- Define data structures (structs) representing game state
- Implement business rules as pure functions
- Validate state transitions
- Calculate derived values (resource generation, combat outcomes)

**What belongs here:**
- Data Structs: `City`, `Unit`, `Player`, `Tile`
- Value Objects: `Coordinates`, `ResourceAmount`, `TerrainType`, `UnitType`
- Domain Logic Modules: `CityFounder`, `Movement`, `Combat`, `ResourceCalculator`
- Validation Functions: Pure functions that return `{:ok, result}` or `{:error, reason}`

**Example:**
```elixir
# lib/primordia/game/city.ex
defmodule Primordia.Game.City do
  @moduledoc """
  City data structure and pure domain logic.
  """

  defstruct [
    :id,
    :name,
    :player_id,
    :x,
    :y,
    population: 1,
    food_stores: 0,
    production_queue: [],
    buildings: [],
    production_item: nil,
    production_completes_at: nil
  ]

  @type t :: %__MODULE__{
    id: String.t(),
    name: String.t(),
    player_id: String.t(),
    x: integer(),
    y: integer(),
    population: pos_integer(),
    food_stores: non_neg_integer(),
    production_queue: list(String.t()),
    buildings: list(String.t()),
    production_item: String.t() | nil,
    production_completes_at: DateTime.t() | nil
  }

  @doc """
  Creates a new city. Pure function - no side effects.
  """
  @spec new(String.t(), String.t(), integer(), integer(), String.t()) :: t()
  def new(id, name, player_id, x, y) do
    %__MODULE__{
      id: id,
      name: name,
      player_id: player_id,
      x: x,
      y: y
    }
  end

  @doc """
  Adds a building to the production queue.
  Returns {:ok, updated_city} or {:error, reason}.
  """
  @spec add_to_queue(t(), String.t()) :: {:ok, t()} | {:error, atom()}
  def add_to_queue(%__MODULE__{production_queue: queue} = city, building_type)
      when length(queue) >= 10 do
    {:error, :queue_full}
  end

  def add_to_queue(%__MODULE__{} = city, building_type) do
    {:ok, %{city | production_queue: city.production_queue ++ [building_type]}}
  end

  @doc """
  Starts production of next item in queue.
  """
  @spec start_production(t(), DateTime.t()) :: {:ok, t(), integer()} | {:error, atom()}
  def start_production(%__MODULE__{production_queue: []} = _city, _now) do
    {:error, :empty_queue}
  end

  def start_production(%__MODULE__{production_item: item} = _city, _now)
      when not is_nil(item) do
    {:error, :already_producing}
  end

  def start_production(%__MODULE__{production_queue: [item | rest]} = city, now) do
    duration_ms = production_duration(item)
    completes_at = DateTime.add(now, duration_ms, :millisecond)

    updated_city = %{city |
      production_item: item,
      production_completes_at: completes_at,
      production_queue: rest
    }

    {:ok, updated_city, duration_ms}
  end

  @doc """
  Completes current production, adding building to city.
  """
  @spec complete_production(t()) :: {:ok, t(), String.t()} | {:error, atom()}
  def complete_production(%__MODULE__{production_item: nil}) do
    {:error, :nothing_in_production}
  end

  def complete_production(%__MODULE__{production_item: item, buildings: buildings} = city) do
    updated_city = %{city |
      buildings: [item | buildings],
      production_item: nil,
      production_completes_at: nil
    }

    {:ok, updated_city, item}
  end

  @doc """
  Applies food from a tick, potentially growing population.
  Pure function - calculates new state.
  """
  @spec apply_food_tick(t(), integer()) :: {t(), boolean()}
  def apply_food_tick(%__MODULE__{} = city, food_generated) do
    new_food_stores = city.food_stores + food_generated
    food_for_growth = city.population * 10

    if new_food_stores >= food_for_growth do
      updated_city = %{city |
        population: city.population + 1,
        food_stores: new_food_stores - food_for_growth
      }
      {updated_city, true}  # true = grew
    else
      {%{city | food_stores: new_food_stores}, false}
    end
  end

  # Private helpers
  defp production_duration("granary"), do: 120_000  # 2 minutes
  defp production_duration("barracks"), do: 180_000  # 3 minutes
  defp production_duration("walls"), do: 300_000     # 5 minutes
  defp production_duration(_), do: 60_000            # 1 minute default
end
```

### Application Layer (`lib/primordia/game/servers/`)

**Purpose:** GenServers that manage state and orchestrate domain logic.

**Responsibilities:**
- Maintain in-memory state for game entities
- Process commands by delegating to domain functions
- Schedule delayed actions with `Process.send_after/3`
- Coordinate persistence via Ecto
- Broadcast events via PubSub

**What belongs here:**
- GenServers: `PlayerServer`, `CityServer`, `UnitServer`, `TickServer`
- Supervisors: `GameSupervisor`, `PlayerSupervisor`
- Process registration and lookup utilities

**Example:**
```elixir
# lib/primordia/game/servers/city_server.ex
defmodule Primordia.Game.Servers.CityServer do
  @moduledoc """
  GenServer managing a single city's state and lifecycle.
  """

  use GenServer
  require Logger

  alias Primordia.Game.City
  alias Primordia.Repo

  # Client API

  def start_link({city_id, player_id}) do
    GenServer.start_link(__MODULE__, {city_id, player_id}, name: via_tuple(city_id))
  end

  def start_production(city_id, building_type) do
    GenServer.call(via_tuple(city_id), {:start_production, building_type})
  end

  def get_state(city_id) do
    GenServer.call(via_tuple(city_id), :get_state)
  end

  def process_tick(city_id) do
    GenServer.cast(via_tuple(city_id), :process_tick)
  end

  # Server Callbacks

  @impl true
  def init({city_id, player_id}) do
    # Load from database
    city = load_city_from_db(city_id)

    state = %{
      city: city,
      player_id: player_id
    }

    # Resume any in-progress production
    state = maybe_schedule_production(state)

    Logger.info("CityServer started for #{city.name} (#{city_id})")
    {:ok, state}
  end

  @impl true
  def handle_call({:start_production, building_type}, _from, state) do
    now = DateTime.utc_now()

    case City.start_production(state.city, now) do
      {:ok, updated_city, duration_ms} ->
        # Schedule completion
        Process.send_after(self(), :complete_production, duration_ms)

        # Persist to database
        persist_city(updated_city)

        # Broadcast event
        broadcast_production_started(state.player_id, updated_city, duration_ms)

        {:reply, {:ok, updated_city}, %{state | city: updated_city}}

      {:error, reason} ->
        {:reply, {:error, reason}, state}
    end
  end

  @impl true
  def handle_call(:get_state, _from, state) do
    {:reply, state.city, state}
  end

  @impl true
  def handle_cast(:process_tick, state) do
    food_generated = calculate_food_generation(state.city)
    {updated_city, grew?} = City.apply_food_tick(state.city, food_generated)

    # Persist
    persist_city(updated_city)

    # Broadcast growth event if population increased
    if grew? do
      broadcast_city_grew(state.player_id, updated_city)
    end

    {:noreply, %{state | city: updated_city}}
  end

  @impl true
  def handle_info(:complete_production, state) do
    case City.complete_production(state.city) do
      {:ok, updated_city, completed_building} ->
        # Persist
        persist_city(updated_city)

        # Broadcast completion
        broadcast_production_complete(state.player_id, updated_city, completed_building)

        Logger.info("#{state.city.name} completed #{completed_building}")
        {:noreply, %{state | city: updated_city}}

      {:error, reason} ->
        Logger.error("Failed to complete production: #{reason}")
        {:noreply, state}
    end
  end

  # Private Functions

  defp via_tuple(city_id) do
    {:via, Registry, {Primordia.GameRegistry, {:city, city_id}}}
  end

  defp load_city_from_db(city_id) do
    # Load from Ecto and convert to domain struct
    db_city = Repo.get!(Primordia.Game.Schemas.City, city_id)
    City.new(db_city.id, db_city.name, db_city.player_id, db_city.x, db_city.y)
    |> Map.merge(%{
      population: db_city.population,
      food_stores: db_city.food_stores,
      buildings: db_city.buildings || [],
      production_item: db_city.production_item,
      production_completes_at: db_city.production_completes_at
    })
  end

  defp persist_city(%City{} = city) do
    Primordia.Game.update_city_from_domain(city)
  end

  defp maybe_schedule_production(%{city: %{production_completes_at: nil}} = state) do
    state
  end

  defp maybe_schedule_production(%{city: city} = state) do
    remaining_ms = DateTime.diff(city.production_completes_at, DateTime.utc_now(), :millisecond)

    if remaining_ms > 0 do
      Process.send_after(self(), :complete_production, remaining_ms)
    else
      # Past due - complete immediately
      send(self(), :complete_production)
    end

    state
  end

  defp calculate_food_generation(%City{population: pop, buildings: buildings}) do
    base_food = pop * 2
    granary_bonus = if "granary" in buildings, do: pop, else: 0
    base_food + granary_bonus
  end

  defp broadcast_production_started(player_id, city, duration_ms) do
    Phoenix.PubSub.broadcast(
      Primordia.PubSub,
      "player:#{player_id}",
      {:production_started, %{
        city_id: city.id,
        item: city.production_item,
        completes_at: city.production_completes_at,
        duration_ms: duration_ms
      }}
    )
  end

  defp broadcast_production_complete(player_id, city, building) do
    Phoenix.PubSub.broadcast(
      Primordia.PubSub,
      "player:#{player_id}",
      {:production_complete, %{
        city_id: city.id,
        building: building
      }}
    )
  end

  defp broadcast_city_grew(player_id, city) do
    Phoenix.PubSub.broadcast(
      Primordia.PubSub,
      "player:#{player_id}",
      {:city_grew, %{
        city_id: city.id,
        name: city.name,
        new_population: city.population
      }}
    )
  end
end
```

### Infrastructure Layer (`lib/primordia_web/`)

**Purpose:** HTTP/WebSocket interfaces and database implementations.

**Responsibilities:**
- Phoenix Channels for WebSocket communication
- REST Controllers for non-realtime operations
- Ecto Schemas for database mapping
- Authentication plugs

**What belongs here:**
- Channels: `GameChannel`, `UserSocket`
- Controllers: `AuthController`, `FallbackController`
- JSON Views: `CityJSON`, `UnitJSON`
- Ecto Schemas: `Primordia.Game.Schemas.City`, etc.

**Example:**
```elixir
# lib/primordia_web/channels/game_channel.ex
defmodule PrimordiaWeb.GameChannel do
  @moduledoc """
  Phoenix Channel for real-time game communication.
  """

  use PrimordiaWeb, :channel

  alias Primordia.Game
  alias Primordia.Game.Servers.{PlayerServer, CityServer}

  @impl true
  def join("game:player:" <> player_id, _params, socket) do
    if socket.assigns.player_id == player_id do
      # Subscribe to player's PubSub topic
      Phoenix.PubSub.subscribe(Primordia.PubSub, "player:#{player_id}")

      # Ensure PlayerServer is running
      Game.ensure_player_server_started(player_id)

      # Send current state
      state = PlayerServer.get_full_state(player_id)
      {:ok, %{state: state}, socket}
    else
      {:error, %{reason: "unauthorized"}}
    end
  end

  @impl true
  def handle_in("found_city", %{"settler_id" => settler_id, "name" => name}, socket) do
    player_id = socket.assigns.player_id

    case Game.found_city(player_id, settler_id, name) do
      {:ok, city} ->
        broadcast!(socket, "city_founded", %{
          city: CityJSON.show(city)
        })
        {:reply, {:ok, %{city_id: city.id}}, socket}

      {:error, reason} ->
        {:reply, {:error, %{reason: to_string(reason)}}, socket}
    end
  end

  @impl true
  def handle_in("start_production", %{"city_id" => city_id, "building" => building}, socket) do
    case CityServer.start_production(city_id, building) do
      {:ok, city} ->
        {:reply, {:ok, %{completes_at: city.production_completes_at}}, socket}

      {:error, reason} ->
        {:reply, {:error, %{reason: to_string(reason)}}, socket}
    end
  end

  @impl true
  def handle_in("move_unit", %{"unit_id" => unit_id, "to" => %{"x" => x, "y" => y}}, socket) do
    player_id = socket.assigns.player_id

    case Game.move_unit(player_id, unit_id, x, y) do
      {:ok, unit} ->
        {:reply, {:ok, %{arrival_time: unit.arrival_time}}, socket}

      {:error, reason} ->
        {:reply, {:error, %{reason: to_string(reason)}}, socket}
    end
  end

  # Handle PubSub messages and forward to client
  @impl true
  def handle_info({:production_started, payload}, socket) do
    push(socket, "production_started", payload)
    {:noreply, socket}
  end

  @impl true
  def handle_info({:production_complete, payload}, socket) do
    push(socket, "production_complete", payload)
    {:noreply, socket}
  end

  @impl true
  def handle_info({:city_grew, payload}, socket) do
    push(socket, "city_grew", payload)
    {:noreply, socket}
  end

  @impl true
  def handle_info({:resources_tick, payload}, socket) do
    push(socket, "resources_tick", payload)
    {:noreply, socket}
  end

  @impl true
  def handle_info({:unit_moved, payload}, socket) do
    push(socket, "unit_moved", payload)
    {:noreply, socket}
  end
end
```

---

## Process Architecture

### Supervision Tree

```
Primordia.Application
├── Primordia.Repo
├── Phoenix.PubSub (name: Primordia.PubSub)
├── PrimordiaWeb.Endpoint
├── {Registry, name: Primordia.GameRegistry}
├── Primordia.Game.TickServer
├── Primordia.Game.WorldServer
└── Primordia.Game.GameSupervisor (DynamicSupervisor)
    ├── Primordia.Game.Servers.PlayerServer (Player 1)
    │   └── (manages cities and units in state)
    ├── Primordia.Game.Servers.PlayerServer (Player 2)
    └── ...
```

### Process Registry

Use `Registry` for named process lookup:

```elixir
# lib/primordia/game/registry.ex
defmodule Primordia.Game.Registry do
  @moduledoc """
  Helpers for registering and looking up game processes.
  """

  def via_player(player_id) do
    {:via, Registry, {Primordia.GameRegistry, {:player, player_id}}}
  end

  def via_city(city_id) do
    {:via, Registry, {Primordia.GameRegistry, {:city, city_id}}}
  end

  def via_unit(unit_id) do
    {:via, Registry, {Primordia.GameRegistry, {:unit, unit_id}}}
  end

  def lookup_player(player_id) do
    case Registry.lookup(Primordia.GameRegistry, {:player, player_id}) do
      [{pid, _}] -> {:ok, pid}
      [] -> {:error, :not_found}
    end
  end
end
```

### Starting Processes

```elixir
# lib/primordia/game/game_supervisor.ex
defmodule Primordia.Game.GameSupervisor do
  use DynamicSupervisor

  def start_link(init_arg) do
    DynamicSupervisor.start_link(__MODULE__, init_arg, name: __MODULE__)
  end

  @impl true
  def init(_init_arg) do
    DynamicSupervisor.init(strategy: :one_for_one)
  end

  def start_player_server(player_id) do
    spec = {Primordia.Game.Servers.PlayerServer, player_id}
    DynamicSupervisor.start_child(__MODULE__, spec)
  end

  def ensure_player_server(player_id) do
    case Registry.lookup(Primordia.GameRegistry, {:player, player_id}) do
      [{pid, _}] -> {:ok, pid}
      [] -> start_player_server(player_id)
    end
  end
end
```

---

## Domain Logic & Pure Functions

### Separation of Concerns

Domain logic should be **pure functions** - given the same inputs, they always produce the same outputs with no side effects.

```elixir
# lib/primordia/game/combat.ex
defmodule Primordia.Game.Combat do
  @moduledoc """
  Pure combat resolution logic.
  No side effects - just calculations.
  """

  alias Primordia.Game.{Unit, Tile}

  @type combat_result :: %{
    attacker_damage: non_neg_integer(),
    defender_damage: non_neg_integer(),
    attacker_survives: boolean(),
    defender_survives: boolean(),
    winner: :attacker | :defender | :draw
  }

  @spec resolve(Unit.t(), Unit.t(), Tile.t()) :: combat_result()
  def resolve(%Unit{} = attacker, %Unit{} = defender, %Tile{} = terrain) do
    terrain_bonus = terrain_defense_multiplier(terrain.terrain_type)

    # Calculate effective combat values
    attack_power = attacker.attack * (0.5 + :rand.uniform() * 0.5)
    defense_power = defender.defense * terrain_bonus * (0.5 + :rand.uniform() * 0.5)

    # Calculate damage
    damage_to_defender = max(0, round(attack_power - defense_power * 0.3))
    damage_to_attacker = max(0, round(defense_power * 0.5 - attack_power * 0.2))

    attacker_survives = attacker.health > damage_to_attacker
    defender_survives = defender.health > damage_to_defender

    winner = cond do
      not defender_survives and attacker_survives -> :attacker
      not attacker_survives and defender_survives -> :defender
      not attacker_survives and not defender_survives -> :draw
      true -> :draw
    end

    %{
      attacker_damage: damage_to_attacker,
      defender_damage: damage_to_defender,
      attacker_survives: attacker_survives,
      defender_survives: defender_survives,
      winner: winner
    }
  end

  @spec apply_damage(Unit.t(), non_neg_integer()) :: Unit.t()
  def apply_damage(%Unit{health: health} = unit, damage) do
    %{unit | health: max(0, health - damage)}
  end

  defp terrain_defense_multiplier(:plains), do: 1.0
  defp terrain_defense_multiplier(:forest), do: 1.25
  defp terrain_defense_multiplier(:mountains), do: 1.5
  defp terrain_defense_multiplier(:water), do: 0.5
  defp terrain_defense_multiplier(_), do: 1.0
end
```

### Why Pure Functions Matter

1. **Testability** - Easy to unit test without mocking
2. **Predictability** - No hidden state or side effects
3. **Composability** - Functions can be combined freely
4. **Process Safety** - Safe to call from any process

---

## State Management with GenServers

### GenServer Best Practices

#### 1. Keep State Minimal

Store only what's needed; derive the rest:

```elixir
# Good - minimal state
defmodule PlayerServer do
  def init(player_id) do
    {:ok, %{
      player: load_player(player_id),
      cities: load_cities(player_id),
      units: load_units(player_id)
    }}
  end
end

# Avoid - derived data in state
# Don't store calculated values that can be derived
```

#### 2. Handle Calls vs Casts

- Use `call` when you need a response
- Use `cast` for fire-and-forget operations
- Use `info` for internal messages and timers

```elixir
# Call - client needs response
def handle_call(:get_state, _from, state) do
  {:reply, state, state}
end

# Cast - fire and forget
def handle_cast(:process_tick, state) do
  new_state = do_tick(state)
  {:noreply, new_state}
end

# Info - internal/scheduled messages
def handle_info(:complete_production, state) do
  new_state = complete_current_production(state)
  {:noreply, new_state}
end
```

#### 3. Delegate to Pure Functions

GenServer callbacks should be thin - delegate to domain modules:

```elixir
def handle_call({:move_unit, dest_x, dest_y}, _from, state) do
  # Delegate to pure domain function
  case Movement.calculate_path(state.unit, dest_x, dest_y, state.world) do
    {:ok, path, arrival_time} ->
      # Update state
      updated_unit = %{state.unit |
        path: path,
        arrival_time: arrival_time
      }

      # Schedule arrival
      ms_until_arrival = DateTime.diff(arrival_time, DateTime.utc_now(), :millisecond)
      Process.send_after(self(), :arrive, ms_until_arrival)

      # Persist
      persist_unit(updated_unit)

      {:reply, {:ok, updated_unit}, %{state | unit: updated_unit}}

    {:error, reason} ->
      {:reply, {:error, reason}, state}
  end
end
```

---

## Event Broadcasting with PubSub

### Topic Design

```elixir
# Player-specific events
"player:#{player_id}"

# World region events (for future scaling)
"world:region:#{region_id}"

# Global events
"game:global"
```

### Publishing Events

```elixir
defmodule Primordia.Game.Events do
  @moduledoc """
  Centralized event broadcasting.
  """

  def broadcast_to_player(player_id, event_type, payload) do
    Phoenix.PubSub.broadcast(
      Primordia.PubSub,
      "player:#{player_id}",
      {event_type, payload}
    )
  end

  def city_founded(player_id, city) do
    broadcast_to_player(player_id, :city_founded, %{
      city_id: city.id,
      name: city.name,
      x: city.x,
      y: city.y
    })
  end

  def unit_arrived(player_id, unit) do
    broadcast_to_player(player_id, :unit_arrived, %{
      unit_id: unit.id,
      x: unit.x,
      y: unit.y
    })
  end

  def combat_occurred(attacker_player_id, defender_player_id, result) do
    payload = %{
      attacker_id: result.attacker_id,
      defender_id: result.defender_id,
      winner: result.winner,
      attacker_damage: result.attacker_damage,
      defender_damage: result.defender_damage
    }

    broadcast_to_player(attacker_player_id, :combat_result, payload)
    broadcast_to_player(defender_player_id, :combat_result, payload)
  end
end
```

### Subscribing in Channels

```elixir
def join("game:player:" <> player_id, _params, socket) do
  # Subscribe to player's topic
  Phoenix.PubSub.subscribe(Primordia.PubSub, "player:#{player_id}")
  {:ok, socket}
end

# Forward PubSub messages to WebSocket
def handle_info({event_type, payload}, socket) do
  push(socket, to_string(event_type), payload)
  {:noreply, socket}
end
```

---

## Persistence with Ecto

### Schema vs Domain Struct

Keep Ecto schemas separate from domain structs:

```elixir
# lib/primordia/game/schemas/city.ex - Ecto Schema (Infrastructure)
defmodule Primordia.Game.Schemas.City do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  schema "cities" do
    field :name, :string
    field :x, :integer
    field :y, :integer
    field :population, :integer, default: 1
    field :food_stores, :integer, default: 0
    field :buildings, {:array, :string}, default: []
    field :production_item, :string
    field :production_completes_at, :utc_datetime

    belongs_to :player, Primordia.Game.Schemas.Player, type: :binary_id

    timestamps()
  end

  def changeset(city, attrs) do
    city
    |> cast(attrs, [:name, :x, :y, :population, :food_stores, :buildings,
                    :production_item, :production_completes_at, :player_id])
    |> validate_required([:name, :x, :y, :player_id])
    |> validate_length(:name, min: 1, max: 50)
  end
end

# lib/primordia/game/city.ex - Domain Struct (Domain)
defmodule Primordia.Game.City do
  # Pure domain logic, no Ecto
  defstruct [:id, :name, :player_id, :x, :y, ...]
end
```

### Context Module

```elixir
# lib/primordia/game.ex
defmodule Primordia.Game do
  @moduledoc """
  The Game context - public API for game operations.
  """

  alias Primordia.Repo
  alias Primordia.Game.Schemas
  alias Primordia.Game.{City, Unit, Player}
  alias Primordia.Game.Servers.{PlayerServer, CityServer}
  alias Primordia.Game.GameSupervisor

  # --- Player Operations ---

  def get_player!(player_id) do
    Schemas.Player
    |> Repo.get!(player_id)
    |> schema_to_domain()
  end

  def create_player(user_id) do
    %Schemas.Player{}
    |> Schemas.Player.changeset(%{user_id: user_id, gold: 100, food: 50, science: 0})
    |> Repo.insert()
  end

  # --- City Operations ---

  def found_city(player_id, settler_id, city_name) do
    # This coordinates between multiple operations
    Repo.transaction(fn ->
      # Get settler
      settler = Repo.get!(Schemas.Unit, settler_id)

      # Validate ownership
      if settler.player_id != player_id do
        Repo.rollback(:not_owner)
      end

      # Validate settler type
      if settler.unit_type != "settler" do
        Repo.rollback(:not_settler)
      end

      # Create city
      {:ok, city} = %Schemas.City{}
      |> Schemas.City.changeset(%{
        name: city_name,
        player_id: player_id,
        x: settler.x,
        y: settler.y
      })
      |> Repo.insert()

      # Delete settler
      Repo.delete!(settler)

      # Start CityServer
      {:ok, _pid} = GameSupervisor.start_city_server(city.id, player_id)

      # Broadcast event
      Events.city_founded(player_id, city)

      city
    end)
  end

  def update_city_from_domain(%City{} = city) do
    Schemas.City
    |> Repo.get!(city.id)
    |> Schemas.City.changeset(%{
      population: city.population,
      food_stores: city.food_stores,
      buildings: city.buildings,
      production_item: city.production_item,
      production_completes_at: city.production_completes_at
    })
    |> Repo.update()
  end

  # --- Unit Operations ---

  def move_unit(player_id, unit_id, dest_x, dest_y) do
    # Delegate to UnitServer
    case Registry.lookup(Primordia.GameRegistry, {:unit, unit_id}) do
      [{pid, _}] ->
        GenServer.call(pid, {:move_to, dest_x, dest_y})
      [] ->
        {:error, :unit_not_found}
    end
  end

  # --- Process Management ---

  def ensure_player_server_started(player_id) do
    GameSupervisor.ensure_player_server(player_id)
  end

  # --- Conversion Helpers ---

  defp schema_to_domain(%Schemas.City{} = schema) do
    %City{
      id: schema.id,
      name: schema.name,
      player_id: schema.player_id,
      x: schema.x,
      y: schema.y,
      population: schema.population,
      food_stores: schema.food_stores,
      buildings: schema.buildings || [],
      production_item: schema.production_item,
      production_completes_at: schema.production_completes_at
    }
  end
end
```

### Transaction Patterns

Use `Ecto.Multi` for complex operations:

```elixir
def found_city(player_id, settler_id, city_name) do
  Multi.new()
  |> Multi.run(:settler, fn repo, _ ->
    case repo.get(Schemas.Unit, settler_id) do
      nil -> {:error, :settler_not_found}
      settler -> {:ok, settler}
    end
  end)
  |> Multi.run(:validate, fn _repo, %{settler: settler} ->
    cond do
      settler.player_id != player_id -> {:error, :not_owner}
      settler.unit_type != "settler" -> {:error, :not_settler}
      true -> {:ok, :valid}
    end
  end)
  |> Multi.insert(:city, fn %{settler: settler} ->
    Schemas.City.changeset(%Schemas.City{}, %{
      name: city_name,
      player_id: player_id,
      x: settler.x,
      y: settler.y
    })
  end)
  |> Multi.delete(:delete_settler, fn %{settler: settler} -> settler end)
  |> Repo.transaction()
  |> case do
    {:ok, %{city: city}} -> {:ok, city}
    {:error, _step, reason, _changes} -> {:error, reason}
  end
end
```

---

## Code Organization

```
primordia/
├── lib/
│   ├── primordia/
│   │   ├── application.ex           # OTP Application, supervision tree
│   │   ├── repo.ex                  # Ecto Repo
│   │   │
│   │   ├── accounts/                # User authentication context
│   │   │   ├── accounts.ex          # Context module
│   │   │   ├── user.ex              # User schema
│   │   │   └── guardian.ex          # JWT configuration
│   │   │
│   │   └── game/                    # Game context
│   │       ├── game.ex              # Context module (public API)
│   │       ├── events.ex            # PubSub event helpers
│   │       │
│   │       ├── schemas/             # Ecto schemas (persistence)
│   │       │   ├── player.ex
│   │       │   ├── city.ex
│   │       │   ├── unit.ex
│   │       │   └── tile.ex
│   │       │
│   │       ├── city.ex              # Domain struct + pure logic
│   │       ├── unit.ex              # Domain struct + pure logic
│   │       ├── player.ex            # Domain struct + pure logic
│   │       ├── tile.ex              # Domain struct + pure logic
│   │       │
│   │       ├── combat.ex            # Combat resolution (pure)
│   │       ├── movement.ex          # Pathfinding (pure)
│   │       ├── resources.ex         # Resource calculation (pure)
│   │       ├── fog_of_war.ex        # Vision calculation (pure)
│   │       │
│   │       ├── servers/             # GenServers (stateful)
│   │       │   ├── game_supervisor.ex
│   │       │   ├── player_server.ex
│   │       │   ├── city_server.ex
│   │       │   ├── unit_server.ex
│   │       │   ├── tick_server.ex
│   │       │   └── world_server.ex
│   │       │
│   │       └── registry.ex          # Process registry helpers
│   │
│   └── primordia_web/
│       ├── endpoint.ex
│       ├── router.ex
│       │
│       ├── channels/
│       │   ├── user_socket.ex
│       │   └── game_channel.ex
│       │
│       ├── controllers/
│       │   ├── auth_controller.ex
│       │   └── fallback_controller.ex
│       │
│       └── json/
│           ├── city_json.ex
│           ├── unit_json.ex
│           └── player_json.ex
│
├── priv/
│   └── repo/
│       ├── migrations/
│       └── seeds.exs
│
└── test/
    ├── primordia/
    │   ├── game/
    │   │   ├── city_test.exs        # Domain logic tests
    │   │   ├── combat_test.exs
    │   │   └── movement_test.exs
    │   └── game_test.exs            # Context tests
    │
    ├── primordia_web/
    │   └── channels/
    │       └── game_channel_test.exs
    │
    └── support/
        └── fixtures.ex
```

---

## Testing Strategy

### Unit Tests (Domain Logic)

Test pure functions in isolation:

```elixir
# test/primordia/game/city_test.exs
defmodule Primordia.Game.CityTest do
  use ExUnit.Case, async: true

  alias Primordia.Game.City

  describe "add_to_queue/2" do
    test "adds building to empty queue" do
      city = City.new("1", "Rome", "player1", 10, 10)

      assert {:ok, updated} = City.add_to_queue(city, "granary")
      assert updated.production_queue == ["granary"]
    end

    test "returns error when queue is full" do
      city = %City{
        City.new("1", "Rome", "player1", 10, 10) |
        production_queue: Enum.map(1..10, &"building#{&1}")
      }

      assert {:error, :queue_full} = City.add_to_queue(city, "another")
    end
  end

  describe "apply_food_tick/2" do
    test "accumulates food without growth" do
      city = City.new("1", "Rome", "player1", 10, 10)

      {updated, grew?} = City.apply_food_tick(city, 5)

      assert updated.food_stores == 5
      assert updated.population == 1
      assert grew? == false
    end

    test "grows population when food threshold reached" do
      city = %City{City.new("1", "Rome", "player1", 10, 10) | food_stores: 8}

      {updated, grew?} = City.apply_food_tick(city, 5)

      assert updated.population == 2
      assert updated.food_stores == 3  # 13 - 10 (threshold)
      assert grew? == true
    end
  end
end
```

### Integration Tests (GenServers)

```elixir
# test/primordia/game/servers/city_server_test.exs
defmodule Primordia.Game.Servers.CityServerTest do
  use Primordia.DataCase, async: false

  alias Primordia.Game.Servers.CityServer

  setup do
    # Create test player and city in database
    {:ok, player} = Primordia.Game.create_player(user_id)
    {:ok, city} = Primordia.Game.create_city(player.id, "Test City", 10, 10)

    # Start CityServer
    {:ok, pid} = CityServer.start_link({city.id, player.id})

    on_exit(fn ->
      if Process.alive?(pid), do: GenServer.stop(pid)
    end)

    %{city: city, player: player, pid: pid}
  end

  test "starts production and completes after duration", %{city: city} do
    # Start production
    assert {:ok, updated} = CityServer.start_production(city.id, "granary")
    assert updated.production_item == "granary"

    # Wait for completion (use shorter duration in test config)
    assert_receive {:production_complete, %{building: "granary"}}, 5000

    # Verify state
    state = CityServer.get_state(city.id)
    assert "granary" in state.buildings
    assert state.production_item == nil
  end
end
```

### Channel Tests

```elixir
# test/primordia_web/channels/game_channel_test.exs
defmodule PrimordiaWeb.GameChannelTest do
  use PrimordiaWeb.ChannelCase

  setup do
    {:ok, user} = create_user()
    {:ok, player} = Primordia.Game.create_player(user.id)
    {:ok, _, socket} = subscribe_and_join(socket, "game:player:#{player.id}")

    %{socket: socket, player: player}
  end

  test "found_city creates city and broadcasts", %{socket: socket, player: player} do
    # Create settler first
    {:ok, settler} = create_settler(player.id, 10, 10)

    # Send found_city command
    ref = push(socket, "found_city", %{"settler_id" => settler.id, "name" => "Rome"})

    # Assert reply
    assert_reply ref, :ok, %{city_id: city_id}

    # Assert broadcast
    assert_broadcast "city_founded", %{city: %{name: "Rome"}}
  end
end
```

---

## Common Patterns and Examples

### Pattern: Scheduled Actions with Process.send_after

```elixir
defmodule Primordia.Game.Servers.UnitServer do
  use GenServer

  def handle_call({:move_to, dest_x, dest_y}, _from, state) do
    path = calculate_path(state.unit, dest_x, dest_y)
    travel_time_ms = calculate_travel_time(path, state.unit.speed)

    # Schedule arrival
    Process.send_after(self(), :arrive, travel_time_ms)

    # Schedule intermediate position updates
    schedule_position_updates(path, state.unit.speed)

    updated_unit = %{state.unit |
      destination_x: dest_x,
      destination_y: dest_y,
      arrival_time: DateTime.add(DateTime.utc_now(), travel_time_ms, :millisecond)
    }

    {:reply, {:ok, updated_unit}, %{state | unit: updated_unit}}
  end

  def handle_info({:position_update, {x, y}}, state) do
    updated_unit = %{state.unit | x: x, y: y}
    Events.unit_moved(state.player_id, updated_unit)
    {:noreply, %{state | unit: updated_unit}}
  end

  def handle_info(:arrive, state) do
    updated_unit = %{state.unit |
      x: state.unit.destination_x,
      y: state.unit.destination_y,
      destination_x: nil,
      destination_y: nil,
      arrival_time: nil
    }

    # Check for combat
    case check_for_enemies(updated_unit) do
      nil -> :ok
      enemy -> initiate_combat(updated_unit, enemy)
    end

    Events.unit_arrived(state.player_id, updated_unit)
    {:noreply, %{state | unit: updated_unit}}
  end

  defp schedule_position_updates(path, speed) do
    path
    |> Enum.with_index()
    |> Enum.each(fn {{x, y}, index} ->
      delay = index * speed_to_ms_per_tile(speed)
      Process.send_after(self(), {:position_update, {x, y}}, delay)
    end)
  end
end
```

### Pattern: Global Tick Coordination

```elixir
defmodule Primordia.Game.Servers.TickServer do
  use GenServer

  @tick_interval_ms 10_000

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{tick_count: 0}, name: __MODULE__)
  end

  def init(state) do
    schedule_tick()
    {:ok, state}
  end

  def handle_info(:tick, state) do
    tick_count = state.tick_count + 1

    # Notify all player servers
    Registry.dispatch(Primordia.GameRegistry, :players, fn entries ->
      for {pid, _} <- entries do
        GenServer.cast(pid, {:process_tick, tick_count})
      end
    end)

    # Broadcast global tick event
    Phoenix.PubSub.broadcast(Primordia.PubSub, "game:global", {:tick, tick_count})

    schedule_tick()
    {:noreply, %{state | tick_count: tick_count}}
  end

  defp schedule_tick do
    Process.send_after(self(), :tick, @tick_interval_ms)
  end
end
```

### Pattern: Fault Recovery

```elixir
defmodule Primordia.Game.Servers.PlayerServer do
  use GenServer
  require Logger

  @snapshot_interval_ms 30_000

  def init(player_id) do
    # Load state from database
    state = load_from_database(player_id)

    # Re-schedule any pending timed actions
    state = recover_scheduled_actions(state)

    # Schedule periodic snapshots
    schedule_snapshot()

    Logger.info("PlayerServer started for #{player_id}")
    {:ok, state}
  end

  def handle_info(:snapshot, state) do
    persist_state(state)
    schedule_snapshot()
    {:noreply, state}
  end

  defp recover_scheduled_actions(state) do
    now = DateTime.utc_now()

    # Recover city productions
    Enum.each(state.cities, fn city ->
      if city.production_completes_at do
        remaining = DateTime.diff(city.production_completes_at, now, :millisecond)
        if remaining > 0 do
          Process.send_after(self(), {:complete_city_production, city.id}, remaining)
        else
          send(self(), {:complete_city_production, city.id})
        end
      end
    end)

    # Recover unit movements
    Enum.each(state.units, fn unit ->
      if unit.arrival_time do
        remaining = DateTime.diff(unit.arrival_time, now, :millisecond)
        if remaining > 0 do
          Process.send_after(self(), {:unit_arrive, unit.id}, remaining)
        else
          send(self(), {:unit_arrive, unit.id})
        end
      end
    end)

    state
  end

  defp schedule_snapshot do
    Process.send_after(self(), :snapshot, @snapshot_interval_ms)
  end
end
```

### Pattern: Concurrency with GenServer Mailbox

The Actor model naturally serializes concurrent requests:

```elixir
# Two players sending commands to the same city
# are automatically serialized by GenServer mailbox

# Player 1 sends "build granary"
CityServer.start_production(city_id, "granary")

# Player 2 sends "build barracks" at nearly the same time
CityServer.start_production(city_id, "barracks")

# The GenServer processes these one at a time in order received
# No race conditions possible - this is the power of the Actor model
```

---

## Key Takeaways

1. **Functional core, imperative shell** - Pure domain logic, side effects in GenServers
2. **Process-per-entity** - Each player/city/unit gets its own supervised process
3. **Message serialization** - GenServer mailboxes eliminate race conditions naturally
4. **Let it crash** - Use supervisors; don't over-handle errors
5. **PubSub for events** - Decouple components with broadcast/subscribe
6. **Ecto for persistence** - Keep schemas separate from domain structs
7. **Test at all levels** - Unit tests for pure functions, integration for GenServers
8. **Schedule with send_after** - No external job queue needed for delayed actions
9. **Registry for lookup** - Dynamic process discovery without global state
10. **Recover on restart** - Load state from DB, re-schedule pending actions

---

## Getting Started

```bash
# Create project
mix phx.new primordia --database postgres

# Set up database
mix ecto.create
mix ecto.migrate

# Generate context
mix phx.gen.context Game City cities name:string x:integer y:integer player_id:references:players

# Run server
iex -S mix phx.server

# Open observer for process debugging
:observer.start()
```

---

**Remember:** Elixir and OTP give us powerful primitives for concurrent, fault-tolerant systems. The architecture should feel natural - processes that communicate via messages, supervisors that restart failed processes, and pure functions that make testing easy. Start simple and add complexity only when needed.
