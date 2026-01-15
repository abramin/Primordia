This Product Requirements Document (PRD) outlines the vision for **"Primordia,"** a persistent, real-time strategy game designed to showcase a modern, event-driven backend stack.

---

## 1. Project Overview

**Primordia** is a simplified 4X (eXplore, eXpand, eXploit, eXterminate) web-based strategy game. Unlike traditional turn-based Civilization games, Primordia operates in **continuous real-time**. Actions like building a granary or marching an army to a distant forest happen asynchronously over minutes or hours, persisting even when the player is offline.

### Objectives

* Provide a seamless, low-latency UI for player commands.
* Manage a persistent world state that evolves independently of user sessions.
* Handle high-concurrency interactions (e.g., two players claiming the same resource) with perfect "first-come, first-served" integrity.

---

## 2. Core Gameplay Mechanics

### A. The World (The Grid)

* **Map:** A coordinate-based grid (e.g., 100x100 tiles).
* **Tile Types:** Plains (Food), Mountains (Production), Water (Trade), and Forests (Mixed).
* **Fog of War:** Tiles are hidden until a player’s unit or city border reveals them.

### B. City Management

* **Settling:** Players start with one "Settler" to found their first city.
* **Growth:** Cities consume food to increase population. Higher population allows for more "Specialist" roles (Miners, Farmers, Scientists).
* **Production Queue:** Cities can build one item at a time (Units or Buildings). Each item has a **Production Cost** translated into **Real-World Time**.

### C. Unit Movement & Combat

* **Travel Time:** Moving from Tile A to Tile B is not instant. Movement speed depends on unit type and terrain.
* **Combat:** Occurs automatically when two hostile units occupy the same tile. Results are calculated based on unit stats and defensive bonuses of the terrain.

### D. The Resource Tick

* Resources are not "mined" manually. The empire generates Gold, Food, and Science every **10-second interval (The Global Tick)**.

---

## 3. Functional Requirements

### 3.1 Player Actions (Command Ingestion)

* The system must accept player commands (Move, Build, Research) via a REST API.
* Commands must be validated for "Possibility" (e.g., *Does the player have enough gold?*) before being queued for execution.

### 3.2 Asynchronous Event Processing

* **Delayed Execution:** Actions with a duration (e.g., "Build Wall - 5 mins") must be tracked and completed exactly when the timer expires.
* **Sequential Integrity:** If two players move to the same tile, the system must process the events in the exact order they were initiated to determine who arrives first.

### 3.3 World Persistence & Evolution

* The world state (resource counts, unit positions) must be stored in a relational format.
* Background processes must update the world state (e.g., resource growth) even if no players are currently logged in.

### 3.4 Real-time Updates

* The frontend must reflect changes in the world (e.g., a unit appearing from the fog) without requiring a manual page refresh.

---

## 4. User Flows

### Flow 1: Settling and Building

1. **Player** logs in and sees their Settler on a tile.
2. **Player** clicks "Found City."
3. **System** validates location, subtracts the unit, and creates a "City" entity.
4. **Player** selects "Build Granary" (Duration: 2 Minutes).
5. **Player** logs off.
6. **System** (2 mins later) completes the building and begins applying the Food bonus to that city’s stats.

### Flow 2: Exploration

1. **Player** orders a Scout to a coordinate 10 tiles away.
2. **System** calculates the path and sets an arrival time.
3. **Player** watches the Scout move tile-by-tile in real-time.
4. **System** reveals new tiles to the player as the Scout’s vision radius touches them.

---

## 5. Non-Functional Requirements

### 5.1 Scalability

* The architecture must support 1 to 2 players initially but be designed to handle thousands of concurrent "Unit Move" events via a message-bus architecture.

### 5.2 Fault Tolerance

* If a processing worker fails, game events must not be lost. They should remain in a buffer until a worker can resume.

### 5.3 Observability

* The system must track:
* **Event Lag:** Time between a player's click and the system processing the action.
* **Worker Health:** Throughput of the "Global Tick" updates.
* **Database Performance:** Latency of spatial queries (finding units near a city).



---

## 6. Game Data Definitions (High Level)

| Entity | Attributes |
| --- | --- |
| **Player** | Username, Total Gold, Total Science, Tech Tree State |
| **Tile** | Coordinates (X,Y), Terrain Type, Visible To (List of Player IDs) |
| **Unit** | Type (Scout/Warrior), Current Location, Destination, Arrival Time |
| **City** | Population, Food Stores, Production Queue, Owner |

---

## 7. Future Scope

* **Diplomacy:** Trading resources between players.
* **AI Barbarians:** Computer-controlled units that spawn in unexplored areas.
* **Victory Conditions:** First to reach "Space Flight" tech or capture the opponent's capital.
