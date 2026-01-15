# Primordia Architecture Guidelines

## Introduction

This document defines the architectural patterns and design principles for **Primordia**, a real-time strategy game. These guidelines are based on Domain-Driven Design (DDD), CQRS, and event-driven architecture patterns to ensure the system is maintainable, scalable, and robust.

The key principles guiding our architecture are:

1. **Rich domain models** that encapsulate business logic
2. **Clear separation** between command (write) and query (read) responsibilities
3. **Event-driven architecture** for asynchronous game state evolution
4. **Transactional consistency** at aggregate boundaries
5. **Domain events** as the primary mechanism for system integration

---

## Table of Contents

1. [Domain Model Overview](#domain-model-overview)
2. [Layer Responsibilities](#layer-responsibilities)
3. [Aggregate Roots and Boundaries](#aggregate-roots-and-boundaries)
4. [Domain Events](#domain-events)
5. [Command and Query Separation (CQRS)](#command-and-query-separation-cqrs)
6. [Transaction Management](#transaction-management)
7. [Asynchronous Processing](#asynchronous-processing)
8. [Code Organization](#code-organization)
9. [Testing Strategy](#testing-strategy)
10. [Common Patterns and Examples](#common-patterns-and-examples)

---

## Domain Model Overview

### Core Aggregates

Primordia's domain is organized around four primary aggregates:

1. **Player Aggregate** - Manages player identity, resources, and technology
2. **City Aggregate** - Manages city state, population, production queues
3. **Unit Aggregate** - Manages unit state, movement, and combat
4. **World/Tile Aggregate** - Manages the game map, tile properties, and fog of war

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

### Domain Layer

**Purpose:** Contains the core business logic, entities, value objects, domain services, and domain events.

**Responsibilities:**
- Define and enforce business rules and invariants
- Model the game's core concepts (City, Unit, Player, Tile)
- Create and record domain events when significant business actions occur
- Remain free of infrastructure concerns (no database, no HTTP, no external APIs)

**What belongs here:**
- Entities: `City`, `Unit`, `Player`, `Tile`
- Value Objects: `Coordinates`, `ResourceAmount`, `TileType`, `UnitType`
- Domain Services: `CityFounder`, `UnitMover`, `CombatResolver`, `ProductionCalculator`
- Domain Events: `CityFoundedEvent`, `UnitMovedEvent`, `CombatOccurredEvent`, `BuildingCompletedEvent`
- Repository Interfaces: `CityRepository`, `UnitRepository`, `PlayerRepository`

**Example:**
```python
# domain/entities/city.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
from domain.value_objects import Coordinates, ResourceAmount
from domain.events import CityFoundedEvent, BuildingCompletedEvent

class City(AggregateRoot):
    """
    City aggregate root - manages city state, production, and growth.
    """

    def __init__(
        self,
        city_id: str,
        name: str,
        owner_id: str,
        coordinates: Coordinates,
        population: int = 1,
        food_stores: int = 0
    ):
        super().__init__()
        self.__city_id = city_id
        self.__name = name
        self.__owner_id = owner_id
        self.__coordinates = coordinates
        self.__population = population
        self.__food_stores = food_stores
        self.__production_queue: list[ProductionItem] = []
        self.__completed_buildings: list[str] = []

    @classmethod
    def found(
        cls,
        city_id: str,
        name: str,
        owner_id: str,
        coordinates: Coordinates
    ) -> "City":
        """
        Factory method to found a new city.
        Records CityFoundedEvent.
        """
        city = cls(city_id, name, owner_id, coordinates)
        city.record(CityFoundedEvent(
            city_id=city_id,
            name=name,
            owner_id=owner_id,
            x=coordinates.x,
            y=coordinates.y,
            founded_at=datetime.utcnow().isoformat()
        ))
        return city

    def add_production_item(self, item: ProductionItem) -> None:
        """
        Adds a building or unit to the production queue.
        Validates that the city is not at production capacity.
        """
        if len(self.__production_queue) >= 10:
            raise ProductionQueueFullError("Cannot add more than 10 items to queue")

        self.__production_queue.append(item)

    def complete_production(self, item_type: str) -> None:
        """
        Marks a production item as completed.
        Records BuildingCompletedEvent or UnitCompletedEvent.
        """
        if not self.__production_queue:
            raise NothingInProductionError("No items in production queue")

        completed_item = self.__production_queue.pop(0)

        if completed_item.item_type != item_type:
            raise InvalidProductionStateError(
                f"Expected {completed_item.item_type}, got {item_type}"
            )

        if completed_item.is_building:
            self.__completed_buildings.append(item_type)
            self.record(BuildingCompletedEvent(
                city_id=self.__city_id,
                building_type=item_type,
                completed_at=datetime.utcnow().isoformat()
            ))

    def apply_food_growth(self, food_amount: int) -> None:
        """
        Adds food to stores and potentially grows population.
        """
        self.__food_stores += food_amount

        food_needed_for_growth = self.__population * 10
        if self.__food_stores >= food_needed_for_growth:
            self.__food_stores -= food_needed_for_growth
            self.__population += 1

    # Getters for read access
    def city_id(self) -> str:
        return self.__city_id

    def name(self) -> str:
        return self.__name

    def owner_id(self) -> str:
        return self.__owner_id

    def population(self) -> int:
        return self.__population
```

### Application Layer

**Purpose:** Orchestrates use cases, manages transactions, and coordinates domain objects.

**Responsibilities:**
- Handle commands and queries via command/query handlers
- Manage database transactions using `UnitOfWork`
- Persist entities via repositories
- Publish domain events via the event bus
- Delegate all business logic to the domain layer

**What belongs here:**
- Command Handlers: `FoundCityCommandHandler`, `MoveUnitCommandHandler`, `StartProductionCommandHandler`
- Query Handlers: `GetPlayerCitiesQueryHandler`, `GetVisibleTilesQueryHandler`
- Commands: `FoundCityCommand`, `MoveUnitCommand`, `StartProductionCommand`
- Queries: `GetPlayerCitiesQuery`, `GetVisibleTilesQuery`

**Example:**
```python
# application/commands/found_city_command.py
from dataclasses import dataclass

@dataclass
class FoundCityCommand:
    """
    Command to found a new city at specified coordinates.
    All fields must be JSON-serializable primitives.
    """
    player_id: str
    settler_unit_id: str
    city_name: str
    x: int
    y: int


# application/handlers/found_city_command_handler.py
class FoundCityCommandHandler:
    """
    Handles the FoundCityCommand.

    Responsibilities:
    - Validate the settler exists and belongs to the player
    - Validate the location is valid for city founding
    - Create the city via domain service
    - Remove the settler unit
    - Persist changes within a transaction
    - Publish domain events
    """

    def __init__(
        self,
        city_founder: CityFounder,
        city_repository: CityRepository,
        unit_repository: UnitRepository,
        player_repository: PlayerRepository,
        event_bus: EventBus,
        unit_of_work: UnitOfWork,
        logger: Logger
    ):
        self.__city_founder = city_founder
        self.__city_repository = city_repository
        self.__unit_repository = unit_repository
        self.__player_repository = player_repository
        self.__event_bus = event_bus
        self.__unit_of_work = unit_of_work
        self.__logger = logger

    def handle(self, command: FoundCityCommand) -> None:
        """
        Executes the command to found a city.
        All side effects happen within a single transaction.
        """
        # Fetch required aggregates
        player = self.__player_repository.find_or_fail_by_id(command.player_id)
        settler = self.__unit_repository.find_or_fail_by_id(command.settler_unit_id)

        # Validate ownership
        if settler.owner_id() != player.player_id():
            raise UnauthorizedActionError("Settler does not belong to player")

        # Use domain service to create city and get events
        coordinates = Coordinates(command.x, command.y)
        city = self.__city_founder.found_city(
            city_name=command.city_name,
            owner=player,
            settler=settler,
            coordinates=coordinates
        )

        # Settler is consumed when founding city
        settler.consume()

        # Persist all changes and publish events in one transaction
        with self.__unit_of_work():
            self.__city_repository.save(city)
            self.__unit_repository.save(settler)  # Marks as consumed
            self.__event_bus.bulk_publish(city.pull_events())

        self.__logger.info(f"City {city.name()} founded at ({command.x}, {command.y})")
```

### Infrastructure Layer

**Purpose:** Provides concrete implementations of interfaces defined in the domain.

**Responsibilities:**
- Implement repository interfaces using specific persistence technology (PostgreSQL, Redis, etc.)
- Implement event bus using message queue (RabbitMQ, Kafka, etc.)
- Provide HTTP controllers/API views
- Manage database connections and sessions
- Handle serialization/deserialization

**What belongs here:**
- Repository Implementations: `PostgresCityRepository`, `PostgresUnitRepository`
- Event Bus Implementation: `RabbitMQEventBus`
- API Controllers: `FoundCityAPIView`, `MoveUnitAPIView`
- Database Models (ORM): `CityModel`, `UnitModel` (Django/SQLAlchemy models)
- External Service Adapters

**Example:**
```python
# infrastructure/persistence/postgres_city_repository.py
from domain.repositories import CityRepository
from domain.entities import City
from infrastructure.models import CityModel

class PostgresCityRepository(CityRepository):
    """
    Concrete implementation of CityRepository using PostgreSQL.
    """

    def save(self, city: City) -> None:
        """Persists city to database."""
        city_model = CityModel.objects.get(id=city.city_id())
        city_model.name = city.name()
        city_model.population = city.population()
        city_model.food_stores = city.food_stores()
        # ... map other fields
        city_model.save()

    def find_by_id(self, city_id: str) -> Optional[City]:
        """Retrieves city by ID, returns None if not found."""
        try:
            city_model = CityModel.objects.get(id=city_id)
            return self.__to_domain(city_model)
        except CityModel.DoesNotExist:
            return None

    def find_or_fail_by_id(self, city_id: str) -> City:
        """Retrieves city by ID, raises exception if not found."""
        city = self.find_by_id(city_id)
        if city is None:
            raise CityNotFoundError(f"City {city_id} not found")
        return city

    def __to_domain(self, city_model: CityModel) -> City:
        """Maps ORM model to domain entity."""
        return City(
            city_id=str(city_model.id),
            name=city_model.name,
            owner_id=str(city_model.owner_id),
            coordinates=Coordinates(city_model.x, city_model.y),
            population=city_model.population,
            food_stores=city_model.food_stores
        )
```

---

## Aggregate Roots and Boundaries

### What is an Aggregate?

An **Aggregate** is a cluster of domain objects (entities and value objects) that are treated as a single unit for data changes. Each aggregate has a root entity called the **Aggregate Root**, which is the only entry point for modifying anything inside the aggregate.

### Primordia's Aggregates

#### 1. Player Aggregate

**Root:** `Player`

**Responsibilities:**
- Manage player identity and authentication state
- Track empire-wide resources (gold, science, food)
- Manage technology tree progression
- Track diplomatic relations (future)

**Invariants:**
- Resources cannot be negative
- Technology prerequisites must be met before unlocking new tech
- Player username must be unique

**Example:**
```python
class Player(AggregateRoot):
    def __init__(
        self,
        player_id: str,
        username: str,
        gold: int = 100,
        science: int = 0
    ):
        super().__init__()
        self.__player_id = player_id
        self.__username = username
        self.__gold = gold
        self.__science = science
        self.__researched_technologies: set[str] = set()

    def spend_gold(self, amount: int) -> None:
        """Deducts gold, enforcing non-negative constraint."""
        if self.__gold < amount:
            raise InsufficientResourcesError(
                f"Need {amount} gold, have {self.__gold}"
            )
        self.__gold -= amount

    def add_gold(self, amount: int) -> None:
        """Adds gold from resource generation."""
        self.__gold += amount

    def research_technology(self, tech_name: str, cost: int) -> None:
        """
        Researches a new technology.
        Validates prerequisites and science cost.
        """
        if tech_name in self.__researched_technologies:
            raise TechnologyAlreadyResearchedError(tech_name)

        # Domain logic: validate prerequisites
        prerequisites = self.__get_tech_prerequisites(tech_name)
        if not prerequisites.issubset(self.__researched_technologies):
            raise PrerequisitesNotMetError(
                f"{tech_name} requires {prerequisites}"
            )

        self.spend_science(cost)
        self.__researched_technologies.add(tech_name)

        self.record(TechnologyResearchedEvent(
            player_id=self.__player_id,
            technology=tech_name,
            researched_at=datetime.utcnow().isoformat()
        ))
```

#### 2. City Aggregate

**Root:** `City`

**Responsibilities:**
- Manage city growth and population
- Manage production queue (buildings and units)
- Calculate resource generation (food, production)
- Manage city borders and worked tiles

**Invariants:**
- Population must be positive
- Production queue cannot exceed capacity
- Cannot build same building twice
- Food stores cannot be negative

**Bounded within:** Single city - does not manage units created by the city

#### 3. Unit Aggregate

**Root:** `Unit`

**Responsibilities:**
- Manage unit position and movement
- Track movement paths and arrival times
- Handle combat initiation
- Manage unit health and experience

**Invariants:**
- Unit cannot move to invalid terrain (e.g., land unit to water)
- Health must be between 0 and max health
- Cannot move while already moving
- Destination must be within movement range

**Example:**
```python
class Unit(AggregateRoot):
    def move_to(
        self,
        destination: Coordinates,
        movement_calculator: MovementCalculator
    ) -> None:
        """
        Initiates movement to destination.
        Calculates path and arrival time.
        """
        if self.__is_moving:
            raise UnitAlreadyMovingError("Unit is already moving")

        # Use domain service to calculate movement
        path, arrival_time = movement_calculator.calculate_path(
            from_coords=self.__current_location,
            to_coords=destination,
            unit_type=self.__unit_type,
            movement_points=self.__movement_points
        )

        if not path:
            raise InvalidDestinationError("No valid path to destination")

        self.__destination = destination
        self.__arrival_time = arrival_time
        self.__is_moving = True

        self.record(UnitMovementStartedEvent(
            unit_id=self.__unit_id,
            from_x=self.__current_location.x,
            from_y=self.__current_location.y,
            to_x=destination.x,
            to_y=destination.y,
            arrival_time=arrival_time.isoformat(),
            unit_type=self.__unit_type.value
        ))

    def complete_movement(self) -> None:
        """
        Called when arrival time is reached.
        Updates position and marks movement as complete.
        """
        if not self.__is_moving:
            raise UnitNotMovingError("Unit is not currently moving")

        if datetime.utcnow() < self.__arrival_time:
            raise ArrivalTimeNotReachedError("Unit has not reached destination yet")

        self.__current_location = self.__destination
        self.__is_moving = False
        self.__destination = None
        self.__arrival_time = None

        self.record(UnitArrivedEvent(
            unit_id=self.__unit_id,
            x=self.__current_location.x,
            y=self.__current_location.y,
            arrived_at=datetime.utcnow().isoformat()
        ))
```

#### 4. World/Tile Aggregate

**Root:** `World` (or `GameMap`)

**Contains:** Collection of `Tile` entities

**Responsibilities:**
- Manage tile properties (terrain type, resources)
- Manage fog of war per player
- Handle tile visibility calculations
- Manage global world state

**Invariants:**
- Tile coordinates must be within world bounds
- Each coordinate must have exactly one tile
- Fog of war state is per-player

**Design Note:** This aggregate is potentially very large (100x100 = 10,000 tiles). Consider:
- Loading tiles in chunks/regions
- Using a separate read model for queries
- Optimizing fog of war updates

---

## Domain Events

Domain events are the primary mechanism for:
1. Communicating that something significant happened in the domain
2. Triggering asynchronous workflows
3. Maintaining eventual consistency across aggregates
4. Integrating with external systems

### Event Design Principles

1. **Events are immutable** - once created, they cannot be changed
2. **Events are self-contained** - carry all relevant context, not just IDs
3. **Events are past-tense** - `CityFoundedEvent`, not `FoundCityEvent`
4. **Events contain JSON-serializable primitives only** - strings, ints, bools, lists, dicts
5. **Events are created in the domain layer** - never in application or infrastructure
6. **Events are published in the application layer** - within the same transaction as persistence

### Core Events

```python
# domain/events/city_events.py
from dataclasses import dataclass

@dataclass
class CityFoundedEvent:
    """
    Emitted when a new city is founded.

    Consumers might:
    - Update player statistics
    - Reveal tiles around city
    - Initialize city borders
    - Send notification to player
    """
    city_id: str
    name: str
    owner_id: str
    x: int
    y: int
    founded_at: str  # ISO-8601 timestamp


@dataclass
class BuildingCompletedEvent:
    """
    Emitted when a building finishes construction.

    Consumers might:
    - Apply building effects to city
    - Start next item in production queue
    - Send notification to player
    """
    city_id: str
    building_type: str  # e.g., "granary", "barracks"
    completed_at: str


@dataclass
class PopulationGrownEvent:
    """
    Emitted when a city's population increases.

    Consumers might:
    - Update player statistics dashboard
    - Check for new citizen assignment
    - Send notification to player
    """
    city_id: str
    city_name: str
    new_population: int
    grown_at: str


# domain/events/unit_events.py
@dataclass
class UnitMovementStartedEvent:
    """
    Emitted when a unit begins moving.

    Consumers might:
    - Update fog of war along path
    - Check for potential combat encounters
    - Update UI with movement animation
    """
    unit_id: str
    owner_id: str
    unit_type: str
    from_x: int
    from_y: int
    to_x: int
    to_y: int
    arrival_time: str


@dataclass
class UnitArrivedEvent:
    """
    Emitted when a unit reaches its destination.

    Consumers might:
    - Reveal fog of war at new location
    - Check for combat with other units
    - Update tile occupation
    - Send notification to player
    """
    unit_id: str
    owner_id: str
    unit_type: str
    x: int
    y: int
    arrived_at: str


@dataclass
class CombatOccurredEvent:
    """
    Emitted when two hostile units engage in combat.

    Consumers might:
    - Update player war statistics
    - Award experience to victor
    - Send battle report to players
    - Check for unit destruction
    """
    attacker_unit_id: str
    attacker_owner_id: str
    defender_unit_id: str
    defender_owner_id: str
    location_x: int
    location_y: int
    attacker_damage_dealt: int
    defender_damage_dealt: int
    victor_unit_id: str  # or None for draw
    occurred_at: str


# domain/events/resource_events.py
@dataclass
class GlobalTickExecutedEvent:
    """
    Emitted every 10 seconds when resources are generated.

    This is a system-level event that triggers resource
    generation for all players and cities.

    Consumers might:
    - Generate resources for each city
    - Progress production queues
    - Apply upkeep costs
    """
    tick_number: int
    executed_at: str


@dataclass
class ResourcesGeneratedEvent:
    """
    Emitted when a player receives resources from the global tick.

    Consumers might:
    - Update player dashboard
    - Check for resource milestones
    - Update analytics
    """
    player_id: str
    gold_generated: int
    food_generated: int
    science_generated: int
    generated_at: str
```

### Event Handling Patterns

#### Pattern 1: Single Event, Single Handler (Most Common)

Most events should have a single, focused consumer.

```python
# application/event_handlers/unit_arrived_event_handler.py
class UnitArrivedEventHandler:
    """
    Handles UnitArrivedEvent.
    Updates fog of war and checks for combat.
    """

    def __init__(
        self,
        fog_of_war_updater: FogOfWarUpdater,
        combat_checker: CombatChecker,
        unit_repository: UnitRepository,
        world_repository: WorldRepository,
        unit_of_work: UnitOfWork
    ):
        self.__fog_of_war_updater = fog_of_war_updater
        self.__combat_checker = combat_checker
        self.__unit_repository = unit_repository
        self.__world_repository = world_repository
        self.__unit_of_work = unit_of_work

    def handle(self, event: UnitArrivedEvent) -> None:
        """Processes unit arrival."""
        unit = self.__unit_repository.find_or_fail_by_id(event.unit_id)
        world = self.__world_repository.get_world()

        # Reveal fog of war around new location
        coordinates = Coordinates(event.x, event.y)
        tiles_to_reveal = self.__fog_of_war_updater.calculate_visible_tiles(
            coordinates=coordinates,
            vision_range=unit.vision_range()
        )

        for tile_coords in tiles_to_reveal:
            world.reveal_tile_for_player(tile_coords, unit.owner_id())

        # Check for combat
        combat_result = self.__combat_checker.check_for_combat(
            unit=unit,
            location=coordinates
        )

        with self.__unit_of_work():
            self.__world_repository.save(world)
            if combat_result:
                # Combat domain service records combat events
                self.__unit_repository.save(combat_result.attacker)
                self.__unit_repository.save(combat_result.defender)
```

#### Pattern 2: Async Command Pattern with "...Requested" Events

For fire-and-forget operations that need to happen asynchronously.

```python
# domain/events/notification_events.py
@dataclass
class PlayerNotificationRequestedEvent:
    """
    Pseudo-command event requesting a notification be sent.
    This is a bridge pattern until we have async command bus.

    MUST have exactly one consumer.
    """
    player_id: str
    notification_type: str  # e.g., "city_founded", "unit_attacked"
    message: str
    data: dict  # Additional context


# application/event_handlers/player_notification_requested_handler.py
class PlayerNotificationRequestedEventHandler:
    """
    THE ONLY consumer of PlayerNotificationRequestedEvent.
    Acts as async command handler.
    """

    def handle(self, event: PlayerNotificationRequestedEvent) -> None:
        """Sends notification to player via websocket or push notification."""
        # This may involve calling external services
        # Does NOT need to be in a transaction
        ...
```

---

## Command and Query Separation (CQRS)

Primordia strictly separates **Commands** (write operations) from **Queries** (read operations).

### Commands

Commands represent **intent to change state**. They are imperatively named and contain all data needed to perform the action.

**Characteristics:**
- Imperative naming: `FoundCityCommand`, `MoveUnitCommand`, `ResearchTechnologyCommand`
- Contain only JSON-serializable primitives
- Processed by Command Handlers
- May produce side effects (write to database, publish events)
- Do NOT return data (void return type)
- Execute within a transaction

**Example:**
```python
@dataclass
class StartProductionCommand:
    """Command to add an item to a city's production queue."""
    player_id: str
    city_id: str
    item_type: str  # "granary", "warrior", etc.
    estimated_completion_time: str  # ISO-8601


class StartProductionCommandHandler:
    def handle(self, command: StartProductionCommand) -> None:
        city = self.__city_repository.find_or_fail_by_id(command.city_id)

        # Validate ownership
        if city.owner_id() != command.player_id:
            raise UnauthorizedActionError()

        # Domain logic
        production_item = ProductionItem(
            item_type=command.item_type,
            completion_time=datetime.fromisoformat(
                command.estimated_completion_time
            )
        )
        city.add_production_item(production_item)

        # Persist
        with self.__unit_of_work():
            self.__city_repository.save(city)
            self.__event_bus.bulk_publish(city.pull_events())
```

### Queries

Queries represent **requests for data**. They are interrogatively named and return view models optimized for the client.

**Characteristics:**
- Interrogative naming: `GetPlayerCitiesQuery`, `GetVisibleTilesQuery`
- Contain only JSON-serializable primitives
- Processed by Query Handlers
- Do NOT produce side effects
- Do NOT modify state
- Return view models (NOT domain entities)
- Do NOT execute within a transaction

**Use Finders, NOT Repositories** for queries.

**Example:**
```python
# application/queries/get_player_cities_query.py
@dataclass
class GetPlayerCitiesQuery:
    """Query to retrieve all cities owned by a player."""
    player_id: str


# application/finders/player_cities_finder.py
class PlayerCitiesFinder(ABC):
    """
    Finder for retrieving player cities.
    Single responsibility: find cities by player.
    """

    @abstractmethod
    def find(self, player_id: str) -> list["CityViewModel"]:
        """Returns view models, NOT domain entities."""
        pass


# infrastructure/finders/postgres_player_cities_finder.py
class PostgresPlayerCitiesFinder(PlayerCitiesFinder):
    """Concrete implementation using PostgreSQL."""

    def find(self, player_id: str) -> list[CityViewModel]:
        # Optimized query, possibly with joins
        cities = CityModel.objects.filter(owner_id=player_id).select_related(...)

        return [
            CityViewModel(
                city_id=str(city.id),
                name=city.name,
                x=city.x,
                y=city.y,
                population=city.population,
                production_queue=[...],
                food_per_turn=city.calculate_food_per_turn()
            )
            for city in cities
        ]


# application/handlers/get_player_cities_query_handler.py
@dataclass
class CityViewModel:
    """
    View model optimized for displaying city list.
    Contains denormalized/calculated data.
    """
    city_id: str
    name: str
    x: int
    y: int
    population: int
    production_queue: list[dict]
    food_per_turn: int


class GetPlayerCitiesQueryHandler:
    def __init__(self, player_cities_finder: PlayerCitiesFinder):
        self.__finder = player_cities_finder

    def handle(self, query: GetPlayerCitiesQuery) -> list[CityViewModel]:
        """
        Returns view models directly.
        No transaction needed - this is read-only.
        """
        return self.__finder.find(query.player_id)
```

### Critical Rule: Commands and Queries Must Remain Independent

❌ **NEVER** execute a query inside a command handler
❌ **NEVER** use a query result to build a command

All validation must happen inside the command handler's transaction using repositories to fetch domain entities.

```python
# ❌ WRONG: Using query result to validate command
class MoveUnitCommandHandler:
    def handle(self, command: MoveUnitCommand) -> None:
        # ❌ BAD: Using finder (query-side) in command handler
        unit_view = self.__unit_finder.find_by_id(command.unit_id)
        if unit_view.is_moving:
            raise UnitAlreadyMovingError()

        # This creates a race condition!
        # The unit's state might have changed between the query and now


# ✅ CORRECT: Fetch domain entity from repository
class MoveUnitCommandHandler:
    def handle(self, command: MoveUnitCommand) -> None:
        # ✅ GOOD: Fetch aggregate root
        with self.__unit_of_work():
            unit = self.__unit_repository.find_or_fail_by_id(command.unit_id)

            # Domain entity enforces invariants atomically
            unit.move_to(
                destination=Coordinates(command.to_x, command.to_y),
                movement_calculator=self.__movement_calculator
            )

            self.__unit_repository.save(unit)
            self.__event_bus.bulk_publish(unit.pull_events())
```

---

## Transaction Management

All database transactions **must be managed in the application layer** using the `UnitOfWork` pattern.

### Rules:

1. **Transactions live in command handlers only**
2. **Domain services never manage transactions**
3. **Infrastructure layer (controllers) never manage transactions**
4. **All side effects happen inside the transaction block**

### Pattern:

```python
class SomeCommandHandler:
    def __init__(
        self,
        repository: SomeRepository,
        event_bus: EventBus,
        unit_of_work: UnitOfWork
    ):
        self.__repository = repository
        self.__event_bus = event_bus
        self.__unit_of_work = unit_of_work

    def handle(self, command: SomeCommand) -> None:
        # 1. Fetch aggregates (outside transaction)
        entity = self.__repository.find_or_fail_by_id(command.entity_id)

        # 2. Execute domain logic (outside transaction)
        entity.do_something()

        # 3. Persist and publish in single transaction
        with self.__unit_of_work():
            self.__repository.save(entity)
            self.__event_bus.bulk_publish(entity.pull_events())
```

### Why This Matters for Primordia:

Given the real-time nature of the game, we need **perfect concurrency control**. For example:

- Two players move units to the same tile simultaneously
- The first to reach the tile should occupy it
- The second should initiate combat

Without proper transactional boundaries, we could end up with:
- Both units occupying the same tile
- Neither unit occupying the tile
- Duplicated combat events

By wrapping all persistence and event publication in a single transaction, we ensure **atomic updates** and **consistent event ordering**.

---

## Asynchronous Processing

Primordia's core feature is **continuous real-time evolution**. This requires robust asynchronous processing.

### Key Async Workflows:

1. **Global Tick** (every 10 seconds) - resource generation
2. **Unit Movement Completion** - when arrival time is reached
3. **Production Completion** - when building/unit finishes
4. **Combat Resolution** - when units meet

### Pattern: Scheduled Event Processing

```python
# application/schedulers/global_tick_scheduler.py
class GlobalTickScheduler:
    """
    Runs every 10 seconds.
    Publishes GlobalTickExecutedEvent.
    """

    def __init__(
        self,
        event_bus: EventBus,
        logger: Logger
    ):
        self.__event_bus = event_bus
        self.__logger = logger
        self.__tick_number = 0

    def execute_tick(self) -> None:
        """Called every 10 seconds by scheduler (e.g., Celery beat)."""
        self.__tick_number += 1

        event = GlobalTickExecutedEvent(
            tick_number=self.__tick_number,
            executed_at=datetime.utcnow().isoformat()
        )

        self.__event_bus.publish(event)
        self.__logger.info(f"Global tick #{self.__tick_number} executed")


# application/event_handlers/global_tick_executed_event_handler.py
class GlobalTickExecutedEventHandler:
    """
    Responds to GlobalTickExecutedEvent.
    Generates resources for all players.
    """

    def __init__(
        self,
        player_repository: PlayerRepository,
        city_repository: CityRepository,
        resource_generator: ResourceGenerator,
        event_bus: EventBus,
        unit_of_work: UnitOfWork
    ):
        self.__player_repository = player_repository
        self.__city_repository = city_repository
        self.__resource_generator = resource_generator
        self.__event_bus = event_bus
        self.__unit_of_work = unit_of_work

    def handle(self, event: GlobalTickExecutedEvent) -> None:
        """Generates resources for all active players."""
        players = self.__player_repository.find_all_active()

        for player in players:
            cities = self.__city_repository.find_by_owner(player.player_id())

            # Domain service calculates resources
            resources = self.__resource_generator.calculate_tick_resources(
                player=player,
                cities=cities
            )

            player.add_gold(resources.gold)
            player.add_science(resources.science)

            for city, food in resources.city_food.items():
                city.apply_food_growth(food)

            # Persist changes
            with self.__unit_of_work():
                self.__player_repository.save(player)
                for city in cities:
                    self.__city_repository.save(city)

                # Publish events
                self.__event_bus.bulk_publish([
                    ResourcesGeneratedEvent(
                        player_id=player.player_id(),
                        gold_generated=resources.gold,
                        food_generated=sum(resources.city_food.values()),
                        science_generated=resources.science,
                        generated_at=event.executed_at
                    )
                ])
```

### Pattern: Time-Based Event Processing

For events that should happen at a specific time (unit arrivals, production completion):

```python
# application/schedulers/timed_events_processor.py
class TimedEventsProcessor:
    """
    Runs frequently (e.g., every 1 second).
    Checks for units/production that have completed.
    """

    def __init__(
        self,
        unit_repository: UnitRepository,
        city_repository: CityRepository,
        event_bus: EventBus,
        unit_of_work: UnitOfWork
    ):
        self.__unit_repository = unit_repository
        self.__city_repository = city_repository
        self.__event_bus = event_bus
        self.__unit_of_work = unit_of_work

    def process_pending_arrivals(self) -> None:
        """Completes unit movements that have reached arrival time."""
        now = datetime.utcnow()

        # Find units that should have arrived
        moving_units = self.__unit_repository.find_units_arriving_before(now)

        for unit in moving_units:
            unit.complete_movement()

            with self.__unit_of_work():
                self.__unit_repository.save(unit)
                self.__event_bus.bulk_publish(unit.pull_events())

    def process_production_completions(self) -> None:
        """Completes production items that have finished."""
        now = datetime.utcnow()

        cities = self.__city_repository.find_cities_with_production_completing_before(now)

        for city in cities:
            # Domain logic handles completion
            city.complete_current_production()

            with self.__unit_of_work():
                self.__city_repository.save(city)
                self.__event_bus.bulk_publish(city.pull_events())
```

---

## Code Organization

```
primordia/
├── domain/
│   ├── entities/
│   │   ├── __init__.py
│   │   ├── aggregate_root.py
│   │   ├── player.py
│   │   ├── city.py
│   │   ├── unit.py
│   │   ├── tile.py
│   │   └── world.py
│   ├── value_objects/
│   │   ├── __init__.py
│   │   ├── coordinates.py
│   │   ├── tile_type.py
│   │   ├── unit_type.py
│   │   └── resource_amount.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── city_founder.py
│   │   ├── unit_mover.py
│   │   ├── combat_resolver.py
│   │   ├── resource_generator.py
│   │   └── movement_calculator.py
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── player_repository.py
│   │   ├── city_repository.py
│   │   ├── unit_repository.py
│   │   └── world_repository.py
│   ├── events/
│   │   ├── __init__.py
│   │   ├── city_events.py
│   │   ├── unit_events.py
│   │   ├── player_events.py
│   │   └── resource_events.py
│   └── exceptions/
│       ├── __init__.py
│       ├── domain_exceptions.py
│       └── validation_errors.py
│
├── application/
│   ├── commands/
│   │   ├── __init__.py
│   │   ├── found_city_command.py
│   │   ├── move_unit_command.py
│   │   ├── start_production_command.py
│   │   └── research_technology_command.py
│   ├── queries/
│   │   ├── __init__.py
│   │   ├── get_player_cities_query.py
│   │   ├── get_visible_tiles_query.py
│   │   └── get_unit_details_query.py
│   ├── handlers/
│   │   ├── command_handlers/
│   │   │   ├── __init__.py
│   │   │   ├── found_city_command_handler.py
│   │   │   ├── move_unit_command_handler.py
│   │   │   └── start_production_command_handler.py
│   │   └── query_handlers/
│   │       ├── __init__.py
│   │       ├── get_player_cities_query_handler.py
│   │       └── get_visible_tiles_query_handler.py
│   ├── event_handlers/
│   │   ├── __init__.py
│   │   ├── unit_arrived_event_handler.py
│   │   ├── global_tick_executed_event_handler.py
│   │   └── combat_occurred_event_handler.py
│   ├── finders/
│   │   ├── __init__.py
│   │   ├── player_cities_finder.py
│   │   ├── visible_tiles_finder.py
│   │   └── unit_details_finder.py
│   ├── view_models/
│   │   ├── __init__.py
│   │   ├── city_view_model.py
│   │   ├── unit_view_model.py
│   │   └── tile_view_model.py
│   └── schedulers/
│       ├── __init__.py
│       ├── global_tick_scheduler.py
│       └── timed_events_processor.py
│
├── infrastructure/
│   ├── persistence/
│   │   ├── __init__.py
│   │   ├── postgres_player_repository.py
│   │   ├── postgres_city_repository.py
│   │   ├── postgres_unit_repository.py
│   │   └── postgres_world_repository.py
│   ├── finders/
│   │   ├── __init__.py
│   │   ├── postgres_player_cities_finder.py
│   │   └── postgres_visible_tiles_finder.py
│   ├── messaging/
│   │   ├── __init__.py
│   │   ├── rabbitmq_event_bus.py
│   │   └── in_memory_event_bus.py
│   ├── transaction/
│   │   ├── __init__.py
│   │   └── django_unit_of_work.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── views/
│   │   │   ├── found_city_view.py
│   │   │   ├── move_unit_view.py
│   │   │   └── get_cities_view.py
│   │   └── serializers/
│   │       ├── city_serializer.py
│   │       └── unit_serializer.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── player_model.py
│   │   ├── city_model.py
│   │   ├── unit_model.py
│   │   └── tile_model.py
│   └── di/
│       ├── __init__.py
│       └── container.py  # Dependency injection container
│
└── tests/
    ├── unit/
    │   ├── domain/
    │   │   ├── test_city.py
    │   │   ├── test_unit.py
    │   │   └── test_player.py
    │   └── application/
    │       └── test_found_city_handler.py
    └── integration/
        ├── test_found_city_flow.py
        └── test_unit_movement_flow.py
```

---

## Testing Strategy

### Unit Tests (Domain Layer)

Domain entities should be thoroughly tested in isolation.

```python
# tests/unit/domain/test_city.py
import pytest
from domain.entities import City
from domain.value_objects import Coordinates
from domain.exceptions import ProductionQueueFullError

class TestCity:
    def test_found_city_records_event(self):
        """City.found() should record CityFoundedEvent."""
        city = City.found(
            city_id="city-123",
            name="Rome",
            owner_id="player-1",
            coordinates=Coordinates(10, 10)
        )

        events = city.pull_events()
        assert len(events) == 1
        assert events[0].city_id == "city-123"
        assert events[0].name == "Rome"

    def test_cannot_exceed_production_queue_capacity(self):
        """Adding more than 10 items to queue should raise error."""
        city = City.found(...)

        for i in range(10):
            city.add_production_item(ProductionItem(f"item-{i}", ...))

        with pytest.raises(ProductionQueueFullError):
            city.add_production_item(ProductionItem("item-11", ...))

    def test_food_growth_increases_population(self):
        """Accumulating enough food should increase population."""
        city = City.found(...)
        assert city.population() == 1

        # Need 10 food to grow to population 2
        city.apply_food_growth(5)
        assert city.population() == 1

        city.apply_food_growth(5)
        assert city.population() == 2
```

### Integration Tests (Application Layer)

Test complete use case flows.

```python
# tests/integration/test_found_city_flow.py
import pytest
from application.commands import FoundCityCommand
from application.handlers import FoundCityCommandHandler

class TestFoundCityFlow:
    def test_successful_city_founding(
        self,
        command_handler: FoundCityCommandHandler,
        test_player,
        test_settler
    ):
        """Complete flow of founding a city."""
        command = FoundCityCommand(
            player_id=test_player.player_id(),
            settler_unit_id=test_settler.unit_id(),
            city_name="Rome",
            x=10,
            y=10
        )

        # Execute command
        command_handler.handle(command)

        # Verify city was created
        city = city_repository.find_by_coordinates(Coordinates(10, 10))
        assert city is not None
        assert city.name() == "Rome"
        assert city.owner_id() == test_player.player_id()

        # Verify settler was consumed
        settler = unit_repository.find_by_id(test_settler.unit_id())
        assert settler.is_consumed()

        # Verify event was published
        events = event_bus.get_published_events()
        assert any(
            isinstance(e, CityFoundedEvent) and e.city_id == city.city_id()
            for e in events
        )
```

---

## Common Patterns and Examples

### Pattern: First-Come-First-Served Concurrency

**Problem:** Two players move units to the same tile. Who gets there first?

**Solution:** Use database-level locks or optimistic concurrency control.

```python
class CompleteUnitMovementCommandHandler:
    def handle(self, command: CompleteUnitMovementCommand) -> None:
        with self.__unit_of_work():
            # Lock the destination tile
            tile = self.__world_repository.find_tile_with_lock(
                Coordinates(command.x, command.y)
            )

            # Check if tile is already occupied
            if tile.is_occupied_by_enemy(command.player_id):
                # Initiate combat instead
                self.__initiate_combat(command)
                return

            # Complete movement
            unit = self.__unit_repository.find_or_fail_by_id(command.unit_id)
            unit.complete_movement()
            tile.set_occupant(unit.unit_id())

            self.__unit_repository.save(unit)
            self.__world_repository.save_tile(tile)
            self.__event_bus.bulk_publish(unit.pull_events())
```

### Pattern: Progressive Fog of War Revelation

**Problem:** As a unit moves, it should reveal tiles along its path.

**Solution:** Calculate visible tiles and update world state incrementally.

```python
# domain/services/fog_of_war_updater.py
class FogOfWarUpdater:
    """Domain service for calculating fog of war."""

    def calculate_visible_tiles(
        self,
        coordinates: Coordinates,
        vision_range: int
    ) -> list[Coordinates]:
        """
        Returns list of tile coordinates visible from given position.
        Uses circle algorithm for vision.
        """
        visible = []
        for dx in range(-vision_range, vision_range + 1):
            for dy in range(-vision_range, vision_range + 1):
                if dx * dx + dy * dy <= vision_range * vision_range:
                    visible.append(
                        Coordinates(coordinates.x + dx, coordinates.y + dy)
                    )
        return visible


# application/event_handlers/unit_moved_event_handler.py
class UnitMovedEventHandler:
    """Reveals fog of war when unit moves."""

    def handle(self, event: UnitMovementStartedEvent) -> None:
        """Updates fog of war along movement path."""
        # Calculate tiles along path
        path = self.__calculate_path(
            from_coords=Coordinates(event.from_x, event.from_y),
            to_coords=Coordinates(event.to_x, event.to_y)
        )

        world = self.__world_repository.get_world()

        for coords in path:
            visible_tiles = self.__fog_updater.calculate_visible_tiles(
                coordinates=coords,
                vision_range=2  # Scout vision range
            )

            for tile_coords in visible_tiles:
                world.reveal_tile_for_player(tile_coords, event.owner_id)

        with self.__unit_of_work():
            self.__world_repository.save(world)
```

### Pattern: Production Queue Processing

**Problem:** Cities have production queues that complete over real-world time.

**Solution:** Store completion timestamps and process them asynchronously.

```python
# domain/entities/city.py (excerpt)
class City(AggregateRoot):
    def start_production(self, item_type: str, duration_seconds: int) -> None:
        """Adds item to production queue with calculated completion time."""
        completion_time = datetime.utcnow() + timedelta(seconds=duration_seconds)

        item = ProductionItem(
            item_type=item_type,
            started_at=datetime.utcnow(),
            completion_time=completion_time
        )

        self.add_production_item(item)

        self.record(ProductionStartedEvent(
            city_id=self.__city_id,
            item_type=item_type,
            completion_time=completion_time.isoformat()
        ))


# application/schedulers/production_processor.py
class ProductionProcessor:
    """Checks for completed production every second."""

    def process_completions(self) -> None:
        """Completes production for all cities with finished items."""
        now = datetime.utcnow()

        # Finder to get cities with completed production
        cities_with_completions = self.__finder.find_cities_with_production_completing_before(now)

        for city_view in cities_with_completions:
            # Execute command to complete production
            command = CompleteProductionCommand(
                city_id=city_view.city_id,
                item_type=city_view.current_production_item
            )
            self.__command_bus.dispatch(command)
```

---

## Key Takeaways

1. **Keep business logic in the domain** - Entities and domain services own the rules
2. **Aggregate roots record events** - Use `aggregate.pull_events()` pattern
3. **Application layer orchestrates** - Manages transactions, persistence, and event publishing
4. **Separate commands from queries** - Use repositories for writes, finders for reads
5. **Events are self-contained** - Include all context, not just IDs
6. **Transactions in application layer only** - Never in domain or infrastructure
7. **Use JSON-serializable primitives** - For all messages crossing boundaries
8. **Preserve exception context** - Use `raise` and `raise ... from e`
9. **Design for concurrency** - Use locks and optimistic concurrency control
10. **Test at all layers** - Unit tests for domain, integration tests for flows

---

## Next Steps

1. **Set up project structure** following the code organization above
2. **Implement core domain entities** (Player, City, Unit, Tile)
3. **Define all domain events** in `domain/events/`
4. **Implement command handlers** for core actions (found city, move unit)
5. **Set up event bus** (start with in-memory, migrate to RabbitMQ)
6. **Implement global tick scheduler** for resource generation
7. **Build query side** with finders and view models
8. **Add real-time updates** via WebSockets
9. **Optimize with read models** (materialized views, Redis cache)
10. **Scale with workers** (Celery for async processing)

---

**Remember:** These patterns exist to help us build a maintainable, scalable system. Apply them pragmatically - start simple and add complexity only when needed. The goal is to deliver a working game that can evolve over time.
