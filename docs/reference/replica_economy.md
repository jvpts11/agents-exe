# agents.exe — Economy Subsystem: Faithful-Replica Spec

Reverse-engineered from the shipped Unity C# source. All paths are absolute; `file:line` refs point to the exact code.

Source root: `agents_src/unity-project/Assets/Scripts/`
- Economy code: `Economy/` (31 files) + subfolder `Economy/Combat/`
- Cross-cutting deps live in `Nations/` (ResourceStockpile, City, Nation, DemandQueries, CityStateEvaluator, CityFoundingSystem) and `Agents/` (Profession, Worker, ProfessionRegistry, ProfessionCapacity).
- The `Assets/Catalogs/*.asset` files are **sprite-only ScriptableObjects** (visual). All *numeric* economy data is hard-coded in static C# registries, not in data assets.

---

## 1. RESOURCE TYPES

### 1.1 Enum (`Nations/ResourceStockpile.cs:3`)
```csharp
public enum ResourceType { Food, Wood, Stone, Iron, Gold, Planks }
```
Ordinals are load-bearing (used elsewhere as `(int)type`): **Food=0, Wood=1, Stone=2, Iron=3, Gold=4, Planks=5**.

| Resource | Produced by | Consumed by | Notes |
|----------|-------------|-------------|-------|
| **Food** | Farmer (Farm), Hunter (Fauna), Civilian (Bush) | Eaten by hungry workers (1/feed) | The survival resource |
| **Wood** | Woodcutter (Tree nodes) | Sawyer input (1 Wood → 4 Planks) | Raw material |
| **Stone** | StoneMiner (StoneNode) | Building costs | Raw material |
| **Iron** | IronMiner (IronNode) | Building costs (Lodge/Sawmill/Barracks) | Raw material |
| **Planks** | Sawyer (processes Wood at Sawmill) | Building costs (every building) | The refined build material |
| **Gold** | **nothing** | **nothing** | Declared in enum + stockpile but never produced or spent anywhere in the economy. Vestigial/reserved. |

### 1.2 Stockpile mechanics (`Nations/ResourceStockpile.cs`)
`ResourceStockpile` is a plain class with 6 public `int` fields (`Food,Wood,Stone,Iron,Gold,Planks`) plus:
- `Add(type, amount)` — ignores amount ≤ 0 (`:16`).
- `Get(type)` (`:30`).
- `TrySpend(type, amount)` — fails if insufficient, else deducts (`:44`).
- `ForceSpend(type, amount)` — deducts unconditionally, floors at `STOCKPILE_MIN = -999` (`:14,:66`). Used for war-chest overspend.

### 1.3 Where stockpiles live — **two tiers**
- **Per-City**: `City.Stockpile` (`Nations/City.cs:21,30`). This is the primary store.
- **Per-Nation**: `Nation.Stockpile` (`Nations/Nation.cs:23`). Fallback only.
- Workers deposit to `AssignedCity ?? _nation.CapitalCity` stockpile, falling back to `_nation.Stockpile` (`Agents/Worker.cs:982-986` `CityStockpile()`).
- Building costs are spent **across all cities of the nation** — primary city first, then siblings (`Economy/BuildingGrowthSystem.cs:318-333` `SpendAcrossCities`).
- Affordability is likewise nation-wide (sum of all city stockpiles) (`BuildingGrowthSystem.cs:239-252` `IsAffordable`).

### 1.4 Resource nodes (the map deposits) — `Economy/ResourceNode.cs`, `ResourceNodeKind.cs`
```csharp
public enum ResourceNodeKind { Tree, StoneNode, IronNode, Bush, Fauna }   // ResourceNodeKind.cs:3
```
`ResourceNode.Init` maps kind → yielded resource (`ResourceNode.cs:36-44`):

| NodeKind | Yields | Seeded capacity | Spawn rule (`ResourceNodeSeeder.cs`) |
|----------|--------|-----------------|--------------------------------------|
| Tree | Wood | **10** | Forest clusters (BFS, size 5–25) on Grassland; + 3% scatter on grass; + 4% on Beach (palm). `:132,:177,:203` |
| Bush | Food | **3** | 4% scatter on grass (`:183`) |
| StoneNode | Stone | **5** | 4% scatter on grass (`:189`) |
| IronNode | Iron | **5** | 2% scatter on grass (`:195`) |
| Fauna | Food | **5** | 1.5% scatter on grass; **mobile** (adds `FaunaWander`) (`:227,273`) |

Note: `ResourceNode.Capacity` field-inits to `50` (`ResourceNode.cs:14`) but the seeder **always** overrides via `Init(kind, cell, capacity, sprite)` with the values above — 50 is never the effective value in-game.

Node harvesting API (`ResourceNode.cs`): `TryReserve(workerId)` (single reserver, `-1`=free, `:59`), `Chop(amount)` returns `min(amount, Remaining)`, self-unregisters + deactivates when depleted (`:72-88`). Manager = spatial hash, bucket size 32, `FindNearestAvailable(kind, from, maxRadius, nationFilter)` (`ResourceNodeManager.cs:62`).

---

## 2. BUILDINGS

### 2.1 Enum (`Economy/BuildingType.cs`)
```csharp
public enum BuildingType {
    Castle, TownHall, Farm, Sawmill, HuntersLodge,
    FishingHut, Smithy, Bakery, Brewery, Forge, Marketplace, Barracks,
    House, HouseMedium, HouseLarge,
}
```
**Only 9 of these 15 are actually registered/buildable.** `FishingHut, Smithy, Bakery, Brewery, Forge, Marketplace` exist in the enum, have footprints (`BuildingFootprintCatalog.cs`) and sprites (`BuildingSpriteCatalog`), but have **no `BuildingDef` in `BuildingRegistry`** and **no behavior** → they can never be built. Treat as reserved/planned. FishingHut additionally has a biome rule (Beach-only) but is still unbuildable.

### 2.2 `BuildingDef` schema (`Economy/BuildingRegistry.cs:9-19`)
```csharp
class BuildingDef {
    BuildingType Type;
    (ResourceType resource, int amount)[] Cost;   // build cost
    BuildingType[] PrereqAny;                      // needs ≥1 constructed of ANY listed type in the city
    bool BuildableByGrowthSystem;
    int PerCityCap;                                // max of this type per city (int.MaxValue = unlimited)
    Func<City,int> DemandFn;                        // >0 means "wanted"; magnitude used by workers
    bool IsHousing;
    Type BehaviorType;                             // MonoBehaviour attached on spawn
}
```

### 2.3 COMPLETE BUILDING TABLE (all buildable types, real numbers)

Costs are from `BuildingRegistry.cs:30-125`. Footprint from `BuildingFootprintCatalog.cs:7-25`. HP from `Building.cs:61-67` + `Combat/CombatStats.cs:11-17`. Biome from `BuildingBiomeRules.cs:11-19`. Housing capacity from `HouseBehavior.cs:49-61`. Worker capacity from each behavior's `CapacityFor`.

| Building | Cost (Planks / Stone / Iron) | Prereq (any-of, constructed, in city) | Growth-built? | Per-city cap | Footprint (WxH) | HP | Biome | Behavior | Worker slots | Produces / Enables |
|----------|------------------------------|----------------------------------------|---------------|--------------|-----------------|-----|-------|----------|--------------|---------------------|
| **Castle** | free (empty cost) | — | **No** (initial only) | 1 | 3×3 | **500** | Grassland | none | Capital marker; prereq for Barracks; adds HP bar |
| **TownHall** | free | — | **No** (initial/founding) | 1 | 2×2 | **300** | Grassland | none | City center; caps Woodcutter/StoneMiner/IronMiner; capture point (see §2.6) |
| **Farm** | **8 / 4 / 0** | — | Yes | **6** | 2×2 | 100 | Grassland | 3 Farmers (`FarmBehavior.cs:51`) | Food (+3/tend, needs ≥2 farmers). Prereq for Houses & Lodge |
| **HuntersLodge** | **8 / 4 / 3** | Farm | Yes | **4** | 2×2 | 100 | Grassland **or Beach** | 3 Hunters (`HuntersLodgeBehavior.cs:48`) | Food (via hunting Fauna) |
| **Sawmill** | **6 / 6 / 3** | — | Yes | **3** | 2×2 | 100 | Grassland | 3 Sawyers (`SawmillBehavior.cs:37`) | Planks (drop point; consumes Wood). Input req: Wood×1 (`SawmillBehavior.cs:39`). Enables TreePlanter |
| **House** | **5 / 3 / 0** | Farm | Yes | ∞ | 1×1 | 100 | Grassland | — (housing) | Housing: **1 couple** |
| **HouseMedium** | **12 / 6 / 0** | Farm | Yes | ∞ | 2×2 | 100 | Grassland | — (housing) | Housing: **2 couples** |
| **HouseLarge** | **25 / 12 / 0** | Farm | Yes | ∞ | 3×3 | 100 | Grassland | — (housing) | Housing: **4 couples** |
| **Barracks** | **10 / 8 / 6** | Castle | Yes | **2** | 3×3 | **200** | Grassland | **12** Soldiers/Generals (`CombatStats.SOLDIER_PER_BARRACKS`, `BarracksBehavior.cs:50`) | Trains/houses military; caps Soldier profession |

Cost array reference (exact): Farm `{(Planks,8),(Stone,4)}`; HuntersLodge `{(Planks,8),(Stone,4),(Iron,3)}`; Sawmill `{(Planks,6),(Stone,6),(Iron,3)}`; House `{(Planks,5),(Stone,3)}`; HouseMedium `{(Planks,12),(Stone,6)}`; HouseLarge `{(Planks,25),(Stone,12)}`; Barracks `{(Planks,10),(Stone,8),(Iron,6)}`; Castle & TownHall = `Array.Empty`.

**HP mapping caveat** (`Building.cs:61-67`): the `switch` only special-cases Castle(500)/TownHall(300)/Barracks(200); **everything else → `BUILDING_HP_HOUSE = 100`** (so Farm/Sawmill/Lodge/all houses = 100, even though `CombatStats` also defines FARM/LODGE/SAWMILL=100 constants that are unused). Field default `HpRemaining = 100` (`Building.cs:18`).

### 2.4 Declared-but-unbuildable types (for completeness)
Footprints exist (`BuildingFootprintCatalog.cs`): FishingHut 2×2, Smithy 2×2, Bakery 2×2, Brewery 2×2, Forge 2×2, Marketplace **3×2**. No cost, no prereq, no behavior, no registry entry → not placeable. FishingHut biome rule = Beach-only (`BuildingBiomeRules.cs:16`).

### 2.5 Construction (blueprint → building)
- Placed as a **blueprint** (`IsConstructed=false`, 50% alpha) via `BuildingManager.SpawnBlueprint` (`BuildingManager.cs:124`). Cost is spent at placement time; **refunded** if no valid site found (`BuildingGrowthSystem.cs:204,211,351`).
- Builders construct it: `Building.ConstructProgress += Time.deltaTime` each frame per builder; completes at `CONSTRUCT_DURATION = 8f` seconds of accumulated builder-time (`Worker.cs:135,1051-1055`). `MAX_BUILDER_SLOTS = 2` (`Building.cs:24`) — 2 builders ≈ halve wall-clock time. `MarkConstructed()` flips alpha to 1 and fires `OnConstructed` (`Building.cs:84`).

### 2.6 Building HP / damage / capture (`Building.cs`, `Combat/CombatStats.cs`)
- Damage constants: vs Worker 15, vs Soldier 15, vs Building **10**; attack interval 0.5s (`CombatStats.cs:19-23`).
- **TownHall is indestructible**: HP floored at 1; crossing HP≤50 (`TOWNHALL_CAPTURE_THRESHOLD_HP`) while an enemy is at war triggers `ConquestSystem.ForceStartDwell` (city capture), not destruction (`Building.cs:113-167`).
- Other buildings destroyed at HP≤0 (`Building.cs:136-141`).

---

## 3. PROFESSIONS ↔ ECONOMY

### 3.1 Enum (`Agents/Profession.cs:4`)
```csharp
public enum Profession { Civilian, Woodcutter, Farmer, Hunter, Builder, Sawyer, TreePlanter, StoneMiner, IronMiner, Trader, Soldier, General }
```

### 3.2 Gather / produce table (rates from `Agents/Worker.cs`)

| Profession | Works at / on | Output resource | Amount per cycle | Cycle timing | Search radius | Code |
|-----------|----------------|-----------------|------------------|--------------|---------------|------|
| **Woodcutter** | Tree node | Wood | `CHOP_AMOUNT=10` (capped by node's Remaining=10 → 10) | `CHOP_INTERVAL=5s` then carry to TownHall | 80 | `Worker.cs:116,658,699,722` |
| **StoneMiner** | StoneNode | Stone | 10 requested, node cap 5 → **5** | 5s + travel | 80 | `Worker.cs:670,722` |
| **IronMiner** | IronNode | Iron | 10 requested, node cap 5 → **5** | 5s + travel | 80 | `Worker.cs:682,722` |
| **Farmer** | Farm building | Food | `FOOD_PER_TEND=3` straight to stockpile | needs **≥2 farmers** (`MinFarmersToOperate`); `TEND_DURATION=10s` | n/a (assigned) | `FarmBehavior.cs:12,55,57-76` |
| **Hunter** | Fauna node (from Lodge) | Food | `HUNT_CHOP_AMOUNT=5` (node cap 5) | `HUNT_DURATION=5s` | 80 | `Worker.cs:130,831,876` |
| **Civilian** | Bush node (idle forage) | Food | `BUSH_CHOP_AMOUNT=3` (node cap 3) | `BUSH_HARVEST_DURATION=3s` | 60 | `Worker.cs:125,613,645` |
| **Sawyer** | Sawmill | **Planks** | consumes 1 Wood → yields **4 Planks** | `SAWYER_PROCESS_TIME=4s` | nearest sawmill | `Worker.cs:155,156,2175-2226` |
| **TreePlanter** | plants Tree nodes near Sawmill | (grows Wood supply) | 1 tree/plant | `PLANT_DURATION=4s`; `SAWMILL_PLANT_RANGE=30` | 30 | `Worker.cs:161,163,2325` |
| **Trader** | moves surplus city→city | any | up to `TRADER_CARRY_CAP=25` | continuous | n/a | `Worker.cs:172,1875,1907-1915` |
| **Builder** | blueprints | (constructs) | 8s builder-time each | — | city-scoped | `Worker.cs:988-1058` |
| **Soldier / General** | combat (Barracks) | — | — | — | — | `Worker.cs:1060`, `BarracksBehavior` |

Deposit flow (gatherers): pick node → reserve → walk (flow field) → `Chop` → `Carrying` slot set → walk to TownHall/city → `TickDepositing` adds to `CityStockpile()` (`Worker.cs:969-980`). Trader priority order: Planks, Food, Wood, Stone, Iron (`Worker.cs:176-179`); surplus threshold 20, demand threshold 5 (`:173-174`).

### 3.3 Worker → building assignment (`Economy/BuildingBehavior.cs`)
`BuildingBehavior.TryAssign(worker)` (`:42-56`): rejects if `CapacityFor(profession) ≤ 0`, if nation mismatch, if city mismatch, or if slot full. Per-building capacities:
- Farm → 3 Farmers (`FarmBehavior.cs:51`)
- Sawmill → 3 Sawyers (`SawmillBehavior.cs:37`)
- HuntersLodge → 3 Hunters (`HuntersLodgeBehavior.cs:48`)
- Barracks → 12 Soldiers+Generals combined (`BarracksBehavior.cs:49,80-90`)

### 3.4 Nation-level profession caps & demand (`Agents/ProfessionRegistry.cs`, `ProfessionCapacity.cs`)
`ProfessionCapacity.CapFor(nation, prof) = max(def.CapFn(nation), def.FloorCap)` (`ProfessionCapacity.cs:33-34`). Per-building constants (`ProfessionCapacity.cs:11-23`):
`WOODCUTTER_PER_TOWNHALL=4, SAWYER_PER_SAWMILL=5, TREE_PLANTER_PER_SAWMILL=1, FARMER_PER_FARM=2, HUNTER_PER_LODGE=3, STONE_MINER_PER_TOWNHALL=2, IRON_MINER_PER_TOWNHALL=2, TRADER_PER_NATION=8, BUILDER_PER_BLUEPRINT=2, SOLDIER_PER_BARRACKS=12`.

`ProfessionDef` fields (`ProfessionRegistry.cs:8-18`): `Id, Weight, DemandFn(nation,city), CapFn(nation), CapFnPerCity, DemandThreshold, FloorCap`.

| Profession | Weight | DemandThreshold | FloorCap | CapFn (nation) | Demand source |
|-----------|--------|-----------------|----------|----------------|---------------|
| Farmer | 1.5 | 30 | **2** | Farms × 2 | `FoodNeeded(city)` |
| Hunter | 1.4 | 30 | 0 | Lodges × 3 | `FoodNeeded(city)` |
| Sawyer | 1.2 | 20 | **2** | Sawmills × 5 | `PlanksNeeded(city)` if nation Wood ≥ 2 |
| TreePlanter | 1.1 | 20 | **2** | Sawmills × 1 | `TreesNeeded(city)` |
| Woodcutter | 0.8 | 60 | **2** | TownHalls × 4 | `WoodNeeded(city)` |
| StoneMiner | 1.2 | 40 | **2** | TownHalls × 2 | `StoneNeeded(city)` |
| IronMiner | 1.1 | 20 | **2** | TownHalls × 2 | `IronNeeded` w/ min reserve 5 |
| Trader | 0.6 | 1 | 0 | ≥2 cities → max(4, cities×4), else 0 | cities ≥ 2 |
| Builder | 1.0 | 1 | 0 | unconstructed × 2 | unconstructed > 0 |
| Soldier | 1.8 | 10 | 0 | Barracks × 12 | `SoldiersNeeded(city)` |
| Civilian | — | — | — | `int.MaxValue` (`ProfessionCapacity.cs:28`) | default/unemployed forager |
| General | — | — | — | 0 (promoted, not assigned) | — |

`CurrentCount` only counts **adult** workers (`ProfessionCapacity.cs:44-49`). Generals count toward Soldier count.

---

## 4. PRODUCTION & CONSUMPTION CHAINS

### 4.1 Processing chain (the only refinement step)
```
Tree node --Woodcutter(chop 10)--> Wood (city stockpile)
Wood --Sawyer: TrySpend 1 Wood @TownHall, walk to Sawmill, process 4s--> 4 Planks --> city stockpile
Planks (+Stone +Iron) --BuildingGrowthSystem--> buildings
```
Ratio: **1 Wood → 4 Planks** (`Worker.cs:155-156`). Demand math agrees: `WOOD_PER_PLANK = 0.25` (`DemandQueries.cs:15`). Food and Stone/Iron are terminal (no processing).

### 4.2 Food consumption — `Economy/HungerSystem.cs`
- Round-robin over workers, 30/frame (`WORKERS_PER_FRAME`).
- Each adult drains hunger every `HUNGER_DRAIN_INTERVAL = 7s` (`:12`).
- When `Hunger ≥ HUNGER_HUNGRY_THRESHOLD (30)` and adult → `TryFeedWorker` (`:16,55`).
- `TryFeedWorker`: spend **1 Food** from `AssignedCity ?? CapitalCity` stockpile; if empty, scan other nation cities; on success `Hunger = 0` (`:62-81`).
- `IsHungry` flag when `Hunger > 70` (`Worker.cs:199`).

### 4.3 Build-cost consumption — `BuildingGrowthSystem.cs`
- On placement: `TrySpendCity` verifies then `SpendAcrossCities` (primary city, then siblings; war-chest can `ForceSpend` into negative) (`:283-333`).
- Only Planks/Stone/Iron are ever spent on buildings (Wood/Food/Gold never directly).

### 4.4 Demand queries (drive both worker mix and building placement) — `Nations/DemandQueries.cs`
Key constants (`:10-24`): `FOOD_RESERVE_MIN=20, WOOD_PER_TILE_EXPANSION=5, PLANKS_PER_HOUSE=5, STONE_PER_HOUSE=3, WOOD_PER_PLANK=0.25, IRON_PER_SAWMILL=3, IRON_PER_HUNTERS_LODGE=3, IRON_RESERVE_MIN=20, WOOD_RESERVE_MIN=50, FRONTIER_SOFT_CAP=250`. Tree upkeep: `TREE_TARGET_COUNT=30` within `TREE_SEARCH_RADIUS=25` of TownHall (`:207-208`).
- `FoodNeeded(city)` = `max(20, members/2) − stock.Food` (`:151-156`).
- `PlanksNeeded(city)` = `max(housing plank demand, sum of unbuilt blueprints' plank cost) − stock.Planks` (`:111-126`).
- `StoneNeeded`, `IronNeeded` similar; Iron adds `IRON_RESERVE_MIN=20` floor when a TownHall exists (`:135-149`).

---

## 5. SETTLEMENTS / CAPITAL

### 5.1 City data model (`Nations/City.cs`)
`City { int Id; int NationId; Vector2Int Center; HashSet<Vector2Int> Territory; List<Building> Buildings; List<Agent> Members; ResourceStockpile Stockpile; FlowField CapitalFlowField; }`. `Population => Members.Count` (`:36`). Each city owns its own stockpile.

### 5.2 Capital creation (`Nations/NationManager.cs:171-194`)
- `CreateCapitalCity`: builds a circular territory of `radius` (passed by caller), inserts city at index 0 of nation's list, owns tiles.
- **Capital city stockpile starts EMPTY** — no starting resources are seeded (only secondary cities get a Wood seed, see below). Nation must bootstrap its economy from map nodes.

### 5.3 Initial buildings per nation (`Economy/InitialBuildingPlacer.cs:13-69`)
On world start, each nation places (in order): **Castle** (anchor = capital − (1,1)), **TownHall**, **Farm**, **Sawmill** (chosen by tree proximity: samples 20 valid sites, picks most trees within radius 6), **Barracks**. Placement uses 500 random attempts in territory, then a spiral fallback (max ring 240).

### 5.4 Founding new cities (`Nations/CityFoundingSystem.cs`)
Constants (`:15-33`): `TICK_INTERVAL=5s, POP_THRESHOLD=60, MIN_CITY_DISTANCE=30, MIN_CROSS_NATION_DISTANCE=50, MAX_CITY_DISTANCE=80, FOUND_COOLDOWN=180s, CITY_RADIUS=5, SEARCH_ATTEMPTS=200, VIABILITY_RADIUS=2, VIABILITY_MIN_TILES=18, FOUNDING_ADULTS=10, HOMELESS_PRESSURE_THRESHOLD=8, SETTLED_MIN_ADULTS=20`.
On found (`:156-194`): create city (radius 5) → place TownHall + Barracks (`InitialBuildingPlacer.PlaceTownHallInCity`) → spawn a **Farm blueprint** → **seed 100 Wood** (`newCity.Stockpile.Add(ResourceType.Wood, 100)`) → spawn 10 founding adults (5M/5F).

### 5.5 Building growth loop (how a settlement places buildings) — `Economy/BuildingGrowthSystem.cs`
Every `TICK_INTERVAL=5s`, for each nation, cities sorted by **fewest buildings first** (`InfraComparer`), up to `PLACEMENTS_PER_TICK_CAP=12` per city per tick (`:13-16,59-75`).
`TryPlaceBest` priority ladder (`:91-141`):
1. **Capital + at war + 0 Barracks** → Barracks (bypasses affordability if `HasActiveWarChest`).
2. Food **Critical** → Farm, else HuntersLodge (`TryPlaceFoodInfra`).
3. Food **Insufficient** → same.
4. Housing **Critical/Insufficient** → random of {House, HouseMedium, HouseLarge}.
5. **At war** → Barracks.
6. → Sawmill.
7. **InfraTier ≥ 1** → HuntersLodge.
8. Housing **not Surplus** → random house.
`TryPlaceType` gate (`:176-187`): registered ∧ `BuildableByGrowthSystem` ∧ count < `PerCityCap` ∧ prereq met ∧ affordable ∧ `DemandFn(city) > 0`.
Site search: 300 random tries in city territory, must be walkable, reachable via city flow field, and pass `CanPlace` (`:359-378`).

### 5.6 City state evaluation (`Nations/CityStateEvaluator.cs`, `CityState.cs`)
`SecurityLevel { Critical, Insufficient, Adequate, Surplus }`.
- Food per-member thresholds: `<0.3 Critical, <0.7 Insufficient, <1.5 Adequate, else Surplus` (`:10-12,40-48`).
- Housing: based on unmarried adults vs available couple-slots×2 (`:50-95`).
- `InfraTier`: `<2` food buildings → 0; no sawmill → 1; else 2 (`:97-110`).
- `ResourceDeficit_*` flags when Plank/Stone/Iron `< 5` (`RESOURCE_DEFICIT_THRESHOLD`).

### 5.7 Placement legality (`Economy/BuildingManager.cs:532-599` `CanPlace`)
For every footprint cell: in-bounds ∧ walkable ∧ within nation territory ∧ allowed biome (grassland default; Lodge grassland+beach; beach allowed only if ≥3 tiles from Ocean/DeepOcean) ∧ cell unoccupied. Plus a **separation** check: Chebyshev distance to any existing nation building footprint must exceed `minBuildingSeparation=1` (Barracks uses separation **0**) (`:17,578`). Occupancy tracked in `_occupiedCells` dictionary (`:19`). Castle placement can demolish adjacent non-essential (houses only) buildings (`:334-412`).

---

## 6. CATALOG / DATA FORMAT (for the replica to mirror)

The economy is **code-as-data**, not asset-as-data. To mirror faithfully, replicate these static registries as data tables:

| Catalog | Type | Location | Contents |
|---------|------|----------|----------|
| `BuildingRegistry` | static `Dictionary<BuildingType,BuildingDef>` in a static ctor | `Economy/BuildingRegistry.cs:28-126` | costs, prereqs, caps, demand fns, behavior type — **the master building data** |
| `ProfessionRegistry` | static `Dictionary<Profession,ProfessionDef>` | `Agents/ProfessionRegistry.cs:27-102` | weights, demand fns, cap fns, floor caps |
| `BuildingFootprintCatalog` | static `switch` → `(w,h)` | `Economy/BuildingFootprintCatalog.cs` | footprint per type |
| `BuildingBiomeRules` | static arrays + `switch` | `Economy/BuildingBiomeRules.cs` | allowed biomes per type |
| `ProfessionCapacity` | `const int` per-building rates | `Agents/ProfessionCapacity.cs:11-23` | worker-per-building constants |
| `CombatStats` | `const int` | `Economy/Combat/CombatStats.cs` | building HP, damage, barracks capacity |

The only Unity ScriptableObject `.asset` catalogs (`Assets/Catalogs/`) are **sprite lookups**, purely cosmetic:
- `BuildingSpriteCatalog.asset` — one `Sprite` (GUID ref) per BuildingType; class `Economy/BuildingSpriteCatalog.cs`, `GetSprite(type)` switch. Fields: Castle, TownHall, Farm, Sawmill, HuntersLodge, FishingHut, Smithy, Bakery, Brewery, Forge, Marketplace, Barracks, House, HouseMedium, HouseLarge.
- `ProfessionSpriteCatalog.asset`, `ProtagonistSpriteCatalog.asset` — visual only.

A faithful replica needs **none** of the `.asset` files for simulation; it needs the six code tables above reproduced with the exact numbers in this document.

---

## 7. KEY CONSTANTS QUICK-REFERENCE

- Node capacities: Tree 10, Bush 3, Stone 5, Iron 5, Fauna 5.
- Gather amounts: chop 10 (wood), stone/iron 5 (node-capped), bush 3, hunt 5, farm tend 3.
- Sawmill: 1 Wood → 4 Planks, 4s.
- Timings: chop interval 5s, tend 10s, hunt 5s, bush 3s, sawyer 4s, plant 4s, construct 8s (builder-time), attack 0.5s.
- Building HP: Castle 500, TownHall 300, Barracks 200, all others 100.
- Damage: vs building 10, vs unit 15.
- Barracks: 12 soldiers. Farm/Sawmill/Lodge: 3 workers each. Farm needs ≥2 to operate.
- Hunger: drain every 7s, eat at ≥30, hungry-flag at >70, cost 1 Food.
- Growth tick 5s, 12 placements/city/tick, city sort = fewest buildings first.
- City radius 5 (secondary); founding adults 10; secondary-city Wood seed 100; capital seed 0.
- Stockpile floor −999 (ForceSpend only).
