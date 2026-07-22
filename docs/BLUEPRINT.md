# agents.exe (LDP3) — Faithful Replica Blueprint

**Goal.** Re-implement the shipped Unity game *agents.exe* (a generative-agents civilisation sandbox
by João Victor Pereira Tavares) **1:1 in LDP3**, and **complete everything the shipped build left
unfinished** — while deliberately exercising the LDP3 language across every subsystem. There is **no
LLM**: the deterministic **Mock heuristic** is the strategic AI, exactly as the shipped game falls
back to when the model is off.

This file is the plan. The exhaustive, code-grounded reverse-engineering of the original C# lives in
`docs/reference/replica_{world,agents,economy,nations,ui,core}.md` — consult those for exact numbers,
enums, thresholds and `file:line` references when implementing each phase.

Two principles, from the project owner:
1. **Same repo, fresh game.** Keep only the reusable engine primitives; rebuild the whole game layer.
2. **1:1 of the shipped game, *plus* what it was missing.** Match shipped behaviour faithfully, then
   also build the parts left as stubs/unfinished, realising the complete design from the proposal.

---

## The faithful game model (what the original actually is)

### World — TWO coordinate systems (the thing the first attempt got most wrong)
- **Simulation space**: a **480×480 integer grid**, `arr[x,y]`, as parallel arrays: biome, walkable,
  territory. Six **elevation-band biomes** — DeepOcean / Ocean / Beach / Grassland / Mountain / Snow —
  thresholded from a normalized heightmap at 0.42 / 0.46 / 0.62 / 0.78. **Walkable = Grassland+Beach only.**
- **Render space**: **isometric**. Tiles are **2:1 diamonds, 64×32 px** (art 64×48, PPU 64).
  - grid→world: `worldX = (gx − gy)·0.5`, `worldY = (gx + gy)·0.25`
  - world→grid (mouse pick): `gx = ⌊wx + 2·wy⌋`, `gy = ⌊2·wy − wx⌋`
  - **depth-sort by world-Y**; bottom-center pivots; **Mountain/Snow z-stack** for cliff height.
- **Worldgen (shipped)**: warped multi-octave **Perlin** + continent/island smoothstep masks +
  2-layer boundary noise, normalized. Deterministic from the seed.
- **Resources**: Tree→Wood(cap 10), Bush→Food(3), Stone(5), Iron(5), Fauna→Food(5, mobile); seeded on
  Grassland/Beach; forests are BFS clusters.
- **Camera**: WASD/arrow pan, scroll zoom (1.5–250).

### Agents (workers) — `docs/reference/replica_agents.md`
- Fields: HP 100, Direction(4), nationId, LifetimeSeconds (360–540s, ×2.5 for protagonists), Hunger
  (0–100), Profession, Role, Family, Carrying. Stage: Child (<60s) / Adult (≥60s).
- **12 professions**: Civilian, Woodcutter, Farmer, Hunter, Builder, Sawyer, TreePlanter, StoneMiner,
  IronMiner, Trader, Soldier, General — assigned by a scored selector under building-capacity caps,
  with 120s demotion grace and switch hysteresis.
- **Hierarchical FSM**: Dead → not-ready → Child(wander) → job FSM. Harvesters:
  Seek→Move→Work(timer)→HaulToCapital→Deposit. Soldiers: SeekBarracks→Idle→MovingToTarget→Engaging
  (15 dmg / 0.5s, range hysteresis 2.25/4.0 sq).
- **Survival (shipped)**: Hunger only (+1 per 7s, death at 100, eat 1 Food when city stock ≥30).
- **Reproduction**: housing-gated (free House, both parents Hunger<30, city food ≥ max(10, members/8),
  ~0.5 per eligible female per second).
- **Render**: 4-direction isometric sprite, per-nation tint, overlays (status bubble, carry icon,
  soldier HP bar, damage flash). Object pooling.
- **Tick cadences (critical)**: AI/FSM at **1/4 frame rate** (4 staggered groups), Hunger 30
  workers/frame, movement **every frame**. Timers accumulate deltaTime → wall-clock accurate.

### Economy — `docs/reference/replica_economy.md`
- **Code-as-data registries** (BuildingRegistry, ProfessionRegistry/Capacity, CombatStats,
  Footprint/BiomeRules). Resources: **Food, Wood, Stone, Iron, Gold, Planks** (Gold vestigial in
  shipped). Stockpiles **per-city**. Refinement chain: Woodcutter→Wood; Sawyer @Sawmill 1 Wood→4 Planks.
- **9 buildable**: Castle(free,3×3,HP500,initial,prereq→Barracks), TownHall(free,2×2,HP300,capture@≤50),
  Farm(8/4/0,2×2,3 farmers), HuntersLodge(8/4/3,pre-Farm,3 hunters), Sawmill(6/6/3,3 sawyers),
  House(5/3/0,pre-Farm,1×1), HouseMedium(12/6/0,2×2), HouseLarge(25/12/0,3×3),
  Barracks(10/8/6,pre-Castle,3×3,HP200,12 soldiers).
- **BuildingGrowthSystem** every 5s: cities by fewest buildings, ≤12/tick, priority ladder
  (war-barracks → food → housing → sawmill → lodge). Build = 8s builder-time, ≤2 builders/blueprint.
  Secondary city (radius 5) seeds 100 Wood + 10 adults.

### Nations, Protagonists, Mock heuristic — `docs/reference/replica_nations.md`
- **Protagonist** = a Worker with Role ∈ {Leader, Consort, Royal}, one of **7 personalities**
  (inherited at birth), lifetime ×2.5. **28 IntentTypes**.
- **MockHeuristic.Decide(nation, state, playerMessage)** — the AI. Called every ~45s (or on event
  wakeups) by a DecisionScheduler; deterministic `rng = seed ^ nationId ^ tick`; **first-match cascade**:
  - **R0** stalemate-peace (war >300s, casualties <8)
  - **R1** crisis reflexes (RecentEvents ≤30 ticks: war_declared→BuildArmy, army decimated→rebuild 20,
    food_crash→Farm/FocusFood, city_lost→Barracks)
  - **R2** player-message obey, gated by a personality `RollObey` table
  - **R3** reactive war (losing-badly if `own < enemy·0.4`; surrender/attack/conscript/retreat ladder)
  - **R3.5** proactive war (personality chance: Aggressive 25 / Cunning 15 / … + distance/strength bonuses)
  - **R4** name-heir
  - **R5** DefaultByPersonality (per-personality `rng.Next(100)` roll over FoundCity / PrioritizeBuilding
    / FocusResource / DeclareWar / Idle)
  - `RelativeStrength = (theirSoldiers + theirPop) / (ourSoldiers + ourPop)` is the war-appetite signal.
- **Succession**: designated heir, else auto-promote the elite worker (longevity, contribution,
  proximity to capital). **War**: WarRegistry + WarVictorySystem (lose at 0 cities / <10 members /
  >300s stalemate; win by city count).

### UI — Windows-95/98 procedural — `docs/reference/replica_ui.md`
- **1280×720 windowed**. Everything drawn in code from a **1×1 tinted sprite + 1–2px bevels**
  (raised/sunken). Palette: gray 192 / dark-gray 128 / blue (0,0,128) / gold (228,185,60) /
  desktop-blue bg (26,58,90). LiberationSans font.
- **Screens**: Main Menu (gold "agents.exe" logo; New Game/Load/Options/Quit), New Game (Seed **42**,
  Randomize-personalities ON, Use-LLM OFF, Start/Back), Load (5 slots + DEL), Options, Pause overlay.
- **In-game HUD**: top bar (Year / Tick / Source / queued), collapsible Wars + Recent-Intents panels
  (top-left), 360px **Inspector** (right; click a worker/building/nation), bottom bar (Pause/1x/2x/4x
  + Send box). Space pause, 1/2/3 → 1x/2x/4x.

### Core loop — `docs/reference/replica_core.md`
- **Real-time, no discrete tick.** `tick = round(scaled Time.time)`, `year = tick/60` (so 1x → 1 year
  = 60 real seconds). Speed = timeScale 0 / 1 / 2 / 4.
- **Per-frame order**: input/speed → agent AI (staggered 1/4) → locomotion → economy/production →
  population (pregnancy/marriage/succession) → nation/territory → combat → decisions (45s per nation)
  → UI (0.4s refresh).
- **Pathfinding**: **flow fields only** (unweighted 8-neighbour BFS from a target cell → Distance/
  Direction), per capital/city/resource/military-target; rebuilt when a mover's target changes. No A*.
- **Bootstrap**: MainMenu → NewGame → Game; **4 nations pre-placed**, 50 workers + a royal couple
  each, seed 42.
- **Save**: seed-based (`save_v1`, 5 slots) — load ≈ New Game with the saved seed.

---

## What the shipped game was MISSING (to build, per "1:1 + o que falta")

- **World**: tectonic plates, hydraulic erosion, rivers, latitude/humidity biomes, and the proposal's
  **separate terrain (water/plains/low-hills/hills/mountain) + biome (green plains/forest/desert/tundra)
  layers**; a fish resource node.
- **Agents**: **thirst and energy/sleep** survival axes and the fuller action vocabulary (swim, flee,
  follow, drink, rest, patrol).
- **Economy**: make the **6 declared-only buildings real** — FishingHut (Beach), Smithy, Bakery,
  Brewery, Forge, Marketplace — and give **Gold** a genuine role.
- **Nations**: implement the stubbed intents — **Surrender, Observe, propose alliance, demand tribute,
  one-off & continuous trade deals, send ambassador**; real diplomacy/alliances/trade.
- **UI / Player**: the **Worldbox god-powers** — paint/remove terrain, paint biome, add/remove
  resources, spawn/kill workers & protagonists, trigger events (drought/flood/earthquake), all logged
  to the event stream; the **nation-origin choice** (pre-placement vs organic emergence).
- **Save**: full state serialization (not just seed).

---

## LDP3 feature map (each subsystem earns its keep as a language stress-test)

| Subsystem | LDP3 features exercised (idiomatic, not forced) |
|---|---|
| World & iso render | `struct Tile` in a flat 480×480 array; Java-style `enum Biome` (color/walkable/threshold); operator-overloaded coord types for **grid *and* iso** vectors; `comptime` iso-transform constants |
| Agents | `region` for the worker pool (up to 3000) + per-tick scratch; exhaustive `match` on FSM state & profession; `enum` Direction/Role/Profession; `property` (isStarving, isAdult); ownership (`move`/`unique`) for the carry slot |
| Economy | **`catalog`** for the building/profession registries (a 1:1 fit for the code-as-data tables); `enum` Building/Resource; `record` costs; per-city stockpile value types |
| Nations & Mock | `enum Personality` (7) with roll tables; `enum IntentType` (28); **`union`/`record`** Intent payload; exhaustive `match` in the dispatcher; `Result`/`Option` for lookups; the Mock cascade as clean guarded `match` |
| UI (Win95) | fillRect/bevel drawing on Batch2D; **reflection** to power the Inspector over agent/nation fields; `property` for computed readouts |
| Core & clock | real-time scaled clock; the 3 staggered cadences; **`async`/`await`** for the decision scheduler (the original's async rodízio); `defer`/`using` for frame/GL resources; `Result` for save/load |
| Pathfinding | flow-field BFS; `Option` step; generics over destinations |

Each phase's commit notes which features it lit up.

---

## Phased build plan

- **Phase 0 — Reset (done).** Strip the wrong game; keep the engine primitives (Vec2, Rng, Batch2D,
  Painter/GlPainter/SoftPainter); 1280×720 window skeleton; this blueprint + reference specs committed.
- **Phase 1 — Isometric world.** 480×480 grid + Perlin/mask worldgen + biome mapping + **isometric
  diamond render** (transform, depth-sort, Mountain/Snow z-stack) + resources + camera pan/zoom.
  *Verify: the world reads as the original's isometric map (readPixels→PPM→PNG).*
- **Phase 2 — Agents.** Worker data model + hierarchical FSM + hunger + professions + 4-direction iso
  sprites + object pooling + the 3 tick cadences + flow-field movement + reproduction.
- **Phase 3 — Economy.** Resources + per-city stockpiles + the 9 buildings (as catalogs) + production/
  refinement + BuildingGrowthSystem + construction.
- **Phase 4 — Nations & Mock.** Nation/protagonist model + 28 intents + the full Mock cascade + intent
  dispatch + succession + war/conquest.
- **Phase 5 — UI & Inspector.** Win95 HUD (top bar, Wars, Recent-Intents), click-to-inspect Inspector,
  speed controls — via reflection + Batch2D bevels.
- **Phase 6 — Shell.** Main Menu / New Game / Load / Options / Pause; the real-time scaled clock;
  seed-based save/load; keyboard shortcuts.
- **Phase 7 — Complete the missing.** Worldbox god-powers; full worldgen (plates/erosion/rivers/
  terrain+biome layers); thirst/energy; the 6 extra buildings + Gold economy; the stubbed intents &
  diplomacy; nation-origin modes; full-state save.

Every phase stays faithful to the reference specs, exercises its language features, and is verified by
rendering (GL readback + the software renderer) before moving on.
