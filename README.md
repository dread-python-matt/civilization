# 🌍 Digital Civilization Simulator

> A from-scratch, real-time 2D world/organism simulation and mini game-engine built in pure Python.
> Perlin/OpenSimplex procedural terrain, autonomous agents with perception, pathfinding and hunting,
> a tick-based physics/movement system, an event-driven domain core, and a `pygame` renderer — all wired
> together with a Domain-Driven, Hexagonal (Ports & Adapters) architecture.

![Elevation heightmap](../output/plot_2025-06-17%2014-39-10_0.png)

*(Generated elevation field — layered OpenSimplex noise, later thresholded into water/grass biomes.)*

---

## Table of Contents

1. [Purpose](#purpose)
2. [What it actually does](#what-it-actually-does)
3. [Quickstart](#quickstart)
4. [Tech Stack](#tech-stack)
5. [Main Features](#main-features)
6. [Architecture](#architecture)
7. [Runtime Workflow](#runtime-workflow)
8. [Project Layout](#project-layout)
9. [Configuration](#configuration)
10. [Controls](#controls)
11. [Known Limitations](#known-limitations)

---

## Purpose

The project is an **educational, self-built game engine / agent-based simulator**. The author set out to
observe emergent interactions between autonomous agents on a procedurally generated map — assigning them
traits, life parameters (hunger, fatigue, health), perception and decision-making.

As candidly documented in the original `README.md`, the scope was deliberately reduced once the cost of
building a performant engine from scratch became clear. What remains is a **working real-time simulation
core**: terrain generation, organisms placed on the map, animals/humans equipped with "brains", a perception
layer, tick-based smooth movement, a hunting behaviour, and a `pygame` renderer driven by state snapshots.

The value of the project is **not** a finished game — it is the **engineering it demonstrates**: a cleanly
layered domain, ports & adapters, an event bus, data-oriented performance work (Structure-of-Arrays, double
buffering, bit-packed ids), and a fixed-timestep game loop.

---

## What it actually does

Running `src/main.py`:

1. Generates a `map_width × map_height` elevation grid from layered OpenSimplex noise.
2. Converts elevation into tiles (currently two terrains: **water** and **grass**).
3. Randomly scatters **plants** (berries, trees), **animals** (rabbits) and **humans** (agents) on valid tiles.
4. Opens a `pygame` window and runs a real-time loop where organisms `tick` and render as sprites.
5. Lets you drive **one example human agent** from the keyboard: walk in four directions, or trigger a
   **hunt** that pathfinds to the nearest living animal and kills it (the victim flips to a "dead" sprite).
6. Provides a Tkinter main menu and an in-game pause overlay.

Animals currently follow a **random-walk** strategy; the human agent adds **hunting** (BFS pathfinding +
kill-in-range). A SQLAlchemy/PostgreSQL persistence layer for saving/loading worlds exists but is **partially
implemented and disabled by default** (see [Known Limitations](#known-limitations)).

---

## Quickstart

### Prerequisites
- **Python 3.12** (uses 3.10+ syntax: `X | None` unions, structural `match`-free but modern typing).
- A desktop environment (the app opens a `pygame` window and a Tkinter menu).
- *(Optional, only for the persistence experiments)* a local **PostgreSQL** instance.

### Install & run

```bash
# 1. Clone
git clone <this-repo>
cd civilization-main

# 2. Create a virtual environment
python -m venv .venv
# Windows
.venv\Scripts\activate
# Linux/macOS
source .venv/bin/activate

# 3. Install runtime dependencies
pip install -r requirements.txt

# 4. Install the package in editable mode (uses the src/ layout so `domain`, `infrastructure`, … resolve)
pip install -e .

# 5. Run the simulation
python src/main.py
```

> **Note on imports:** the source uses a *flat* top-level package layout under `src/` (e.g. `from domain...`,
> `from infrastructure...`). `pip install -e .` (via `setup.py`, `package_dir={'': 'src'}`) is the intended
> way to put those packages on the path. Running `python src/main.py` from the repo root works because
> `main.py` lives inside `src/`; if you hit `ModuleNotFoundError`, set `PYTHONPATH=src`.

> **Database is optional:** `main.py` calls `init_db()` which will try to reach PostgreSQL at
> `postgresql+psycopg2://civilian:civilian@localhost:5432/civilization`. If you don't need persistence,
> comment out `init_db()` in `main()` — the simulation itself does not depend on the database.

### Running the tests
The repo ships a `tests/` suite (pytest). **Heads-up:** several test modules are out of sync with the current
code and will not pass as-is (see [EVALUATION.md](./EVALUATION.md)). `pytest` is also not listed in
`requirements.txt`:

```bash
pip install pytest
PYTHONPATH=src pytest tests
```

---

## Tech Stack

| Area | Technology | Notes |
|------|-----------|-------|
| **Language** | Python 3.12 | Modern typing (`Protocol`, `NamedTuple`, `dataclass(slots=True)`, PEP 604 unions) |
| **Rendering / game loop** | `pygame` 2.6.1 | Window, surfaces, blitting, event/timer system, sprite rotation |
| **Procedural noise** | `opensimplex` 0.4.5.1 | Multi-octave elevation generation |
| **Numerics** | `numpy` 2.3.3 | Elevation grid as `float32` ndarray |
| **Menu UI** | `tkinter` (stdlib) | Main menu; `pygame` overlay used for the pause screen |
| **Persistence / ORM** | `SQLAlchemy` 2.0.34 (+ PostgreSQL via `psycopg2`) | Declarative models (`WorldDB`, `TileDB`), repository, session context manager |
| **Visualization/debug** | `matplotlib` 3.10.6 | Elevation heightmap plots (see `output/`) |
| **Profiling** | `codetiming` 1.4.0 | `Timer` decorators/context managers around hot paths |
| **Progress bars** | `tqdm` 4.66.5 | World-generation progress |
| **Packaging** | `setuptools` / `setup.py` | `src/` layout, `find_packages` |
| **Testing** | `pytest` | Present in `tests/` (currently stale; not in requirements) |
| **Std-lib highlights** | `array`, `dataclasses`, `enum`, `collections` (`deque`, `defaultdict`), `itertools`, `functools.lru_cache`, `abc`, `logging` (`RotatingFileHandler`), `contextlib`, `asyncio` (experimented with, now synchronous) |

**Paradigms & patterns used:** Domain-Driven Design, Hexagonal / Ports & Adapters, Dependency Injection
(composition root in `bootstrap/`), Event Bus (pub/sub **and** a command bus with responses), Facade,
Factory Method / Abstract Factory, Strategy, State, Command + Sequence (movement actions), Adapter,
Command–Query Separation (`Transform` read-only vs writer views), and **data-oriented design**
(Structure-of-Arrays + double buffering).

---

## Main Features

### 1. Procedural terrain generation
- Layered **OpenSimplex** noise with configurable **octaves** (frequency/amplitude pairs) and an
  **elevation power** curve (`ElevationGenerator`, `OpenSimplexNoiseGenerator`).
- Deterministic via a configurable **seed**; normalized to `[0,1]`, thresholded into biomes by `TileAdapter`.

### 2. Entity / organism system
- **Prefabs** (`OrganismPrefab`, `PlantPrefab`) describe a species (name, allowed terrains, edibility,
  block radius). **Factories** assemble the full object graph per organism.
- Three organism families: **Plant** (static), **Animal** (moving, brain-driven), **Human** (Animal subclass
  with hunting). Placement respects allowed terrain and (for trees) a spacing **block radius**.

### 3. Perception ("vision")
- Each brain owns a `Vision` over a square window around the agent, materialized as a **Perception**
  Structure-of-Arrays (`xs`, `ys`, `terrains`, `organisms`, `organisms_alive`, `offsets`, …) built by the
  `WorldPerceptionAdapter` behind a `VisionPort`.
- Vision provides queries: possible moves, nearest living animal, animals within distance, neighbours with
  allowed terrain, etc.

### 4. Brains, strategies & hunting
- `Brain` runs a **decision strategy** (`RandomWalkStrategy` / `IdleStrategy`) on a **bucketed schedule**
  (`DECIDE_BUCKETS`, per-agent `_phase`) so not every agent decides every tick.
- `HumanBrain` adds **`Hunting`**: `PathPlanner.find_shortest_path` (BFS over perceived, terrain-valid
  neighbours) to the closest living animal, then a kill request when in range.

### 5. Smooth, tick-based movement
- Movement is a **sequence of actions** (`RotationAction` → `InterpolationAction`) advanced one step per
  tick, producing sub-tile interpolation and rotation toward the travel direction.
- `Transform` exposes **read-only** and **writer** facets; sub-tile motion uses an integer offset with an
  `OFFSET_TO_POSITION_RATIO` so positions stay integral while rendering interpolates.

### 6. Event-driven world interactions
- A central `EventBus` carries **events** (`position update`, `death`) and **commands with responses**
  (`walk_request`, `kill_request`). `WorldInteractionsHandler` validates moves (bounds, terrain, occupancy,
  **reservations** to prevent two agents claiming a tile) and mutates authoritative world state.

### 7. Real-time rendering
- `WorldSnapshotAdapter` produces an immutable **`WorldFrameSnapshot`** (tiles + organisms as SoA) each
  frame. `WorldRenderer` / `WorldPresenter` decode it, pick sprites, apply rotation and offsets, and blit.
- The full tile map is **pre-rendered once** to an off-screen surface; each frame blits only the camera
  viewport rectangle. Dead vs alive organisms select different sprites; a simple layer system orders draws.

### 8. Game loop, camera & UI
- **Fixed-timestep** loop decoupling logic (`LOGIC_HZ = 120`) from render (`RENDER_HZ = 60`) using `pygame`
  timers, an **event-wait** (no busy spin), and a **max-ticks-per-frame** clamp to avoid the "spiral of death".
- Pan-able `Camera` with clamping; **Tkinter** main menu; **pygame** translucent pause overlay.

### 9. Cross-cutting infrastructure
- Rotating **file + console logging** per subsystem (`logs/`), `matplotlib` elevation debug plots (`output/`),
  and a bit-packed **`IdRegistry`** encoding `(group, kind)` into a single integer.

---

## Architecture

The codebase follows a **Hexagonal / layered DDD** structure. Dependencies point **inward** toward the
`domain`; the outside world (pygame, SQLAlchemy, files) sits in `infrastructure`, `input` and `view` and is
reached only through **ports** (Python `Protocol`s / ABCs).

```
                         ┌──────────────────────────────────────────────┐
                         │                   bootstrap                   │
                         │  (composition root — wires everything via DI) │
                         └───────────────────────┬──────────────────────┘
                                                 │ builds
                 ┌───────────────────────────────▼───────────────────────────────┐
   input/  ────► │                          DOMAIN CORE                            │ ◄──── view/
  (game loop,    │                                                                 │   (tkinter menu,
   keyboard)     │   WorldFacade ── WorldMap / WorldStateService                    │    pause overlay)
                 │       │              (authoritative tiles + positions)           │
                 │       │                                                          │
                 │   EventBus ◄──► WorldInteractionsHandler / Validator (ports)     │
                 │       ▲                                                          │
                 │   Organisms: Plant / Animal / Human                              │
                 │       └── Brain → Strategy / Hunting / PathPlanner               │
                 │              ├── Vision ──(VisionPort)──► WorldPerceptionAdapter │
                 │              ├── Movement → Actions (Rotation, Interpolation)    │
                 │              └── Transform (read-only / writer)                  │
                 │   Services: generators (noise, elevation, plants, animals, …)    │
                 └───────────────────────┬─────────────────────────────┬──────────┘
                                         │ snapshots (SoA)              │ ports
                        ┌────────────────▼──────────────┐   ┌───────────▼───────────────┐
                        │      infrastructure/rendering  │   │ infrastructure/persistance │
                        │  SnapshotAdapter → Renderer →  │   │  SQLAlchemy models, repo,  │
                        │  Presenter → pygame sprites    │   │  session (PostgreSQL)      │
                        └────────────────────────────────┘   └────────────────────────────┘
```

**Key architectural decisions**

- **Ports & Adapters:** `VisionPortProtocol`, `WorldPerceptionAdapterProtocol`,
  `WorldInteractionsValidatorProtocol` decouple the domain from concrete implementations, enabling
  substitution and testing.
- **Composition root:** `bootstrap/world_setup.py` is the single place that constructs and injects
  collaborators — no service locators, minimal globals.
- **Facade:** `WorldFacade` is the one surface the game loop touches (`tick`, `add_organism`,
  `get_example_agent`, `is_position_allowed`, …).
- **Event bus with two channels:** fire-and-forget **events** (`emit`/`on`) and **commands that return a
  result** (`emit_with_response`/`on_command`) — an informal CQRS split.
- **Command–Query Separation on data:** `Transform` yields a `TransformReadOnly` (raises on write) and a
  `TransformWriter`, so perception/AI can read pose safely while only movement mutates it.
- **Data-oriented performance layer:** perception and render frames are **Structure-of-Arrays** using the
  `array` module with tight dtypes (`'B'`, `'H'`, `'I'`, `'b'`, `'f'`), plus **double (ping-pong) buffering**
  and cached bound `append` methods on `__slots__` objects.

See [ADDITIONAL.md](./ADDITIONAL.md) for a deeper pattern-by-pattern breakdown, and
[PROS_AND_CONS.md](./PROS_AND_CONS.md) for the trade-offs each of these choices carries.

---

## Runtime Workflow

### Startup (composition)
```
main()
 ├─ get_logger("civilization")
 ├─ create_world_generator(logger)          # bootstrap wires noise → elevation → generators → event bus
 ├─ init_db()                               # SQLAlchemy: create tables (PostgreSQL) — optional/disable-able
 ├─ create_world_service(world_generator)
 └─ world_generator.create_world_facade_and_its_adapter(W, H, scale)
      1. _generate_map_array         → numpy float32 grid; per cell: ElevationGenerator(OpenSimplex octaves)
      2. TileAdapter.to_tiles        → threshold elevation → Terrain → list[Tile]
      3. build WorldMap, WorldStateService, Validator, PerceptionAdapter, VisionPort, WorldFacade
         (WorldFacade registers WorldInteractionsHandler on the EventBus)
      4. generate plants → animals → humans (factories build Transform+Movement+Vision+PathPlanner+Brain;
         each Brain registers a BrainInteractionsHandler on the EventBus)
      5. build WorldSnapshotAdapter
 → run_game(world_facade, world_snapshot_adapter)
```

### Frame loop (`input/game_loop.py`)
```
while running:
   wait for the next event, then drain the queue
   ├─ LOGIC_EVENT  (120 Hz timer) → accumulate logic ticks (clamped to max_logic_ticks_per_frame)
   ├─ RENDER_EVENT (60 Hz timer)  → request a render
   └─ KEY events                  → map to an action (walk/hunt/pause) for the example human
   if action:  drive agent._brain (walk(direction) / hunt() / pause overlay)
   if ticks:   world_facade.tick(t)  →  for each organism: organism.tick(t)  →  brain.tick(t)
   if render:  update camera → snapshot (SoA) → blit map viewport + organisms → flip
```

### One movement, end-to-end (event flow)
```
Brain.decide → walk(dir)
  → BrainInteractionsHandler.emit_walking_decision(dir)
      → EventBus command "walk_request"
          → WorldInteractionsHandler validates (bounds / terrain / occupied / reserved)
              → reserves target tile, returns MoveResult.SUCCESS
  → Movement.move(dir) starts SequenceAction[RotationAction, InterpolationAction]
  → (subsequent ticks) Movement.tick() steps the sequence until SUCCESS
  → Brain notifies "position_changed"
      → handler ends the move (occupied ↔ reserved swap in WorldStateService)
      → emits "position update"  → every Brain refreshes its Vision
```

### Hunting (human agent)
```
HumanBrain.hunt()
  → Vision.detect_closest_alive_animal()  (nearest living animal in the perception window)
  → PathPlanner.find_shortest_path (BFS over terrain-valid perceived neighbours)
  → each tick: advance one step along the path; when within range →
       EventBus command "kill_request" → Validator (both alive?) → register death
         → "death" event → victim Brain.kill_itself() → renderer draws the dead sprite
```

---

## Project Layout

```
civilization-main/
├─ src/
│  ├─ main.py                     # entry point (compose + run)
│  ├─ bootstrap/                  # composition root / DI wiring
│  ├─ domain/                     # ~3,400 LOC — the heart of the project
│  │  ├─ components/              # Position (NamedTuple), Direction, Terrain, Renderable
│  │  ├─ organism/
│  │  │  ├─ instances/            # Organism / Animal / Human / Plant
│  │  │  ├─ brain/                # Brain, HumanBrain, PathPlanner, interactions handler
│  │  │  ├─ movement/             # Movement + action/ (rotation, interpolation, sequence)
│  │  │  ├─ perception/           # Vision, Perception (SoA), TargetInfo, adapters/ports
│  │  │  ├─ strategy/             # DecisionStrategy: RandomWalk / Idle
│  │  │  ├─ state/                # OrganismState: Idle / Walking / Hunting (see caveats)
│  │  │  ├─ hunting/              # Hunting behaviour
│  │  │  ├─ factory/              # per-species factories
│  │  │  ├─ prefabs/              # species definitions
│  │  │  ├─ transform/            # Transform + read-only/writer views
│  │  │  └─ vitals.py             # hunger / fatigue / health
│  │  ├─ services/                # world_service, event_bus, tile_adapter,
│  │  │  ├─ generators/           #   world/elevation/plants/animals/human generators
│  │  │  │  └─ noise/             #   OpenSimplex + octaves
│  │  │  └─ movement/             #   MoveResult, MovementSystem (legacy/dead — see caveats)
│  │  └─ world_map/               # WorldMap, WorldStateService, Facade, interaction handler/validator, VisionPort
│  ├─ infrastructure/
│  │  ├─ rendering/               # renderer, presenter, snapshot adapter, camera, soa/, sprite/
│  │  └─ persistance/             # SQLAlchemy base/session/models/repositories (partial)
│  ├─ input/                      # game_loop, keyboard
│  ├─ view/                       # tkinter menu + pygame pause overlay
│  └─ shared/                     # config, constants, id_registry, logger, paths, utils
├─ tests/                         # pytest suite (currently stale — see EVALUATION.md)
├─ assets/                        # sprites (terrain / plants / animals / human)
├─ data/                          # JSON: settings, biomes, climate_zones, rules (partly aspirational)
├─ logs/                          # rotating logs (runtime artifacts, committed)
├─ output/                        # matplotlib elevation plots (committed)
├─ requirements.txt
└─ setup.py
```

---

## Configuration

Runtime configuration lives in `src/shared/config.py` (a `CONFIG` dict). A parallel `data/settings.json`
mirrors many of the same keys (the JSON is **not** currently loaded by the app — `config.py` is the source
of truth). Notable knobs:

| Key | Meaning |
|-----|---------|
| `map_width`, `map_height`, `scale` | Grid dimensions and noise scale |
| `elevation_seed`, `elevation_power`, `elevation_octaves` | Terrain determinism & shape |
| `animals_count`, `plants_count`, `human_count` | Population sizes |
| `human_vision_radius` | Perception window radius |
| `tile_size`, `screen_width`, `screen_height`, `fps` | Rendering |

Species distributions (`PLANTS_DIST`, `ANIMALS_DIST`, `HUMAN_DIST`) and key bindings
(`DEFAULT_KEY_BINDINGS`) are also defined there.

---

## Controls

| Key | Action |
|-----|--------|
| `W` / `A` / `S` / `D` | Pan the camera |
| `↑ / ↓ / ← / →` | Move the example human agent one tile |
| `3` | **Hunt** — pathfind to and kill the nearest living animal |
| `Esc` | Pause overlay (Resume / Main menu) |

*(Action-key mapping is defined in `input/keyboard.py`; note a couple of placeholder bindings there.)*

---

## Known Limitations

These are summarized here and analysed in depth in
[PERFORMANCE.md](./PERFORMANCE.md), [PROS_AND_CONS.md](./PROS_AND_CONS.md) and [EVALUATION.md](./EVALUATION.md):

- **⚠️ The app lags — confirmed and measured.** This is the most noticeable real-world issue. Every
  completed move broadcasts a global `"position update"`, so **all** brains rebuild their entire vision —
  **O(N) full rebuilds per move, O(N²) per busy tick**. A single completed move at the default ~30 agents
  already costs **~12 ms of vision rebuilds** on one synchronous thread, exceeding the whole **8.33 ms**
  logic-tick budget (120 Hz). On the render side, `pygame.transform.rotate` runs **per sprite, every
  frame, uncached**. See **[PERFORMANCE.md](./PERFORMANCE.md)** for measurements, root causes (with
  `file:line`) and a ranked fix plan.
- **Persistence is partial and disabled by default.** SQLAlchemy models/repository/session exist, but the
  save/load path in `WorldService.get_world_by_name` is out of sync with the current `WorldMap`
  constructor and would fail; DB credentials are hardcoded. The DB is not required to run the sim.
- **The test suite is stale.** Multiple modules reference an older API (renamed classes, changed
  constructors, a non-existent `PerceivedObject`) and will not collect/pass without repair. `pytest`
  is missing from `requirements.txt`.
- **Some dead/vestigial code** remains (`services/movement/MovementSystem`, empty `animal_brain.py`,
  the `state/` classes are not actually driven by the tick loop, an emitted `organism_moved` event has no
  listener).
- **Two biomes only** (water/grass); the `data/` climate/biome/rules JSON files describe a richer,
  not-yet-implemented world model.
- **Comments/log messages are largely in Polish**, and several identifiers contain typos
  (`constans`, `persistance`, `proccess_action`, …).

---

*Documentation authored from a full review of the repository — static reading plus a runnable benchmark of
the real perception hot path (see [PERFORMANCE.md](./PERFORMANCE.md)). For the reviewer's scoring and a
candidate-oriented skills read-out, see [EVALUATION.md](./EVALUATION.md) and [ADDITIONAL.md](./ADDITIONAL.md).*
