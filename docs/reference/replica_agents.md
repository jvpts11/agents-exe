# agents.exe — Worker Agents (Tipo 1) — Faithful-Replica Spec

Reverse-engineered from `unity-project/Assets/Scripts/Agents/` (+ two cross-referenced files in
`Assets/Scripts/Economy/`). All paths below are relative to
`.../scratchpad/agents_src/unity-project/Assets/Scripts/`. Line numbers cited as `File.cs:NN`.

Namespace: `AgenticGame.Agents`. Class hierarchy: `Worker : Agent : MonoBehaviour`.

Unity semantics to replicate: `Time.time` = seconds since level load (monotonic float); `Time.deltaTime`
= seconds since last frame. All timers below are **wall-clock seconds**, and many per-tick effects
accumulate `Time.deltaTime` — so effects are frame-rate independent unless noted. `UnityEngine.Random.value`
∈ [0,1); `Random.Range(a,b)` float is [a,b), int is [a,b).

---

## 0. Cross-referenced constants (outside Agents/, but load-bearing)

- `Economy/Combat/CombatStats.cs` — combat tuning (soldier logic reads these):
  - `SOLDIER_HP = 150` (CombatStats.cs:9) — **DEAD CONSTANT, never applied.** Confirmed only
    reference in the codebase is the definition itself. Every worker (soldier included) actually has
    **100 HP** (Agent.cs:13, ResetForReuse Agent.cs:90, SoldierHpBar `_maxHpCache = 100` SoldierHpBar.cs:19).
  - `WORKER_HP = 100` (CombatStats.cs:10) — also unused; informational.
  - `DAMAGE_VS_WORKER = 15`, `DAMAGE_VS_SOLDIER = 15`, `DAMAGE_VS_BUILDING = 10` (CombatStats.cs:19-21).
  - `ATTACK_INTERVAL = 0.5f` s (CombatStats.cs:23).
  - `COMBAT_RANGE_SQ = 2.25f` (enter-attack range² ⇒ 1.5 world units) (CombatStats.cs:25).
  - `COMBAT_HOLD_RANGE_SQ = 4.0f` (leave-attack range² ⇒ 2.0 units; hysteresis) (CombatStats.cs:27).
  - `TARGET_SEARCH_RADIUS = 30` cells (CombatStats.cs:29); `TARGET_SEARCH_RADIUS_EXIT = 25` (CombatStats.cs:31).
  - `MILITARY_PROXIMITY_RADIUS = 8`, `..._SQ = 64` (soldier-vs-soldier engage gate) (CombatStats.cs:33-34).
  - `TARGET_RESEARCH_INTERVAL = 3f` s (CombatStats.cs:35). `PATROL_RADIUS = 8f` (unused here).
- `Economy/HungerSystem.cs` — the survival driver (see §4).
- `Nations/ResourceStockpile.cs:3` — `enum ResourceType { Food, Wood, Stone, Iron, Gold, Planks }`.
- `Economy/BuildingType.cs:6` — `enum BuildingType { Castle, TownHall, Farm, Sawmill, HuntersLodge,
  FishingHut, Smithy, Bakery, Brewery, Forge, Marketplace, Barracks, ... }` (Barracks used by soldiers;
  Farm/HuntersLodge/Sawmill/TownHall used by profession caps).

---

## 1. Worker data model

### 1a. `Agent` base fields (Agent.cs)

| Field | Type | Initial / rule | Line |
|---|---|---|---|
| `ADULT_AGE_SECONDS` | const float | `60f` (child→adult threshold) | :9 |
| `_position` | Vector2 | (0,0); setter mirrors to `transform.position=(x,y,0)` | :11, :23-31,:113 |
| `_direction` | Direction | `Direction.SE` default; setter fires `OnDirectionChanged` → renderer | :12,:46-55 |
| `_hp` | int | `100` | :13 |
| `_nationId` | int | `-1` | :14 |
| `_lifetimeSeconds` | float | `180f` default (overwritten at spawn, see §7) | :15 |
| `Gender` | Gender | `Gender.Male` default (property) | :17 |
| `Stage` | LifeStage (computed) | `Age >= 60 ? Adult : Child` — **note: never returns Elder** | :19 |
| `_spawnTime` | float | set to `Time.time` on `OnEnable` and `ResetForReuse` | :21,:92,:122 |
| `LocalGrid` | Grid | non-serialized; used for CellPosition | :33 |
| `CellPosition` | Vector2Int (computed) | `LocalGrid.WorldToCell(transform.position)` or round(_position) | :35-44 |
| `Age` | float (computed) | `Time.time - _spawnTime` | :60 |
| `IsAlive` | bool | `_hp > 0 && Age < _lifetimeSeconds` | :62 |
| `IsDead` | bool | `!IsAlive` | :63 |
| `Killer` | Agent | set when HP hits 0 from an attacker | :65,:74 |
| `OnDied` | event Action<Agent> | fires once (`_diedFired` guard) | :78-86 |

Key methods: `TakeDamage(amount, attacker)` (:67-76) — ignores `amount<=0` or already-dead;
subtracts; on reaching ≤0 clamps to 0 and records `Killer` if attacker non-null. `ResetForReuse()`
(:88-94) resets hp=100, spawnTime=now, clears Killer/diedFired. `ForceAge(seconds)` (:96-99) does
`_spawnTime -= seconds` (ages the agent instantly). `LifeStage` enum has `{Child, Adult, Elder}`
(LifeStage.cs:3) but **Elder is never produced** — `Stage` only ever returns Child or Adult.

### 1b. `Worker` fields (Worker.cs)

Movement / tuning (SerializeField defaults):
- `speed = 1.5f` (:38), `wanderSpeed = 1.5f` (:40), `wanderRadius = 5f` (:41).
- `SpeedMultiplier` (float, default `1f`, :93) — multiplies applied movement in `Update()` (:2024).
- `_moveVelocity` (Vector3) — per-tick desired velocity; applied in `Update()`.

Identity / social:
- `_job` / `Profession` (Profession, default `Civilian`) — setter updates sprites + GameObject name
  (:42-54, :56-78).
- `Role` (`ProtagonistRole?`, nullable) — Leader/Consort/Royal; `IsProtagonist => Role.HasValue` (:80,:89).
- `Personality` (Personality) (:81).
- `Father`, `Mother` (Worker) (:82-83); `Children` (List<Worker>) (:85); `IsHeir` (bool) (:87).
- `Family` (FamilyComponent, always non-null) (:99).
- `GetLifetimeMultiplier() => IsProtagonist ? 2.5f : 1f` (:91).
- `DemoteCountdownStart` (float, -1) (:95), `LastDemoteTime` (float, -100) (:97).

Survival / logistics:
- `Hunger` (int, 0..100) (:191); `_lastHungerTick` (float) (:192).
- `Carrying` (CarrySlot struct: `{ResourceType Type; int Amount}`, `IsEmpty => Amount<=0`) (:194, CarrySlot.cs).
- `AssignedCity` (Nations.City) (:196); `_nation` (Nation) (:104).
- Derived flags: `IsHungry => Hunger > 70` (:199); `IsWorking` (woodcutter/stoneminer/ironminer in a
  work/haul state) (:200-205); `IsCarrying => !Carrying.IsEmpty` (:206); `IsIdle => _state==Wandering` (:207).

FSM / job-assignment scratch (per profession): `_assignedNode` (ResourceNode), `_assignedFarm`,
`_assignedBush`, `_assignedLodge`, `_assignedBlueprint`, `_assignedSawmill`, `_assignedBarracks`,
`_tradeSource`/`_tradeDest`/`_tradeResource`, plus flow-fields per activity and per-activity timers.

Combat scratch: `_combatTargetAgent` (Agent), `_combatTargetBuilding` (Building), `_combatFlowField`,
`_attackTimer`, `_combatRetargetAt`, `_directedAttackTarget` (Vector2Int?), `AssignedBarracks`,
`_wasAtWar`. Military flags: `IsGeneral` (:183), `IsMilitary => Soldier||General` (:185).

Stuck-detection: `_blockedFrames` (int), `STUCK_FRAME_THRESHOLD = 300` frames (:1997),
`_stuckCooldownUntil` (float), `STUCK_COOLDOWN_SECONDS = 20f` (:2000), `_consecutiveStuckResets`,
`_lastStuckPos`, `CLUSTER_RADIUS_WORLD = 3f` (:2045).

Per-profession action constants (Worker.cs):
- Chop (tree/mine): `CHOP_INTERVAL = 5f` s (:115), `CHOP_AMOUNT = 10` (:116).
- Farm: `TEND_DURATION = 10f` s (:120).
- Bush (forage): `BUSH_HARVEST_DURATION = 3f` s (:124), `BUSH_CHOP_AMOUNT = 3` (:125).
- Hunt: `HUNT_DURATION = 5f` s (:129), `HUNT_CHOP_AMOUNT = 5` (:130).
- Build: `CONSTRUCT_DURATION = 8f` s (:135).
- Sawyer: `SAWYER_PROCESS_TIME = 4f` s (:154), `SAWYER_WOOD_PER_CYCLE = 1` (:155),
  `SAWYER_PLANKS_PER_CYCLE = 4` (:156).
- Plant: `PLANT_DURATION = 4f` s (:161), `SAWMILL_PLANT_RANGE = 30` cells (:163), `EX_FOREST_SAMPLE_CAP=200` (:165).
- Trader: `TRADER_CARRY_CAP = 25` (:172), `TRADE_SURPLUS_THRESHOLD = 20` (:173),
  `TRADE_DEMAND_THRESHOLD = 5` (:174), priority order `[Planks, Food, Wood, Stone, Iron]` (:176-179).
- March: `MARCH_WAR_SPEED_MULTIPLIER = 1.3f` (:1270).

### 1c. Enums (exact)

- `Direction { NE=0, SE=1, SW=2, NW=3 }` (Direction.cs:5). `FromMovement(v)`: x≥0&y≥0→NE; x≥0&y<0→SE;
  x<0&y<0→SW; else NW (Direction.cs:16-22).
- `Gender { Male, Female }` (Gender.cs:3).
- `LifeStage { Child, Adult, Elder }` (LifeStage.cs:3) — Elder unused.
- `Personality { Aggressive, Cautious, Expansionist, Mercantile, Pious, Cunning, Stoic }` (Personality.cs:3).
- `ProtagonistRole { Leader, Consort, Royal }` (ProtagonistRole.cs:3).
- `Profession { Civilian, Woodcutter, Farmer, Hunter, Builder, Sawyer, TreePlanter, StoneMiner,
  IronMiner, Trader, Soldier, General }` (Profession.cs:4) — 12 values.

### 1d. FamilyComponent (FamilyComponent.cs)

Fields: `Gender`, `Spouse` (Worker), `Mother`, `Father`, `Children` (List<Worker>, cap 4 hint),
`AssignedHouse` (HouseBehavior), `IsPregnant` (bool), `PregnancyEndTime` (float),
`NextProfessionEvalTime` (float). `Clear()` resets all (:20-30). Static `IsAncestor(ancestor,
descendant)` — BFS up the Mother/Father tree, depth-limited to 6 (:32-59), used to block incest in marriage.

---

## 2. Professions / functions

12 professions (§1c). `CurrentActivity` string map at Worker.cs:211-248 (UI labels).
A worker's job is set via the `Profession` property (Worker.cs:44-54): on change it calls
`UpdateProfessionSprites()` and `UpdateProfessionName()` (renames GameObject to
`Soldier_/General_/Worker_ {NationId}_{suffix}`).

**How a worker gets/changes profession:**
- **At spawn:** `ProfessionSelector.PickForNewAdult(nation, city, gender)` (AgentSpawner.cs:138,194);
  children & royals spawn as `Civilian` (AgentSpawner.cs:229,296).
- **Re-evaluation:** `MarriageMatchmaker.AssignProfession` (MarriageMatchmaker.cs:127-142) runs per
  adult city member every marriage tick (2s), but only if `Time.time >= Family.NextProfessionEvalTime`
  (interval `PROFESSION_EVAL_INTERVAL = 10f`, MarriageMatchmaker.cs:15) AND not in stuck-cooldown.
  Calls `ProfessionSelector.ReevaluateFor`; if it returns a different job, sets `Profession` and calls
  `ReconfigureProfession()` (drops carried goods to stockpile, resets FSM state per job, Worker.cs:353-385).

**`ProfessionSelector` (ProfessionSelector.cs) decision logic:**
- `ReevaluateFor` (:23-93): Females who are Soldier/General → demote to Civilian immediately
  (records LastDemoteTime). General stays General. Soldier stays Soldier while nation `IsAtWar`.
  If current job over capacity → Civilian (LastDemoteTime=now). If demoted <15s ago → stay Civilian.
  Else compute `best = PickBestByScore`. If best==current, keep. War+male+best==Soldier → Soldier.
  Demotion to Civilian is **grace-delayed** by `DEMOTE_GRACE_SECONDS = 120f` (:12,:75-83). Switching
  between two non-civilian jobs requires the new score to beat current by `SWITCH_MARGIN = 0.40f`
  (:10,:85-90) — hysteresis to prevent thrash.
- `PickBestByScore` (:95-148): **Priority order:** (1) male + nation at war + soldier slot free +
  no builder need → Soldier (rate-limited per nation by `SOLDIER_PROMO_COOLDOWN_SECONDS = 10f`, :14).
  (2) blueprints exist + builder slot → Builder. (3) city FoodSecurity ≤ Insufficient → Farmer then
  Hunter (if city slot). (4) otherwise argmax of `ScoreFor` over all registered professions (skipping
  Builder; Soldier only for males).
- `ScoreFor(p)` (:150-174): `gapNorm = clamp01(DemandFn/DemandThreshold)`;
  `score = gapNorm * Weight / (1 + currentCount)`; ×1.5 if it matches `nation.FocusedResource`.
  Civilian and General always score 0.

**`ProfessionRegistry` (ProfessionRegistry.cs) — per-profession defs** (Id, Weight, DemandThreshold,
FloorCap, cap formula):

| Prof | Weight | DemandThreshold | FloorCap | Cap formula (constructed buildings) |
|---|---|---|---|---|
| Farmer | 1.5 | 30 | 2 | Farms × `FARMER_PER_FARM(2)` |
| Hunter | 1.4 | 30 | 0 | Lodges × `HUNTER_PER_LODGE(3)` |
| Sawyer | 1.2 | 20 | 2 | Sawmills × `SAWYER_PER_SAWMILL(5)` (demand only if nation wood≥2) |
| TreePlanter | 1.1 | 20 | 2 | Sawmills × `TREE_PLANTER_PER_SAWMILL(1)` |
| Woodcutter | 0.8 | 60 | 2 | TownHalls × `WOODCUTTER_PER_TOWNHALL(4)` |
| StoneMiner | 1.2 | 40 | 2 | TownHalls × `STONE_MINER_PER_TOWNHALL(2)` |
| IronMiner | 1.1 | 20 | 2 | TownHalls × `IRON_MINER_PER_TOWNHALL(2)`; demand keeps a reserve of 5 iron |
| Trader | 0.6 | 1 | 0 | if ≥2 cities: max(4, cities×4), else 0 |
| Builder | 1.0 | 1 | 0 | unconstructedBlueprints × `BUILDER_PER_BLUEPRINT(2)` |
| Soldier | 1.8 | 10 | 0 | Barracks × `SOLDIER_PER_BARRACKS(12)` |

Caps in `ProfessionCapacity.cs`. `CapFor = max(CapFn, FloorCap)` (:33-34). Civilian cap = int.MaxValue;
General cap = 0. `CurrentCount` counts adult members with that job; **Generals count toward the Soldier
count** (:47,:74). Per-city variants exist for Farmer/Hunter/Sawyer/Stone/Iron.

---

## 3. Behavior tree / FSM

There is **no data-driven behavior tree**; behavior is a hand-written hierarchical FSM. The private
`enum WorkerState` (Worker.cs:11-36) has 30+ states grouped by profession. Master dispatch is
`Worker.Tick()` (Worker.cs:423-500), invoked by `AgentSpawner.Update()` (staggered — see §7).

### 3a. Top-level priority (Tick, Worker.cs:423-500)

1. **Dead check (highest priority):** if `IsDead` → zero velocity, log death once (cause: combat if
   Killer, else starvation if Hunger≥100, else old age if Age≥Lifetime, else unknown), release ALL
   assignments (node/farm/lodge/sawmill/barracks/blueprint/combat), free house survivor slot, break
   spouse link, dump carried goods to stockpile, `FireDied()`, return (:425-464).
2. **Not-ready guard:** if `_grid==null` or town-hall flow-field null → zero velocity, return (:466-470).
3. **Child branch:** if `Stage==Child` → `TickWandering()` only. **Children never work/fight/reproduce**
   (:472-474).
4. **Adult branch:** `switch(_job)` → the profession's Tick method (:478-492). General uses `TickSoldier`.
5. **Facing:** if `|_moveVelocity|²>0.001` → set `Direction = FromMovement(velocity)` (:495-499).

Note survival is **not** gated inside `Tick`; hunger is driven externally by `HungerSystem` (§4), and
Civilians get an interrupt: `TickCivilian` (:502-521) forces `_state=BushSeeking` when `IsHungry`
(Hunger>70) unless already foraging/hauling — i.e. a hungry civilian self-forages food. Non-civilian
professions do NOT self-interrupt to eat; they rely on `HungerSystem` feeding from the stockpile.

### 3b. Movement primitives

- **Flow-field follow:** `FlowFieldVelocity(ff, cell, spd)` (:1982-1994) reads the flow direction at the
  current cell, aims at the next cell center, returns unit-vector×spd. This is the walk-to-target
  primitive used by every "MovingToX" state.
- **Wander:** `TickWandering()` (:541-584) — biases toward city center when farther than `wanderRadius`
  (5) via a walkable probe; otherwise picks a random walkable direction (`PickWalkableRandomDir`, 8
  angular probes, :586-602); repicks every `Random(2,4)` s or when a wall is ahead; after >2 consecutive
  stuck-resets it abandons the target bias.
- **Physical step & collision:** `Update()` (:2002-2041) runs every frame (not staggered). Applies
  `delta = velocity × SpeedMultiplier × deltaTime`; tries full move (mid+end walkable), then X-only,
  then Y-only (wall sliding); if all blocked increments `_blockedFrames`; at 300 blocked frames →
  `ResetStuckMovement()`.
- **Stuck recovery:** `ResetStuckMovement()` (:2047-2077) releases all assignments/flowfields, sets
  state=Wandering, and teleports to a nearby walkable cell (`TryJumpToNearbyWalkable`, expanding ring
  radius `min(2+2·consec, 12)`, :2079-2108); sets a 20s cooldown (blocks profession re-eval during it).
- **Walkability:** `IsWalkableAt(worldPos)` (:2122-2129) samples the `bool[,] _walkable` passability map
  (from `PassabilityMap.ForGroundUnits`, AgentSpawner.cs:93).

### 3c. Per-profession sub-FSMs (triggers → effects)

All resource harvesting shares the pattern **Seek → Move → Work(timer) → Haul to capital → Deposit**.
`ResourceNode` reservations use `TryReserve(instanceId)`/`ReleaseReservation`; nodes have `.Cell`,
`.Yields` (ResourceType), `.IsDepleted`, `.Chop(amount)→int`.

- **Woodcutter** (`TickWoodcutter` :654-664): SeekingResource→`TickSeeking(Tree)` (search radius **50**,
  :896) → MovingToResource → Chopping. `TickChopping` (:928-951): accumulate deltaTime to
  `CHOP_INTERVAL(5s)`, then `Chop(10)`, set `Carrying={node.Yields, got}`, → MovingToCapital.
- **StoneMiner / IronMiner** (`TickStoneMiner`/`TickIronMiner` :666-688): `TickMinerSeeking(kind)` search
  radius **80** (:699); MovingToMineNode → Mining; `TickMiningDeposit` (:708-730) same 5s/10-unit chop.
- **Farmer** (`TickFarmer` :732-742): find a `FarmBehavior` in own city with a slot (`TryAssign`);
  WalkingToFarm → Tending; `TickTending` (:779-792) after `TEND_DURATION(10s)` calls
  `farm.OnWorkerArrived(this)` (farm produces food), releases, → MovingToCapital. Falls back to wander
  if no farm.
- **Hunter** (`TickHunter` :794-805): first claim a `HuntersLodge` slot, then HuntingSeek finds a
  `Fauna` node (radius **80**, :831), HuntingMove → Hunting; `TickHunting` (:862-885) after
  `HUNT_DURATION(5s)` `Chop(5)`, `Carrying={Food, got}`, releases lodge, → MovingToCapital.
- **Bush foraging** (Civilian survival; `TickBushSeeking` :604-621, `TickBushChopping` :623-652): find
  `Bush` node (radius **60**, :613), move onto it, after `BUSH_HARVEST_DURATION(3s)` `Chop(3)`,
  `Carrying={Food,got}`, → MovingToCapital.
- **Builder** (`TickBuilder` :988-999): `GetUnconstructedForCityWithSlot` (prefers
  `nation.PrioritizedBuildingType`), `TryReserveBuilder`; MovingToBlueprint → Constructing;
  `TickConstructing` (:1040-1058) adds deltaTime to `blueprint.ConstructProgress`; at
  `CONSTRUCT_DURATION(8s)` calls `MarkConstructed()`. Multiple builders stack progress.
- **Sawyer** (`TickSawyer` :2131-2151): MovingToCapital → pick up `SAWYER_WOOD_PER_CYCLE(1)` Wood from
  town-hall stockpile (`TrySpend`) → walk to nearest Sawmill drop cell → SawyerProcessing
  (`SAWYER_PROCESS_TIME 4s`) → `Carrying={Planks, SAWYER_PLANKS_PER_CYCLE(4)}` → back to capital →
  Deposit. (1 wood → 4 planks.)
- **TreePlanter** (`TickTreePlanter` :2229-2244): SeekingPlantSpot picks a cell (former-forest tiles
  first, else adjacent-to-tree within `SAWMILL_PLANT_RANGE(30)`, else random territory) → MovingToPlantSpot
  → Planting (`PLANT_DURATION 4s`) → `ResourceNodeSeeder.PlantTreeAt(cell)`.
- **Trader** (`TickTrader` :1784-1803): TraderEvaluating scans all city pairs × priority resources
  `[Planks,Food,Wood,Stone,Iron]`; a source with stock >`TRADE_SURPLUS_THRESHOLD(20)` and a dest that
  needs it (demand query >0 or dest stock <`TRADE_DEMAND_THRESHOLD(5)`) → MovingToSource → PickingUp
  (`min(TRADER_CARRY_CAP 25, available)`, `TrySpend`) → MovingToDest → Depositing (adds to dest stockpile).
- **Deposit** (shared `TickDepositing` :969-980): at capital, `CityStockpile().Add(Carrying)`, clear,
  → SeekingResource. Haul target is the town-hall flow-field (`GetTownHallFlowField` :347-351).

### 3d. Soldier / General combat FSM (`TickSoldier` :1060-1174)

Pre-checks each tick: drop target if it's same-nation (friendly fire), dead, or a destroyed building;
if not at war, disengage and go home. If no barracks assigned or state==SeekingBarracks →
`TickSeekingBarracks` (:1543-1575) claims a constructed Barracks (`SOLDIER_PER_BARRACKS 12` cap).
Three combat states:
- **SoldierIdle** (`TickSoldierIdle` :1176-1268): if a `_directedAttackTarget` (player/LLM order) exists
  and is farther than `TARGET_SEARCH_RADIUS(30)` → build flow-field, go MovingToTarget. Else, if at
  peace, march home if >30 cells away. Else every `TARGET_RESEARCH_INTERVAL(3s)` call
  `CombatTargeting.TryFindNearestEnemy`; on hit → acquire target + flow-field → MovingToTarget. Else wander.
- **SoldierMovingToTarget** (`TickSoldierMovingToTarget` :1272-1395): re-acquire if no target; if within
  `COMBAT_RANGE_SQ(2.25)` → SoldierEngaging (reset attackTimer); rebuild flow-field when a moving target
  drifts >1 cell; at war march ×`1.3`; if target <15 units away with clear LOS, beeline directly.
- **SoldierEngaging** (`TickSoldierEngaging` :1416-1541): strafe perpendicular to target at 0.3×speed
  (visual jitter); if target range² > `COMBAT_HOLD_RANGE_SQ(4.0)` → back to MovingToTarget (hysteresis);
  else accumulate to `ATTACK_INTERVAL(0.5s)` and strike: `TakeDamage(15)` vs agent (15 vs both worker &
  soldier), `TakeDamage(10)` vs building. Damage attributes `Killer=this`.

**`CombatTargeting.TryFindNearestEnemy`** (CombatTargeting.cs:14-120): among nations at war, within
radius 30. Target priority: (1) nearest **enemy military within proximity radius 8** (score = dist² +
claims·6400·100 to spread soldiers across targets), else (2) nearest **enemy building** within 30, else
(3) nearest **enemy civilian** within 30. `_claims` counts how many friendly soldiers already target
each enemy, penalizing over-focused targets.

---

## 4. Survival loop (real numbers)

Driven by `Economy/HungerSystem.cs` (a MonoBehaviour singleton), not by `Worker.Tick`.

- **Hunger drain:** `HungerSystem.Update` (:41-60) processes `WORKERS_PER_FRAME = 30` (:14) workers per
  frame round-robin. For each, `worker.TickHunger(now, HUNGER_DRAIN_INTERVAL=7f)` (:12,:53).
  `Worker.TickHunger` (:409-421): if `now - _lastHungerTick >= 7s` → `Hunger = min(100, Hunger+1)`. So
  **+1 hunger every 7 seconds**; 0→100 in ~700 s if never fed.
- **Starvation death:** in `TickHunger`, when `Hunger >= 100` → `Hp = 0` (:415-419). Death is then
  finalized on the next `Worker.Tick` (cause "starvation").
- **Feeding:** after draining, if `Hunger >= HUNGER_HUNGRY_THRESHOLD(30)` and adult, `TryFeedWorker`
  (:62-81): spend **1 Food** from home city stockpile (else any other city of the nation); on success
  `Hunger = 0`. So a well-supplied nation keeps workers oscillating 0↔~30. Children are not fed (and do
  not accrue combat, but DO accrue hunger via TickHunger — only the *feed* is adult-gated).
- **`IsHungry` threshold = 70** (Worker.cs:199) — separate from the feed threshold(30). Civilians with
  Hunger>70 self-interrupt to forage bushes (§3a) if the stockpile hasn't fed them.
- **No thirst, no energy, no sleep, no explicit HP regen.** Those concepts don't exist in this build.
  The only stat depletion is Hunger. HP only changes via combat (`TakeDamage`) and the starvation
  `Hp=0`. There is no rest/sleep action — "Idle" bubble is just the Wandering state.
- **Death causes** (Worker.cs:433-436, resolved at Tick): `combat` (Killer set) | `starvation`
  (Hunger≥100) | `old age` (Age≥LifetimeSeconds) | `unknown`.
- **Old-age "stochastic collapse":** not a random roll — deterministic. Each worker gets
  `LifetimeSeconds = Random.Range(lifetimeMin, lifetimeMax)` at spawn (defaults **360–540 s**,
  AgentSpawner.cs:51-52,:129); protagonists ×2.5 (Worker.cs:91, AgentSpawner.cs:290). `IsAlive` is false
  once `Age >= LifetimeSeconds`. The stochasticity is entirely in the per-agent lifetime roll.
- **Combat death:** HP 100 → 15 dmg/hit at 0.5 s cadence ⇒ ~7 hits (~3.25 s) to kill an unaided worker.

---

## 5. Reproduction

Three parallel systems; all are MonoBehaviour singletons ticking on their own intervals.

### 5a. Commoner reproduction — `PregnancySystem.cs` (Worldbox-style, marriage-gated)

Constants: `TICK_INTERVAL = 1f` s (:12), `PREGNANCY_DURATION = 45f` s (:13),
`HUNGER_THRESHOLD = 30` (:17), `PREGNANCY_CHANCE = 0.5f` (:19).
Per city, per second (`TickNation` :42-103): candidate = female, adult, with a `Spouse` and an
`AssignedHouse` (or already pregnant). For each:
- If pregnant and `Time.time >= PregnancyEndTime` → `Birth` (spawns a child, SpeedMultiplier back to 1).
- Else (conception): require **well-fed** = female.Hunger<30 AND spouse.Hunger<30 (:83-84); spouse must
  also have a house; `CanReproduce(female,city)` (:105-112) = city FoodSecurity ≠ Critical AND city
  Food ≥ `max(10, cityMembers/8)`. If all pass, roll `Random.value < 0.5` → become pregnant, set
  PregnancyEndTime = now+45, and extend lifetime if it would end before birth+10s.
- `Birth` (:114-131): `AgentSpawner.SpawnChildAt(city, motherPos, mother, father)`; registers child to
  both parents' `Children`, assigns to mother's house, records a birth metric.

**Density/food model:** this is the "local adult density + food + saturation" model — conception needs
a free House (housing cap = density limiter), both parents fed (<30 hunger), and city food above a
per-capita floor (`members/8`). Rate ≈ 0.5 per eligible female per second while conditions hold.

### 5b. Marriage — `MarriageMatchmaker.cs`

`TICK_INTERVAL = 2f` s (:13); processes ONE nation per tick round-robin (:33-35). Per city: gather
unspoused adults; for each unmarried male, find an available constructed `HouseBehavior` in the city
(else stop — housing is the bottleneck); pair with first eligible female who is not a blood relative
(`IsParentChild`/`IsAncestor`), assign both to the house (`house.AssignCouple`), set mutual `Spouse`.
Marrying into royalty promotes the commoner spouse (`PromoteWorkerIfMixedRoyalMarriage` :95-125:
Consort if partner is Leader and no Consort yet, else Royal; ×lifetime multiplier; random personality).
Also drives profession re-eval (§2).

### 5c. Royal reproduction — `RoyalReproductionSystem.cs`

`TICK_INTERVAL = 5f` s (:13), `PREGNANCY_DURATION = 45f` (:12), `ROYAL_COOLDOWN_POST_PARTUM = 90f` s
(:11). Leader+Consort couple auto-conceive when eligible (`CanReproduce` = same food gate as §5a,
:227-235), one pregnancy at a time per nation, 90 s post-partum cooldown. Non-heir Royal females may
have **at most 1 child** (`_nonHeirChildrenBorn` cap, :139). `OnRoyalChildBorn` (:169-225): child.Role =
Royal; personality 50% inherited (from a parent) / 50% random; lifetime ×2.5; first royal child with no
existing heir becomes `IsHeir=true`; child scaled to 0.7× size; emits `royal_birth` event + LLM wakeup.

### 5d. Gender & families

Gender assigned at spawn: initial adults alternate M/F by index (AgentSpawner.cs:103); others toggle
`_genderToggle` (:135,:191); children 50/50 (:228). Families tracked via `FamilyComponent` (spouse,
parents, children, house). Succession (`SuccessionSystem.cs`) handles leader death → heir/child/royal/
random-worker promotion, and nation dissolution if <10 living workers and no successor.

---

## 6. Rendering (`AgentRenderer.cs` + helpers)

**Not** GPU-instanced. Each worker is a Unity `GameObject` tree of `SpriteRenderer`s (built in
`AgentSpawner.InstantiateNewWorker` :342-371):
- Root `Worker` GameObject; child **"Sprite"** with `SpriteRenderer` + `AgentRenderer`; child **"Bubble"**
  (`WorkerBubble`); child **"Carry"** (`WorkerCarryIcon`); plus `SoldierHpBar` and `WorkerDamageFlash`
  components on the root.

**`AgentRenderer`** (AgentRenderer.cs): holds `spritesByDirection` (Sprite[4], indexed by `Direction`
enum 0..3 = NE/SE/SW/NW). `SetDirection(d)` swaps `_sr.sprite = spritesByDirection[(int)d]`, `flipX=false`
(no mirroring; each of the 4 iso directions has its own sprite). `SetSprites(ne,se,sw,nw)` replaces the
array. `SetTint(color)` sets `_sr.color` — **per-nation tint** applied at spawn from `nation.Color`
(AgentSpawner.cs:143,198,238,312). Sorting: layer "Default", `sortingOrder = 1`, pivot sort point.

So the visual is a **4-direction isometric billboard sprite, tinted by nation color**. Direction is
derived from movement each tick (`FromMovement`, §3a). Sprite *set* is chosen by profession/role:
- `ProfessionSpriteCatalog.GetSprites(p,...)` (ProfessionSpriteCatalog.cs:33-45): Soldier & General →
  Soldier sprites; **all economic professions → the same Civil sprites** (no per-profession marker for
  civilians). Catalog also has Scout & Protagonist slots (Scout unused by code).
- Protagonists override to Royal sprites (`ProtagonistSpriteCatalog`: RoyalNE/SE/SW/NW + a `Marker`
  sprite), via `Worker.UpdateProfessionSprites` (:387-407). Leaders additionally get a `LeaderMarker`
  overlay (AgentSpawner.AttachLeaderMarker :334-340). Royal children scaled to 0.7.

**Overlays (all polled, not every frame):**
- `WorkerBubble` (WorkerBubble.cs): a status bubble above head (offset y=0.5, sortingOrder 10), updated
  every 0.25 s: Alert sprite if `IsHungry`, Work sprite if `IsWorking`, Sleep sprite if `IsIdle`, else none.
- `WorkerCarryIcon` (WorkerCarryIcon.cs): resource icon above head (offset y=0.7, order 11), shown when
  `IsCarrying`; icon by `Carrying.Type` (Food/Wood/Stone/Iron/Gold/Planks), updated every 0.25 s.
- `SoldierHpBar` (SoldierHpBar.cs): only visible for military & alive. Two quads (bg dark, fg colored),
  width 0.7 × height 0.10, y-offset 0.7, built from a runtime white sprite. Fg width scales with
  `Hp/100` (`_maxHpCache=100`); color green >0.66, yellow >0.33, else red. Orders 4/5.
- `WorkerDamageFlash` (WorkerDamageFlash.cs): tints the Sprite child red `(1,0.25,0.25)` for
  `FLASH_DURATION = 0.18` s whenever HP drops while alive, then restores base color.

---

## 7. Performance / lifecycle (`AgentPool.cs`, `AgentSpawner.cs`)

- **Object pooling:** `AgentPool<T>` (AgentPool.cs) — a `Stack<T>` of inactive agents. `Acquire(factory)`
  pops & `SetActive(true)` + `ResetForReuse()`, or creates via factory (and subscribes `OnDied` →
  `Return`). `Return` deactivates and pushes back. On death, `OnDied` auto-returns the worker to the pool
  (AgentPool.cs:41-44). No hard preallocation; grows on demand.
- **No fixed 3000 cap in this subsystem.** Population is emergent: `workersPerNation = 50` initial adults
  per nation (AgentSpawner.cs:20, spawned 25M/25F alternating), plus a royal couple per nation, plus
  births, minus deaths. Deaths of non-protagonist, non-combat workers **auto-respawn** a replacement
  adult (`OnWorkerDied` :373-405) — so nation population self-heals toward its economic capacity.
  (A global cap, if any, would live in a higher-level manager not in this folder.)
- **Staggered thinking (Worker.Tick):** `AgentSpawner.Update` (:472-479) ticks only every Nth worker
  each frame: `tickGroups = 4` (:24) ⇒ each worker's `Tick()` (the FSM/AI) runs **once every 4 frames**;
  `_currentTickGroup` cycles 0..3. Effects that use `Time.deltaTime` inside Tick still integrate
  correctly because deltaTime spans the skipped frames.
- **Staggered hunger:** `HungerSystem` processes 30 workers/frame round-robin (§4) — independent of the
  Tick stagger.
- **Physical movement (`Worker.Update`) runs every frame** for smooth motion, independent of AI stagger.
- **Spawn seeding:** `spawnSeed = 42` (:21) drives a `System.Random` for spawn-cell placement; note
  gender/lifetime/personality use `UnityEngine.Random` (global, not the seeded RNG). Spawn placement
  prefers walkable territory tiles (`PickSpawnInTerritory`, 200 attempts) or a reachable buffered cell
  (`PickSpawn`, up to 5000 attempts × 4 relaxation passes).
- Diagnostics: `AgentPositionLogger` dumps every worker's `GetPositionSnapshot()` to a JSONL file every
  5 s (id, pos, prof, stage, hp, age, state, intent, velocity...). Useful as a golden-output oracle for
  validating a 1:1 reimplementation.

---

## 8. Replica gotchas (things easy to get wrong)

1. **Soldiers have 100 HP, not 150.** `SOLDIER_HP=150` is a dead constant. Every worker is 100 HP.
2. **`Stage` is binary** (Child<60s, Adult≥60s). `Elder` in the enum is never produced.
3. **Only one survival stat exists: Hunger** (+1 / 7 s, death at 100). No thirst/energy/sleep/HP-regen.
4. **AI ticks at 1/4 rate** (tickGroups=4), but movement & hunger are on separate cadences. deltaTime
   integration makes work timers wall-clock accurate regardless.
5. **Non-civilians don't self-eat** — they depend on `HungerSystem` feeding from the stockpile; only
   hungry (>70) Civilians self-forage bushes.
6. **Deposit/haul always targets the capital/town-hall flow-field**, not the nearest depot.
7. **Deaths auto-respawn** replacements (except protagonists & combat kills), so population is homeostatic.
8. **Direction is 4-way iso** (NE/SE/SW/NW), derived from velocity sign quadrant; no left/right flip.
9. Reproduction requires **housing** (a free `HouseBehavior`) as the density gate, both parents
   Hunger<30, and city food ≥ max(10, members/8); conception ≈ 0.5/eligible-female/second.
