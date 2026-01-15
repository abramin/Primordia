# Primordia - Vertical Slices

This document breaks down the Primordia game into vertical slices that can be implemented incrementally. Each slice delivers end-to-end functionality across the full stack (Django, Angular, PostgreSQL, Kafka, Celery).

---

## Slice 1: User Authentication & Static World View

**Goal:** Players can register, log in, and see a static game map.

### Frontend (Angular)
- Login/registration forms with form validation
- Authentication service with JWT token management
- Route guards for protected pages
- Basic map component displaying a 100x100 grid
- Render different tile types (Plains, Mountains, Water, Forests) with distinct colors/icons

### Backend (Django)
- User model and authentication endpoints (register, login, logout)
- JWT token generation and validation
- World generation command to create initial 100x100 tile grid
- REST API endpoint: `GET /api/world/tiles` (returns all tiles with coordinates and terrain type)
- Tile model with fields: x, y, terrain_type

### Database (PostgreSQL)
- Users table
- Tiles table with spatial indexing on (x, y)

### Infrastructure
- Basic Django project structure
- Angular project setup with routing
- CORS configuration
- Environment configuration for DB connection

**Acceptance Criteria:**
- User can register and log in
- Authenticated user sees a rendered 100x100 map with different terrain types
- Map is static (no fog of war yet)

---

## Slice 2: Player State & Initial Settler

**Goal:** New players spawn with one Settler unit that they can see on the map.

### Frontend (Angular)
- Unit rendering layer on top of the map
- Display unit icon at specific coordinates
- Player resource panel showing Gold, Food, Science (initially zero)
- Service to fetch current player state

### Backend (Django)
- Player model (extends User) with fields: gold, food, science
- Unit model with fields: unit_type, owner (FK to Player), x, y, movement_speed, health
- Game initialization: when user first logs in, spawn a Settler at random valid location
- REST API endpoints:
  - `GET /api/player/state` (returns player resources and owned units)
  - `GET /api/units/mine` (returns all units owned by authenticated player)

### Database (PostgreSQL)
- Players table (gold, food, science, tech_tree_state JSON)
- Units table with FK to players and position fields

**Acceptance Criteria:**
- New player automatically receives one Settler unit at game start
- Player can see their Settler's position on the map
- Player resource panel displays current resources (all zero initially)

---

## Slice 3: City Founding (Synchronous)

**Goal:** Player can click their Settler and found a city instantly.

### Frontend (Angular)
- Unit selection: click on Settler to select it
- Context menu or button: "Found City" action
- City rendering on map (different from unit icon)
- Update map when city is founded (Settler disappears, City appears)

### Backend (Django)
- City model with fields: name, owner (FK to Player), x, y, population, food_stores, production_queue (JSON)
- REST API endpoint: `POST /api/cities/found`
  - Request body: `{ x, y }` (coordinates of settler)
  - Validation: Check player owns a Settler at those coordinates
  - Action: Delete Settler unit, create City entity
  - Response: New city data
- Business logic: City naming (auto-generate or let player choose)

### Database (PostgreSQL)
- Cities table

**Acceptance Criteria:**
- Player can click on their Settler and select "Found City"
- Settler is removed from the map
- City appears at the same location
- API validates that player owns the Settler before allowing city founding

---

## Slice 4: Global Tick - Passive Resource Generation

**Goal:** Cities automatically generate resources every 10 seconds, even when player is offline.

### Frontend (Angular)
- Auto-refresh player state every 10 seconds via polling or WebSocket
- Animate resource counter updates

### Backend (Django)
- Resource generation logic: each city generates Food, Gold, Science based on tile types in its radius
- Celery periodic task: `global_tick` runs every 10 seconds
  - Query all cities
  - Calculate resource generation for each city
  - Update player resource totals
- REST API: Enhanced `GET /api/player/state` reflects updated resources

### Infrastructure (Kafka + Celery)
- Install and configure Kafka
- Install and configure Celery with Redis/RabbitMQ as broker
- Celery beat scheduler for periodic tasks
- Create Celery task: `tasks.global_tick()`

### Database (PostgreSQL)
- Add timestamp fields to Player model: `last_tick_at`

**Acceptance Criteria:**
- Every 10 seconds, all cities generate resources
- Player resource totals increase automatically
- Resource generation continues even when player is logged out
- Frontend reflects resource updates in real-time

---

## Slice 5: Building Production Queue (Asynchronous)

**Goal:** Player can queue a building (e.g., Granary) that takes real-world time to complete.

### Frontend (Angular)
- City detail panel showing production queue
- Building selection menu with available buildings and their costs/durations
- Display production timer counting down
- WebSocket listener for building completion events

### Backend (Django)
- Building model/enum with types: Granary, Wall, Market (each with production_cost, duration_seconds)
- REST API endpoint: `POST /api/cities/{city_id}/build`
  - Request body: `{ building_type }`
  - Validation: Check city not currently building something, player has resources
  - Action: Deduct resources, add building to city's production queue, publish event to Kafka
- Kafka producer: Publish `BuildingStarted` event
- Celery consumer: Listen for `BuildingStarted` events, schedule delayed task
- Celery delayed task: `complete_building.apply_async(args=[city_id], countdown=duration_seconds)`
  - When fires: Update city with completed building, publish `BuildingCompleted` event

### Infrastructure (Kafka)
- Kafka topic: `game-events`
- Celery task: `tasks.complete_building(city_id, building_type)`

### Database (PostgreSQL)
- Buildings table (or JSON field on City): building_type, started_at, completes_at, completed

**Acceptance Criteria:**
- Player can select a city and choose "Build Granary"
- System validates and deducts resources
- Building appears in production queue with countdown
- After the specified duration (e.g., 2 minutes), building completes automatically
- Building completion applies effects (e.g., Granary increases food generation)

---

## Slice 6: Unit Movement (Asynchronous)

**Goal:** Player can order a unit to move to a destination tile, and it travels over time.

### Frontend (Angular)
- Click unit to select, click destination tile to move
- Display movement path preview
- Animate unit position updates as it moves tile-by-tile
- Show ETA for unit arrival

### Backend (Django)
- REST API endpoint: `POST /api/units/{unit_id}/move`
  - Request body: `{ destination_x, destination_y }`
  - Validation: Player owns unit, destination is valid
  - Action: Calculate path, calculate total travel time, publish `MovementStarted` event
- Pathfinding logic: A* or simple Manhattan distance
- Movement processing: Celery periodic task or event-driven
  - Option A: Global tick (every 10 seconds) advances all moving units
  - Option B: Per-unit scheduled tasks for arrival
- Kafka producer: Publish `MovementStarted`, `UnitMoved` (per tile), `MovementCompleted` events
- Celery consumer: Process movement events, update unit positions in DB

### Database (PostgreSQL)
- Unit table additions: destination_x, destination_y, arrival_time, path (JSON array)

**Acceptance Criteria:**
- Player can select a unit and click a destination
- Unit begins moving toward destination
- Unit position updates in real-time on the map (tile-by-tile)
- Movement takes calculated real-world time based on distance and unit speed
- Unit reaches exact destination at calculated time

---

## Slice 7: Fog of War

**Goal:** Tiles are hidden until revealed by units or cities.

### Frontend (Angular)
- Render tiles as "fog" (gray/black) if not visible to player
- Reveal tiles when player's units/cities can see them
- Store visibility state per player

### Backend (Django)
- TileVisibility model: player_id, tile_x, tile_y, revealed_at
- Vision calculation: When unit moves or city founded, calculate visible tiles (radius around position)
- REST API endpoint: `GET /api/world/visible-tiles` (returns only tiles visible to authenticated player)
- Update visibility when units move or cities are founded

### Database (PostgreSQL)
- TileVisibility table (indexed by player_id and tile coordinates)

**Acceptance Criteria:**
- New player sees only tiles around their starting Settler
- As units move, they reveal new tiles
- Revealed tiles remain visible even after unit leaves (persistent vision)
- Other players' units are not visible unless in vision range

---

## Slice 8: Combat System

**Goal:** When two hostile units occupy the same tile, combat occurs automatically.

### Frontend (Angular)
- Combat animation/notification when battle occurs
- Display unit health bars
- Show combat results (winner/loser, damage dealt)

### Backend (Django)
- Combat calculation logic: compare unit stats, apply terrain bonuses
- Trigger: When unit movement completes and destination tile has enemy unit
- Kafka event: `CombatInitiated`
- Celery task: `resolve_combat(attacker_id, defender_id, tile_x, tile_y)`
  - Calculate outcome
  - Update unit health or remove destroyed units
  - Publish `CombatResolved` event
- REST API: Combat log endpoint to retrieve recent battles

### Database (PostgreSQL)
- CombatLog table: attacker_id, defender_id, location, outcome, timestamp

**Acceptance Criteria:**
- When two hostile units meet on same tile, combat triggers automatically
- Combat resolution considers unit stats and terrain
- Defeated units are removed from map
- Players receive notification of combat results

---

## Slice 9: City Growth & Population

**Goal:** Cities consume food to grow population over time.

### Frontend (Angular)
- City detail panel shows population and food stores
- Display growth progress bar
- Show specialist assignments (Farmers, Miners, Scientists)

### Backend (Django)
- City growth logic: consume food to increase population
- Enhanced global tick: process city growth for all cities
  - Accumulate food in `food_stores`
  - When threshold reached, increase population
- REST API: Endpoints to view and assign specialists

### Database (PostgreSQL)
- City table additions: population, food_stores, growth_rate

**Acceptance Criteria:**
- Cities accumulate food each tick
- When food threshold reached, city population increases
- Higher population enables more specialist roles
- Food production affects growth rate

---

## Slice 10: Technology Research

**Goal:** Players can research technologies that unlock new units/buildings.

### Frontend (Angular)
- Technology tree UI showing available and researched techs
- Research selection and progress display
- Show tech requirements and benefits

### Backend (Django)
- Technology model: name, cost, prerequisites, unlocks
- REST API endpoint: `POST /api/research/start`
  - Request body: `{ tech_name }`
  - Validation: Prerequisites met, player has science points
  - Action: Deduct science, start research timer
- Celery task: Complete research after duration
- Update player's tech tree state

### Database (PostgreSQL)
- Technologies table
- PlayerTech table: player_id, tech_id, researched_at, in_progress

**Acceptance Criteria:**
- Player can select a technology to research
- Science points are consumed
- Research completes after duration
- Completed tech unlocks new units/buildings

---

## Slice 11: Real-time Updates via WebSockets

**Goal:** Replace polling with WebSockets for instant updates.

### Frontend (Angular)
- WebSocket service to connect to backend
- Subscribe to channels: player state, world events, combat notifications
- Update UI reactively when events received

### Backend (Django)
- Django Channels integration for WebSocket support
- Kafka consumer publishes events to WebSocket channels
- WebSocket routing for authenticated connections
- Send targeted updates to specific players

### Infrastructure
- Install Django Channels and configure ASGI
- Redis channel layer for Django Channels

**Acceptance Criteria:**
- Player receives instant notifications for game events
- Resource updates appear without polling
- Unit movements and combat results push to client immediately
- Connection handles disconnects gracefully

---

## Slice 12: Concurrency & Event Ordering

**Goal:** Ensure perfect "first-come, first-served" integrity for concurrent actions.

### Backend (Django)
- Implement optimistic locking or database transactions for critical operations
- Kafka partition keys to ensure ordered processing of related events
- Event timestamp tracking and ordering logic
- Handle race conditions (e.g., two players moving to same tile)

### Infrastructure (Kafka)
- Configure Kafka partitioning strategy by game world regions
- Celery task idempotency checks

### Database (PostgreSQL)
- Add version fields or timestamps for optimistic locking
- Database transactions for atomic operations

**Acceptance Criteria:**
- When two players simultaneously attempt to claim same resource, first request succeeds
- Events are processed in exact order they were initiated
- No duplicate processing of events
- System handles concurrent writes without data corruption

---

## Slice 13: Multiple Unit Types & Advanced Combat

**Goal:** Add Warriors, Archers, Cavalry with different stats and abilities.

### Frontend (Angular)
- Different unit icons for each type
- Unit stat display (attack, defense, movement speed)
- Combat preview showing odds

### Backend (Django)
- Extend Unit model with unit_type enum and type-specific stats
- Enhanced combat calculation with rock-paper-scissors mechanics
- Unit training in cities (production queue)

### Database (PostgreSQL)
- UnitTypes reference table or enum

**Acceptance Criteria:**
- Players can train different unit types in cities
- Each unit type has distinct stats and movement speed
- Combat calculations consider unit type matchups

---

## Slice 14: Diplomacy & Trading

**Goal:** Players can trade resources and form alliances.

### Frontend (Angular)
- Diplomacy UI showing other players
- Trade offer creation and acceptance
- Alliance/war status display

### Backend (Django)
- Diplomacy model: player relationships (neutral, allied, at war)
- Trade offer model: sender, receiver, offered resources, requested resources
- REST API endpoints for diplomacy actions
- Kafka events for trade offers and status changes

### Database (PostgreSQL)
- PlayerRelationships table
- TradeOffers table

**Acceptance Criteria:**
- Players can send trade offers to each other
- Trade offers can be accepted or rejected
- Players can declare war or form alliances
- Diplomatic status affects combat and visibility

---

## Slice 15: Victory Conditions & Game End

**Goal:** Implement win conditions and game completion.

### Frontend (Angular)
- Victory progress tracking UI
- End game screen showing winner
- Leaderboard

### Backend (Django)
- Victory condition checks (e.g., reach Space Flight tech, capture all capitals)
- Game state model (in progress, completed)
- End game processing and player ranking
- Kafka event: `GameEnded`

### Database (PostgreSQL)
- GameSessions table
- PlayerScores table

**Acceptance Criteria:**
- System detects when victory condition is met
- Game ends and winner is declared
- Players can view final game state and statistics
- New games can be started

---

## Implementation Order Recommendation

1. **Slices 1-3:** Core authentication, world, and basic gameplay (synchronous actions)
2. **Slice 4:** Global tick infrastructure (asynchronous foundation)
3. **Slices 5-6:** Asynchronous building and movement (core event-driven gameplay)
4. **Slice 7:** Fog of war (strategic layer)
5. **Slice 8:** Combat (player interaction)
6. **Slice 11:** Real-time updates (UX improvement - can be done earlier if desired)
7. **Slices 9-10:** City growth and tech (depth mechanics)
8. **Slice 12:** Concurrency hardening (scalability)
9. **Slices 13-15:** Advanced features (polish and completion)

Each slice is independently deployable and testable, providing incremental value while building toward the complete game.
