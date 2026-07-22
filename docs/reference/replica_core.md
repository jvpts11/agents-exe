# agents.exe — Faithful-Replica Spec: Pathfinding, Save, Orchestration (the glue)

Source root: `unity-project/Assets/Scripts/`. All paths below are relative to that root.
Every claim is grounded in code with `file:line`.

---

## 0. TL;DR — the single most important architectural fact

**There is NO GameManager, no central tick loop, no discrete simulation clock.**
The game is a **flat bag of independent `MonoBehaviour` "system" objects**, each living in the
`Game.unity` scene, each driving itself off Unity's per-frame `Update()` and coroutines. The whole
simulation is **real-time, frame-based, scaled by `Time.timeScale`**.

- "Tick" is **not** an integer counter that anything increments. It is a *display/serialization
  derived value*: `tick = Mathf.RoundToInt(Time.time)` — i.e. **rounded seconds of scaled game
  time**. See `UI/HUDController.cs:549`, `LLM/DecisionScheduler.cs:357`, `Nations/...` (`GetCurrentTick`).
- **Year = tick / 60** (integer division). So **60 seconds of scaled game time = 1 year**.
  (`HUDController.cs:550`, `Save/SaveSystem.cs:70`.)
- Speed (1x/2x/4x/pause) = **setting `Time.timeScale`** to `1/2/4/0`. Everything scales because
  every system multiplies motion by `Time.deltaTime` (which is `Time.timeScale`-scaled).

A replica in another engine must reproduce this as: a real-time loop with a global `timeScale`
multiplier and a `gameTime` accumulator (seconds); `tick = round(gameTime)`, `year = tick/60`;
each "system" is an object with its own `Update(dt)` where `dt = realDelta * timeScale`.

---

## 1. The Game scene object graph (`Assets/Scenes/Game.unity`)

The scene is a flat list of GameObjects, most carrying exactly one system component. Enumerated
from `Game.unity` (`m_Name:` entries):

**World / rendering**
- `WorldBootstrap` (`World/WorldBootstrap.cs`) — owns worldgen; produces `Biome[,]`.
- `Grid` — Unity `Grid` (isometric cell↔world mapping); referenced by spawner/agents/HUD.
- `Tilemap_Terrain`, `Tilemap_Territory_Border`, `TerritoryPainter`, `StockpileOverlay`, `WarOverlay` — rendering.
- `Main Camera` (+ `Player/CameraController`).

**Core sim ownership**
- `NationManager` (`Nations/NationManager.cs`) — **singleton**, owns `List<Nation>`, the
  `int[,] _tileNation` territory grid, city registry, and the static `bool[,] _walkable`.
- `Agents` GameObject → `AgentSpawner` (`Agents/AgentSpawner.cs`) — **singleton**, owns
  `List<Worker>`, the per-nation `FlowField` cache, the `AgentPool<Worker>`.
- `BuildingManager` (`Economy/BuildingManager.cs`) — **singleton**, owns buildings.
- `ResourceNodeManager`, `ResourceNodeSeeder` — resource nodes.

**Per-domain systems (each a self-ticking MonoBehaviour)**
- Population/agents: `HungerSystem`, `PregnancySystem`, `MarriageMatchmaker`, `SuccessionSystem`,
  `RoyalReproductionSystem`, `DeathNotification`.
- Nations/territory: `CityFoundingSystem`, `CityMigrationSystem`, `TerritoryExpansionSystem`,
  `NationMetricsTracker`, `BuildingGrowthSystem`.
- Combat: `CombatSystems` GameObject (hosts the `Economy/Combat/*` systems: attack orders,
  conquest, war victory, mobilization, general promotion — see §5), `WarRegistry`.
- Decisions/LLM: `DecisionScheduler`, `ProtagonistDecisionSystem`, `NationStateLogger`.
- UI/glue: `HUD` (`UI/HUDController.cs`), `GameSpeedHud`, `DecisionToastController`,
  `PauseMenuSpawner`, `SettingsMenu`, `UICleanup`, `CityStatusLogger`, `CityScreenshotCapture`,
  `TimestampedLogger`.

**How systems reference each other:** almost entirely through **static `Instance` singletons**
(`NationManager.Instance`, `AgentSpawner.Instance`, `BuildingManager.Instance`,
`ProtagonistDecisionSystem.Instance`, `DecisionScheduler.Instance`, `WarRegistry.Instance`,
`SuccessionSystem.Instance`, `HungerSystem.Instance`, `NationMetricsTracker.Instance`, …) plus a
few **C# events** for wiring (see §6). There is no dependency-injection container and no ownership
tree beyond Unity's scene. `NationManager` is the de-facto root of world state.

**Startup ordering is via the `Awake`→`Start` + event handshake**, not a scripted sequence (§6).

---

## 2. Time model & the "tick" (exact)

| Concept | Formula | Where |
|---|---|---|
| Scaled game time (seconds) | `Time.time` (accumulates `deltaTime`, scaled by `timeScale`) | Unity built-in |
| Displayed / logged tick | `Mathf.RoundToInt(Time.time)` | `HUDController.cs:549`, `DecisionScheduler.cs:357` |
| Displayed / logged year | `tick / 60` | `HUDController.cs:550`, `SaveSystem.cs:70` |
| **Saved** tick | `Mathf.RoundToInt(Time.unscaledTime)` | `PauseMenuController.cs:79` ⚠ uses **unscaled** time |
| Agent age (seconds) | `Time.time - _spawnTime` | `Agent.cs:60` |
| Adulthood threshold | `Age >= 60f` (`ADULT_AGE_SECONDS`) → `LifeStage.Adult` | `Agent.cs:9,19` |

Top bar text (`HUDController.RefreshTexts`, lines 547–557):
`"Year {year}   ·   Tick {tick}   ·   Source: {src}{queued msgs}"`, refreshed every
`REFRESH_S = 0.4f` of **unscaled** time (`Update`, lines 103–107).

⚠ **Replica gotcha:** the HUD/decision tick uses **scaled** `Time.time`; the *save* tick uses
**unscaled** `Time.unscaledTime` (`PauseMenuController.cs:79`). They diverge whenever the game ran
at ≠1x or was paused. Reproduce both exactly.

### Speed multipliers & pause (`HUDController.cs`)
- Bottom-bar buttons "Pause/1x/2x/4x" set `Time.timeScale = 0/1/2/4` (`BuildBottomBar`, lines 455–470).
- Keyboard: `Space` = toggle pause (`ToggleSpeed`, 117–121; remembers `_lastNonZeroSpeed`);
  `1`/`2`/`3` keys → `SetSpeed(1/2/4)` (lines 96–101, 111–116). Only active when no input field focused.
- `PauseMenuController.OnResumeClicked` forces `Time.timeScale = 1f` (line 46);
  `OnQuitToMenuClicked` restores `1f` before scene change (line 101).

### Staggered ("tick group") ticking of agents (`AgentSpawner.Update`, lines 472–479)
The heaviest per-frame cost — ticking N workers — is spread across `tickGroups = 4` frames:
```
Update():
  n = max(1, tickGroups)              // 4
  for (i = _currentTickGroup; i < _workers.Count; i += n) _workers[i].Tick();
  _currentTickGroup = (_currentTickGroup + 1) % n
```
So each worker's `Tick()` (AI/state machine) runs **once every 4 frames** (round-robin), while its
`Worker.Update()` (locomotion) runs **every frame** (`Worker.cs:2002`). Replica must keep this split:
**decision/AI cadence = every 4th frame per agent; movement integration = every frame.**

---

## 3. The per-frame "tick loop" — ordered

There is no single loop; Unity calls each enabled MonoBehaviour's `Update()` once per frame. Within
one rendered frame the effective simulation order (grouping by role; intra-group order is Unity's
script order, not guaranteed, and the game does not depend on it except via events) is:

**Per frame:**
1. **Input & speed control** — `HUDController.Update` (map clicks, speed keys), camera.
2. **Agent AI tick (staggered)** — `AgentSpawner.Update` ticks ~1/4 of workers via `Worker.Tick()`
   (`AgentSpawner.cs:472`). Each `Worker.Tick()` is a **state machine** by profession
   (`Worker.cs:423`, switch at 478): Civilian/Woodcutter/Farmer/Hunter/Miner/Builder/Soldier/General,
   plus Child→`TickWandering`. Tick sets `_moveVelocity` (via a flow field) and runs job timers
   (chop/harvest/tend/build/attack). Dead workers release all reservations here (`Worker.cs:425–469`).
3. **Agent locomotion (every frame)** — `Worker.Update` (`Worker.cs:2002`) integrates
   `Position += moveVelocity * SpeedMultiplier * Time.deltaTime` with wall-sliding (§4).
4. **Economy/production systems** — `HungerSystem`, `FarmBehavior`, `SawmillBehavior`,
   `HuntersLodgeBehavior`, `BarracksBehavior`, `BuildingGrowthSystem` each run their own interval
   timers in `Update`.
5. **Population systems** — `PregnancySystem`, `MarriageMatchmaker`, `RoyalReproductionSystem`,
   `SuccessionSystem` (interval-gated in their `Update`s).
6. **Nation/territory systems** — `CityFoundingSystem`, `CityMigrationSystem`,
   `TerritoryExpansionSystem`, `NationMetricsTracker`.
7. **Combat systems** — `Economy/Combat/*` (targeting, attack orders, conquest, war victory).
8. **Decision scheduling** — `DecisionScheduler.Update` (`DecisionScheduler.cs:60`):
   `ProcessTickedNations()` then `DrainEventQueue()` (§7). This is the closest thing to a discrete
   "nation turn", and it is **interval-based (default 45 s), staggered per nation, async**.
9. **UI refresh** — `HUDController.RefreshTexts` (every 0.4 s unscaled), overlays, toasts, loggers.

**Spawning** is event-driven, not part of a loop step: initial spawn on `OnNationsReady`
(`AgentSpawner.cs:87`); respawn on death via `OnWorkerDied` (`AgentSpawner.cs:373–405`) — a
non-combat, non-protagonist death immediately spawns a replacement adult in the same city.
**Cleanup** (releasing nodes/farms/houses, removing from lists) happens inside `Worker.Tick`'s dead
branch (`Worker.cs:425–469`) and `AgentSpawner.OnWorkerDied`.

---

## 4. Pathfinding — **flow fields only; no A***

Files: `Pathfinding/FlowField.cs`, `Pathfinding/PassabilityMap.cs`. Grep across all of `Scripts/`
confirms **there is no A*/Dijkstra/priority-queue pathfinder anywhere** — every mover uses a
`FlowField`. Local obstacle avoidance is wall-sliding, and children use a random walk.

### 4.1 Passability (`PassabilityMap.cs:8-22`)
`ForGroundUnits(Biome[,])` → `bool[,]` where a cell is walkable iff
`biome == Grassland || biome == Beach`. **Buildings do NOT affect walkability** — the walkable grid
is a pure function of biomes and is **static for the whole session** (biomes never change). This is
why there is no "map-edit invalidation" of flow fields (see 4.4).

### 4.2 FlowField.Build (`FlowField.cs:27-91`) — multi-source-free BFS from one target
A **uniform-cost BFS (unweighted, FIFO queue)** from a single `target` cell over 8 neighbors:
- Data: `int[,] Distance` (init `int.MaxValue`; `FlowField.cs:45-47`), `Vector2Int[,] Direction`
  (the step FROM a cell TOWARD the target, stored as `(-dx,-dy)`; line 85), plus `Width/Height/Target`.
- Deltas = 8 directions (4 orthogonal + 4 diagonal), `FlowField.cs:15-21`.
- If target is out of bounds or non-walkable → returns an all-`MaxValue` field (unreachable) (55-59).
- BFS: `distance[target]=0`, enqueue target; pop cell, for each neighbor: skip if OOB / non-walkable
  / already visited; **diagonal moves require BOTH orthogonal side cells walkable** (no corner
  cutting; lines 79-83); set `distance = cur+1`, `direction = (-dx,-dy)`, enqueue (61-88).
- Because BFS is unweighted, `Distance` = number of (chebyshev) steps, not euclidean length.
- Accessors: `DirectionAt(x,y)` → step vector or `zero` (93-97); `IsReachable(x,y)` →
  `Distance != MaxValue` (99-103).
- Static instrumentation only (`FlowField.cs:23-39`): counts rebuilds, logs rebuild-rate every 30 s.
  No behavioral effect.

### 4.3 How movers consume the field (`Worker.cs`)
`FlowFieldVelocity(ff, cell, spd)` (`Worker.cs:1982-1994`): read `flowDir = ff.DirectionAt(cell)`;
if `zero` → stop; else compute world center of `cell + flowDir`, return unit vector toward it × speed.
`Worker.Update` (`Worker.cs:2002-2032+`) integrates it every frame:
`delta = moveVelocity * SpeedMultiplier * Time.deltaTime`; tries full move (checks `newPos` and the
midpoint for walkability), else **x-only slide**, else **y-only slide**, else counts blocked frames
(`STUCK_FRAME_THRESHOLD = 300`; soldiers reset their route when stuck, `Worker.cs:1996-2018`).
Children don't use a field: `TickWandering` (`Worker.cs:541+`) does a bounded random walk toward the
city center with wall probing (`PickWalkableRandomDir`).

### 4.4 Which destinations get a field, and recompute triggers
Fields are built **per-destination, on demand, and cached**; a field is rebuilt only when its
**target changes** (not on any map edit — the map is static):
- **Per-nation capital field**: `AgentSpawner.OnNationsReady` builds one `FlowField.Build(walkable, capital)`
  per nation, cached in `_flowFieldByNation` (`AgentSpawner.cs:98-99`), reused for spawn placement.
- **Per-city field** `City.CapitalFlowField`: built when a city is created
  (`NationManager.CreateCapitalCity:175`, `FoundCity:200`); workers path "home" via
  `GetTownHallFlowField()` → `city.CapitalFlowField` (`Worker.cs:347-350, 525, 955-966`).
- **Resource fields** (`_resourceFlowField`): rebuilt each time a worker targets a specific node/
  bush/farm/fauna/mine cell (`Worker.cs:618, 704, 756, 836, 901`), cleared on arrival.
- **Blueprint field** (`_blueprintFlowField`): built toward a construction site (`Worker.cs:1015`).
- **Combat field** (`_combatFlowField` + `_combatFlowFieldTarget`): rebuilt **only when the target
  cell differs** from the cached target (`Worker.cs:1202-1204, 1231-1234, 1252, 1283-1299, 1354-1360`) —
  target may be a directed attack cell, enemy position, or home center.
- **Trade / plant fields**: `_tradeFlowField`, `_plantFlowField` (`Worker.cs:159-170`).
- **Capital selection** also uses fields: `NationManager.PickCapitals` builds a field from the first
  capital to test reachability of other candidate capitals (`NationManager.cs:127-128`).

**Recompute triggers (complete list):** (a) new city founded → build its field; (b) a worker picks a
new resource/blueprint/combat destination whose cell ≠ cached target. **No trigger on building
placement or terrain change**, because walkability ignores both.

---

## 5. Combat / conquest wiring (glue only)
Under `Economy/Combat/` (hosted by the `CombatSystems` GameObject): `CombatTargeting`,
`AttackOrder`+`AttackOrderSystem`, `CombatStats`, `CombatEventRecorder`, `ConquestSystem`,
`WarVictorySystem`, `MilitaryMobilization`, `GeneralPromoter`. Wars are tracked by
`Nations/WarRegistry.cs` (singleton, `OnWarDeclared` event). Kills are reported from
`AgentSpawner.OnWorkerDied` → `NationMetricsTracker.RecordDeath` and `CombatEventRecorder.RegisterKill`
(`AgentSpawner.cs:382-388`). These feed the DecisionScheduler's event queue (§7). Deep combat rules
are out of this subsystem's scope; the *glue* is: soldiers move by combat flow fields (§4.4), deaths
route through the spawner, wars route through `WarRegistry` → `DecisionScheduler`.

---

## 6. Bootstrapping — New Game & the Awake→Start→event handshake

### 6.1 Scene flow (build order in `ProjectSettings/EditorBuildSettings.asset`)
Index 0 `MainMenu` → 1 `NewGameMenu` → 2 `LoadGameMenu` → 3 `OptionsMenu` → 4 `Game`.
(There is also an unused `SampleScene.unity`, not in build settings.)

- `MainMenuController` (`UI/MainMenuController.cs`): scene-name constants (9-12);
  New Game → load `"NewGameMenu"` (or fall back to `"Game"`); Load → `"LoadGameMenu"`;
  Options → `"OptionsMenu"`; Quit → `Application.Quit`. On `Start` it clears any pending load (17).
- `NewGameMenuController.OnStartClicked` (`UI/NewGameMenuController.cs:26-40`): reads seed input,
  sets `GameSessionState.PendingWorldGen` (a `WorldGenParams` with the parsed `Seed`),
  `GameSessionState.PendingDecisionSource = LLM if useLlmToggle else Mock`,
  `MenuOverrideActive = true`, clears pending load, then `SceneManager.LoadScene("Game")`.

### 6.2 `GameSessionState` — the cross-scene hand-off (`UI/GameSessionState.cs`)
A `static` class (survives scene loads): `PendingWorldGen` (WorldGenParams),
`PendingDecisionSource` (Mock/LLM), `PendingLoadSlotPath` (string), `MenuOverrideActive` (bool),
`ClearPendingLoad()`. This is the *only* channel from menu → game scene.

### 6.3 In-Game init sequence (event handshake, not a script)
1. `WorldBootstrap.Awake` copies `worldGenParams = GameSessionState.PendingWorldGen` (`WorldBootstrap.cs:32`).
2. `WorldBootstrap.Start` → `Regenerate()` (37, 41): if `!MenuOverrideActive` may apply inspector
   seed override / randomize (49-55); then `HeightmapGenerator.Generate` → `BiomeMapper.Map` →
   `tilemapPainter.Paint`; stores `_biomes`; fires `OnGenerated(biomes)` (59-69).
3. `NationManager.Start` (`NationManager.cs:45-53`): if biomes already exist, run `OnWorldGenerated`
   immediately, else subscribe to `WorldBootstrap.OnGenerated`.
   `OnWorldGenerated` (60-82): build `_tileNation` (all -1), `_walkable = PassabilityMap.ForGroundUnits`,
   `PickCapitals` (seeded `System.Random(capitalSeed=7)`; far-apart, reachable, min area; §nation-origin),
   create `numNations=4` nations each with a capital city + initial radius-10 territory; set `_ready`;
   fire `OnNationsReady`.
4. `AgentSpawner.Start` (`AgentSpawner.cs:70-79`): if `NationManager.IsReady` run now, else subscribe
   to `OnNationsReady`. `OnNationsReady` (87-111): rebuild walkable, `System.Random(spawnSeed=42)`;
   per nation build capital flow field, spawn `workersPerNation=50` adults (alternating M/F), then a
   **royal couple** (`SpawnRoyalCouple`, 256-272: one Leader + one Consort, random which gender leads).
5. `ProtagonistDecisionSystem.Awake` (`LLM/ProtagonistDecisionSystem.cs:17-23`): `ActiveSource =
   GameSessionState.PendingDecisionSource` (fallback `defaultSource = Mock`).
6. `DecisionScheduler.Start` → `HookEvents` (§7).

### 6.4 Nation-origin mode: **pre-placement, not organic emergence**
Nations exist from t=0 via `NationManager.PickCapitals` + `CreateNation`/`CreateCapitalCity`
(`NationManager.cs:87-194`). Capitals are chosen deterministically (`capitalSeed=7`) with constraints:
`minCapitalDistance=100` (Manhattan), `CountReachableArea ≥ MIN_REACHABLE_AREA=400` within
`LEBENSRAUM_RADIUS=20`, and reachability from capital[0] via a flow field (`NationManager.cs:105-156`).
Organic new cities *can* form later via `CityFoundingSystem`, but the **initial** nations are
pre-placed. Workers spawn inside territory (`spawnInTerritory=true`, `AgentSpawner.cs:26,118-120`).

---

## 7. DecisionScheduler — the nation "decision tick" (`LLM/DecisionScheduler.cs`)
The one genuinely tick-like system, but still interval/real-time based, not a fixed step.
- `tickInterval` default **45 s** (clamped 30–300, `DecisionScheduler.cs:13-19`).
- `Update` (60-64): `ProcessTickedNations()` then `DrainEventQueue()`.
- `ProcessTickedNations` (175-199): per nation, first time it schedules `_nextTickAt[id] = now +
  id*interval/nationCount` — a **staggered** phase offset so the 4 nations don't all decide on the
  same frame; thereafter every `now >= nextAt` it enqueues a `{NationId, "tick"}` request, advances
  `nextAt += interval`, and runs `CheckFoodCrash` (60 s cooldown) + `CheckArmyDecimation` (<50% of
  baseline → wake-up).
- **Event queue** (`_eventQueue`) also gets *wake-ups* from events wired in `HookEvents` (90-107):
  `WarRegistry.OnWarDeclared` → both belligerents; `SuccessionSystem.OnLeaderSuccession` /
  `OnNationDissolved`; `BuildingManager.OnBuildingRegistered` → hook `Building.OnDestroyed`
  (castle/townhall/barracks loss → "city_lost"/"building_destroyed" wake-up, 129-161). Player chat
  enqueues `"player_message"` (373-383).
- `DrainEventQueue` (253-292): if source is `None`, freeze (skip). Else drain **≤ 2 per frame**
  (`MAX_DRAIN_PER_FRAME`), one per nation per frame, re-enqueueing if a nation was already processed
  or is on cooldown (`CanProcessNow`: Mock always ok; LLM gated by `DECISION_COOLDOWN_REAL=15 s` real
  time, 338-350).
- `ProcessRequest` (async, 294-336): serialize nation via `NationStateSerializer.Serialize(nation,
  GetCurrentTick(), playerMsg)` → `ProtagonistDecisionSystem.GetIntent` (Mock heuristic or LLM, with
  LLM→Mock fallback, `ProtagonistDecisionSystem.cs:35-72`) → `IntentDispatcher.Apply(nation, intent,
  source)` mutates the nation. Recent events recorded with `Mathf.RoundToInt(Time.time)` as the tick.

---

## 8. Save system (`Save/SaveSystem.cs`)

### 8.1 Schema (`SaveFile`, lines 21-34) — JSON via Newtonsoft, `Formatting.Indented`
```
schema_version : "save_v1"   (const SCHEMA_VERSION, line 16)
saved_at_utc   : DateTime.UtcNow "o" (ISO-8601 round-trip)
slot           : int
label          : string          (default "Slot {n}" or "Save HH:mm")
tick           : int             (Mathf.RoundToInt(Time.unscaledTime) — from PauseMenuController:79)
year           : int  = tick / 60            (SaveSystem.cs:70)
world_gen      : WorldGenParams  (the pending worldgen — seed etc.)
decision_source: string          ("Mock"/"LLM"/"None")
nation_states_json : List<string>  (one serialized NationStateDoc per nation, as a JSON string each)
```

### 8.2 Slot layout & API
- `MAX_SLOTS = 5` (line 17). Dir = `Application.persistentDataPath + "/saves"` (line 19); on Windows
  that resolves under `%APPDATA%\..\LocalLow\<Company>\<Product>\saves\`. File =
  `slot_{slot}.json` (`SlotPath`, 36-40; `EnsureDir` creates the folder, 122-126).
- `Save(slot,label,tick)` (60-89): grabs `NationManager.Instance`; builds `SaveFile`;
  `world_gen = GameSessionState.PendingWorldGen` (via `WorldBootstrapAccessor2.GetWorldGenParams`,
  129-140 — note: returns the *pending* params, NOT live world state);
  `decision_source = ProtagonistDecisionSystem.Instance.ActiveSource` (or Mock);
  for each nation: `NationStateSerializer.Serialize(nation, tick, null)` →
  `NationStateLogger.SerializeDoc(doc)` appended to `nation_states_json`;
  `File.WriteAllText`.
- `ReadSlotMeta` (44-58) parses for menu listing (try/catch, warns on failure).
- `Load(slot)` (91-114): reads + deserializes; **rejects if `schema_version != "save_v1"`** (102-106).
- `DeleteSlot` (116-120).

### 8.3 ⚠ What Load actually restores — **only the seed**
This is the single biggest replica trap. Trace the load path:
- `LoadGameMenuController.LoadSlot` (`UI/LoadGameMenuController.cs:44-52`):
  `sf = SaveSystem.Load(slot)`; sets `GameSessionState.PendingWorldGen = sf.world_gen`,
  `GameSessionState.PendingLoadSlotPath = SaveSystem.SlotPath(slot)`, `MenuOverrideActive = true`,
  then `LoadScene("Game")`.
- **`GameSessionState.PendingLoadSlotPath` is written but NEVER read anywhere** (grep: only set in
  `LoadGameMenuController.cs:49`, cleared in `GameSessionState.ClearPendingLoad`). Likewise the saved
  `nation_states_json`, `tick`, `decision_source` are **never re-hydrated** — nothing consumes them
  on load.
- Therefore **"Load" == "New Game with the saved world_gen seed"**: `WorldBootstrap.Awake` pulls
  `PendingWorldGen` and regenerates the same deterministic world; nations/agents/economy start fresh
  from t=0 with the fixed seeds (`capitalSeed=7`, `spawnSeed=42`). The save file's nation snapshots
  are effectively **write-only telemetry** (for the LLM/dataset pipeline), not a restore point.

A faithful replica must reproduce this: saving serializes full nation docs + tick, but loading
restores *only* `world_gen` (the seed) and re-runs deterministic bootstrap. (If the intent is a true
save/load, that is a divergence from the shipped behavior and must be an explicit, separate decision.)

---

## 9. Object-graph cheat-sheet for the replica

```
GameSessionState (static, cross-scene)
   └─ PendingWorldGen (seed), PendingDecisionSource, PendingLoadSlotPath(unused), MenuOverrideActive

WorldBootstrap ── OnGenerated(Biome[,]) ──▶ NationManager
   (HeightmapGenerator → BiomeMapper → TilemapPainter)              │
                                                                    ▼
NationManager (Instance)  owns: List<Nation>, int[,] _tileNation, bool[,] _walkable,
   cellToCity, capitals(FlowField reachability). Events: OnNationsReady, OnTerritoryChanged,
   OnCityFounded.                                                   │
                                        OnNationsReady ─────────────┤
                                                                    ▼
AgentSpawner (Instance) owns: List<Worker>, _flowFieldByNation, AgentPool<Worker>.
   Update() → staggered Worker.Tick() (tickGroups=4). Respawn on OnWorkerDied.
   Each Worker.Update() integrates flow-field velocity every frame.

BuildingManager (Instance) owns buildings. ProtagonistDecisionSystem (Instance) picks Mock/LLM.
DecisionScheduler (Instance) interval(45s, staggered) + event queue → IntentDispatcher.Apply(nation).
WarRegistry / SuccessionSystem / HungerSystem / NationMetricsTracker … = singletons feeding events.

HUDController: tick = round(Time.time), year = tick/60; speed buttons/keys set Time.timeScale.
SaveSystem: writes slot_N.json (save_v1); load restores ONLY world_gen seed.
```

## 10. Concrete constants to copy
- `ADULT_AGE_SECONDS = 60` (`Agent.cs:9`); default worker lifetime `360–540 s` (`AgentSpawner.cs:51-52`).
- `numNations = 4`, `minCapitalDistance = 100`, `initialRadius = 10`, `capitalSeed = 7`
  (`NationManager.cs:15-20`); `LEBENSRAUM_RADIUS = 20`, `MIN_REACHABLE_AREA = 400` (84-85);
  capital candidate pool capped at 2000 shuffled walkable tiles (92-103).
- `workersPerNation = 50`, `spawnSeed = 42`, `tickGroups = 4`, `spawnBuffer = 2` (`AgentSpawner.cs:20-29`).
- FlowField `FLUSH_INTERVAL = 30` (log only); 8-neighborhood, no corner cutting (`FlowField.cs:15-25,79-83`).
- DecisionScheduler `tickInterval = 45` (30–300), `DECISION_COOLDOWN_REAL = 15`,
  `FOOD_CRASH_COOLDOWN = 60`, `MAX_DRAIN_PER_FRAME = 2` (`DecisionScheduler.cs:13,21,23,263`).
- Save `SCHEMA_VERSION = "save_v1"`, `MAX_SLOTS = 5`, year = tick/60 (`SaveSystem.cs:16,17,70`).
- Worker stuck: `STUCK_FRAME_THRESHOLD = 300`, `STUCK_COOLDOWN_SECONDS = 20` (`Worker.cs:1997-2000`).
