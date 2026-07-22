# agents.exe — Nations / Protagonists / Strategic Decisions / Mock Heuristic — Faithful-Replica Spec

Source root: `unity-project/Assets/Scripts/`. All paths below are relative to it unless noted.
This subsystem is the **strategic brain**: each Nation has a Leader (a protagonist Worker) whose
strategic **Intent** is chosen every scheduler tick, dispatched into concrete world changes, and fed
back through events. **This replica has NO LLM** — the active decision source defaults to `Mock`
(`ProtagonistDecisionSystem.defaultSource = DecisionSource.Mock`, `ProtagonistDecisionSystem.cs:14`),
so **`MockHeuristic.Decide` IS the entire AI**. Reproduce it verbatim.

---

## 0. Decision data-flow (who calls the brain)

```
DecisionScheduler (MonoBehaviour, Update loop)
  → picks a Nation whose tick is due (or an event wakeup)
  → NationStateSerializer.Serialize(nation, tick, playerMsg)  → NationStateDoc  (the "world snapshot")
  → ProtagonistDecisionSystem.GetIntent(nation, state, playerMsg)
        if ActiveSource == Mock:  MockHeuristic.Decide(nation, state, playerMsg)   ← THE AI
        if ActiveSource == LLM:   LLMClient.DecideAsync(...) then fallback to Mock on failure
  → IntentDispatcher.Apply(nation, intent, source)  → mutates world, returns IntentResult
  → appends a "decision" RecentEvent to the nation
```

Key constants (`DecisionScheduler.cs`):
- `tickInterval` = **45s** default, clamped [30,300] (`:13-19`). Each nation decides every ~45s.
- Initial per-nation stagger: `nextTickAt = now + nationId*tickInterval/numNations` (`:186`).
- `DECISION_COOLDOWN_REAL` = 15s (only gates the LLM path; Mock has no cooldown, `CanProcessNow` `:338-350`).
- `MAX_DRAIN_PER_FRAME` = 2 decisions/frame; one decision per nation per frame (`:263-291`).
- `FOOD_CRASH_COOLDOWN` = 60s (`:23`).
- Event **wakeups** (immediate decisions, bypass the tick clock) are enqueued for: `war_declared`,
  `war_attacked` (`HandleWarDeclared` `:66-71`), `leader_succession` (`:73-77`), `food_crash`
  (`CheckFoodCrash` `:201-221`), `army_decimated` (`current < baseline*0.5`, `CheckArmyDecimation` `:223-241`),
  `building_destroyed`/`city_lost` (Castle/TownHall/Barracks destroyed, `OnBuildingDestroyed` `:129-161`),
  `consort_died` (`NotifyProtagonistDied` `:163-173`), `royal_birth`, `peace_signed`, `city_founded`,
  `heir_without_fallback`. Dead-leader nations are skipped (`ProcessRequest` `:304`).

`ProtagonistDecisionSystem` also drains an out-of-band `PlayerMessageQueue`/`GameSessionState.PendingDecisionSource`
(`:21-22`, `:39-44`) — for the replica, `PendingDecisionSource` will be None→falls back to `defaultSource` (Mock).

---

## 1. NATION MODEL — `Nations/Nation.cs`

`public class Nation` (plain C# object, not a MonoBehaviour):

| Field | Type | Notes | Line |
|---|---|---|---|
| `Id` | int (get) | index in `NationManager._nations`; assigned `= _nations.Count` | :12, NationManager:163 |
| `Name` | string (get) | e.g. `"Nation 1"` | :13 |
| `Color` | UnityEngine.Color (get) | from `NationPalette.GetColor(i)` | :14 |
| `Capital` | Vector2Int (get) | capital tile coord (immutable) | :15 |
| `Cities` | List<City> | `Cities[0]` is always the capital city (`Insert(0,...)`) | :17 |
| `CapitalCity` | City (computed) | `Cities.Count>0 ? Cities[0] : null` | :18 |
| `Territory` | HashSet<Vector2Int> | owned tiles | :20 |
| `Members` | List<Agent> | all living+dead agents that belong to the nation | :22 |
| `Stockpile` | ResourceStockpile | nation-level fallback stockpile (cities have their own) | :23 |
| `Buildings` | List<Building> | | :25 |
| `DesiredSoldiers` | int (set) | army target; drives conscription/mobilization | :27 |
| `WarsWon` / `WarsLost` | int (set) | | :29-30 |
| `LeaderPromotionTime` | float (set) | `Time.realtimeSinceStartup` when current leader promoted | :32 |
| `RoyalFamily` | List<Worker> | protagonists (Leader/Consort/Royal) | :34 |
| `Leader` | Worker (computed) | `RoyalFamily.First(w.IsAlive && Role==Leader)` | :36 |
| `Consort` | Worker (computed) | `Role==Consort` | :37 |
| `Heir` | Worker (computed) | `w.IsAlive && w.IsHeir` | :38 |
| `LastLeaderContextSummary` | string | epitaph of previous leader (fed to next as `PreviousLeaderSummary`) | :40 |
| `LastLeaderIntent` / `LastLeaderReason` | string | last **applied** intent + reason | :42-43 |
| `RecentEvents` | RecentEventsBuffer | ring buffer of world events (the heuristic reads this) | :45 |
| `CurrentAttackOrder` | AttackOrder | set by AttackCity intent; drives soldiers | :55 |
| `HasActiveWarChest` | bool | set true on mobilization | :57 |
| `IsDissolved` | bool | | :59 |
| `CitiesFoundedCount` | int | | :61 |
| `FocusedResource` | ResourceType? | set by FocusResource intent | :63 |
| `PrioritizedBuildingType` | BuildingType? | set by PrioritizeBuilding intent | :65 |

Methods: `AddTile`/`RemoveTile`/`Contains` (territory helpers, :67-71).

### `ResourceType` enum — `Nations/ResourceStockpile.cs:3`
`{ Food, Wood, Stone, Iron, Gold, Planks }` (ordinal order matters).
`ResourceStockpile` fields: `Food, Wood, Stone, Iron, Gold, Planks` (all int). `STOCKPILE_MIN = -999`
(ForceSpend can go negative to -999; `Add` ignores amount<=0; `TrySpend` fails if insufficient). :1-90

### `BuildingType` enum — `Economy/BuildingType.cs:4`
`{ Castle, TownHall, Farm, Sawmill, HuntersLodge, FishingHut, Smithy, Bakery, Brewery, Forge, Marketplace, Barracks, House, HouseMedium, HouseLarge }`

### `City` — `Nations/City.cs`
`Id` (get), `NationId` (private set; `SetNationId`), `Center` (Vector2Int), `Territory` (HashSet),
`Buildings`, `Members` (List<Agent>), `Stockpile` (own ResourceStockpile), `CapitalFlowField`.
`Population => Members.Count` (:38). `FootprintCount` sums building footprints. Capital = `Cities[0]`.

### `NationManager` — `Nations/NationManager.cs` (world setup + tile ownership)
- `numNations` = **4** default (:15), `minCapitalDistance` = 100 (:17), `initialRadius` = 10 (:19),
  `capitalSeed` = 7 (:20).
- Capitals chosen from walkable pool (shuffled by `System.Random(capitalSeed)`, capped 2000), spaced
  ≥`minCapitalDistance` (Manhattan), each needing `CountReachableArea ≥ MIN_REACHABLE_AREA(400)` within
  `LEBENSRAUM_RADIUS(20)` and reachable from capital 0 (`PickCapitals` :87-159).
- `CreateCapitalCity`: circular claim radius `initialRadius` (dx²+dy²≤r²), owns tiles, `Cities.Insert(0,...)`.
- `FoundCity(nation, center, radius)`: appends a satellite city, claims a circle, fires `OnCityFounded`.
- `OwnTile(x,y,nationId)`: single source of truth for `_tileNation[,]`; -1 = unowned; maintains
  `Territory` sets; fires `OnTerritoryChanged`.
- `RemoveCity`: releases tiles; if capital removed, destroys its Castle and rebuilds one at new `Cities[0]`.

---

## 2. PROTAGONIST MODEL — `Agents/Worker.cs` (extends `Agents/Agent.cs`)

A **protagonist is just a `Worker` with `Role != null`**. There is no separate class.

### Base `Agent` (`Agents/Agent.cs`)
- `ADULT_AGE_SECONDS` = **60f** (:9). `Stage => Age>=60 ? Adult : Child` (:19).
- `Hp` int (default 100), `NationId` int (default -1), `LifetimeSeconds` float (**default 180f**, :15),
  `Gender` (default Male), `Age => Time.time - _spawnTime` (:60).
- `IsAlive => Hp>0 && Age < LifetimeSeconds` (:62); `IsDead => !IsAlive`.
- `Killer` (Agent, set by `TakeDamage(amount, attacker)` when hp hits 0, :67-76).
- `OnDied` event fired once via `FireDied()` (:78-86). `ForceAge(sec)` shifts spawn time.

### `Worker` protagonist-relevant fields (`Agents/Worker.cs:80-97`)
| Field | Type | Meaning |
|---|---|---|
| `Role` | `ProtagonistRole?` | null = ordinary worker; else protagonist |
| `Personality` | `Personality` | drives the heuristic (see §2.1) |
| `Father` / `Mother` | Worker | royal lineage pointers (separate from `Family.Father/Mother`) |
| `Children` | List<Worker> | royal children |
| `IsHeir` | bool | designated successor |
| `IsProtagonist` | `=> Role.HasValue` | |
| `GetLifetimeMultiplier()` | `=> IsProtagonist ? 2.5f : 1f` | protagonists live **×2.5** (180→450s) |
| `DemoteCountdownStart`, `LastDemoteTime` | float | demotion bookkeeping |
| `Profession` | `Profession` | civilian job / Soldier / General |
| `Family` | `FamilyComponent` | spouse/children/house/pregnancy (see below) |
| `Hunger` | int | 0..100; `IsHungry => Hunger>70`; 100 = death by starvation |

`ProtagonistRole` enum (`Agents/ProtagonistRole.cs`): `{ Leader, Consort, Royal }`.
`ResetForReuse` (pool recycle) clears Role, IsHeir, Father, Mother, Children, Personality=default, etc. (:299-313).

Worker's `Tick()` is the **operational** worker/soldier FSM (chopping, mining, combat, etc.) — 30+
`WorkerState`s (:11-36). Not decision-relevant except: `IsMilitary => Profession∈{Soldier,General}`
(:185), `IsGeneral` (:183), soldiers obey `_directedAttackTarget` / nation `CurrentAttackOrder`, and
march at `MARCH_WAR_SPEED_MULTIPLIER=1.3×` while their nation is at war (:1270, :1364-1370).

### `FamilyComponent` (`Agents/FamilyComponent.cs`)
`Gender, Spouse, Mother, Father, Children(List), AssignedHouse, IsPregnant, PregnancyEndTime,
NextProfessionEvalTime`. `IsAncestor(a,b)` BFS up to depth 6 (incest guard). Cleared on death/reuse.

### 2.1 `Personality` enum — `Agents/Personality.cs`
`{ Aggressive, Cautious, Expansionist, Mercantile, Pious, Cunning, Stoic }` (7 values, ordinal order).
**Parsing:** `MockHeuristic.ParsePersonality(s)` → `Enum.TryParse(...,ignoreCase)`; **null/unknown ⇒ Stoic**
(`MockHeuristic.cs:1001-1005`). `RandomPersonality()` = uniform over all 7 (`SuccessionSystem.cs:453-457`).

How personality affects decisions: it is the primary switch inside the Mock heuristic's fallback
(`DefaultByPersonality`), the proactive-war chance table, the surrender/disband rolls, the
player-message obedience roll (`RollObey`), and the flavor text (`REASON_TEMPLATES`). All exact numbers in §4.

---

## 3. INTENTION VOCABULARY

### 3.1 `IntentType` enum — `LLM/IntentType.cs` (28 values, declaration order = ordinal 0..27)
Core (0–11): `DeclareWar, MakePeace, BuildArmy, DisbandArmy, FoundCity, Migrate, ExpandTerritory,
NameHeir, Disinherit, FocusResource, PrioritizeBuilding, Idle`
Reserved-Ready (12–17): `ProposeTrade, Festival, ArrangeMarriage, AttackCity, Retreat, Conscript`
Reserved-Future (18–27): `FormAlliance, BreakAlliance, SendGift, DemandTribute, Surrender, AbandonCity,
RelocateCapital, Fortify, Mourn, Observe`

### 3.2 `Intent` payload — `LLM/Intent.cs` (all fields nullable/optional)
`IntentType Type;` plus optional: `int? TargetNationId, TargetCityId, FromCityId, ToCityId, RoyalId,
WorkerId, DeceasedRoyalId, Count, TargetSize, Amount;` `Direction? Direction; DistanceHint? DistanceHint;
ResourceType? Resource; BuildingType? BuildingType;` `string Reason; bool RespondedToPlayer;`
`string MockRulePath;` (which heuristic rule fired — see §4) `ResourceBundle Offer, Request;`
- `Direction` enum (`LLM/Direction.cs`): `{ N, S, E, W, NE, NW, SE, SW }`.
- `DistanceHint` enum (`LLM/DistanceHint.cs`): `{ Near, Medium, Far }`.
- `ResourceBundle` (`LLM/ResourceBundle.cs`): `{ ResourceType Resource; int Amount; }`.

### 3.3 JSON contract (for LLM path / interop) — `shared/schemas/intent.schema.json`, `shared/grammars/intent.gbnf`
Discriminated union on `"type"`; JSON keys are **snake_case** (`target_nation_id`, `target_size`,
`from_city_id`, `to_city_id`, `royal_id`, `worker_id`, `deceased_royal_id`, `building_type`,
`distance_hint`, `responded_to_player`, `offer`/`request` = `{resource,amount}`). `reason` ≤200 chars.
Required fields per type match the intent's mandatory params (e.g. DeclareWar requires `target_nation_id`;
FoundCity requires `direction`+`distance_hint`; Migrate requires `from_city_id`+`to_city_id`+`count`).
Schema `resource` enum = `[Food,Wood,Stone,Iron,Gold,Planks]`; `building_type` enum = the 15 BuildingTypes.
For the replica (no LLM) this contract is informational; the Mock builds `Intent` objects directly.

### 3.4 DISPATCH — `LLM/IntentDispatcher.cs` (`Apply(nation, intent, source) → IntentResult`)
Pipeline: `ValidateSchema` (null/undefined type ⇒ Malformed) → `ValidateAuth` (null nation ⇒ Rejected;
**no living leader ⇒ Rejected**, `:87-94`) → big `switch` on `intent.Type`. On `Applied` with a live
leader, writes `nation.LastLeaderIntent/Reason` (:58-62). **Always** appends a `"decision"` RecentEvent
with args `{intent, reason, source, status, result_reason}` (:64-73). Then `IntentLogger.Record` +
fires `OnIntentDispatched` (:329-334).

`IntentResult` = `{ IntentStatus Status; string Reason; object SideEffects; }`.
`IntentStatus` enum (`LLM/IntentStatus.cs`): `{ Applied, Rejected, Deferred, NotImplemented, Malformed }`.
`IntentSource` enum (`LLM/IntentSource.cs`): `{ Mock, LLM, Player }`.

| Intent | Handler behavior (file:line) |
|---|---|
| **DeclareWar** | require `TargetNationId`; target exists, ≠ self, not already at war; `WarRegistry.DeclareWar`; then `MilitaryMobilization.ArmDefensively(both)`. :96-111 |
| **MakePeace** | require target, must currently be at war; `WarRegistry.MakePeace`; `Demobilize(both)`. :113-128 |
| **BuildArmy** | `target = min(TargetSize ?? 8, MAX_DESIRED=60)`; `DesiredSoldiers = max(DesiredSoldiers, target)`; `EnsureBarracksCapacity`. :130-140 |
| **DisbandArmy** | `DesiredSoldiers = 0`. :235-239 |
| **FoundCity** | `CityFoundingSystem.TryFoundForIntent(nation, Direction)`; Deferred if no viable site. :241-250 |
| **Migrate** | `CityMigrationSystem.RequestMigrationBias(nation, Direction)` (sets a bias; population equalizer acts later). :252-259 |
| **ExpandTerritory** | `TerritoryExpansionSystem.RequestExpansionBias(nation, Direction)`. :261-268 |
| **NameHeir** | require `RoyalId`; royal must be in `RoyalFamily`, `Role==Royal` (not Leader/Consort), alive; clears all `IsHeir`, sets chosen. :270-286 |
| **Disinherit** | require `RoyalId`; set that royal `IsHeir=false`. :288-297 |
| **FocusResource** | require `Resource`; `nation.FocusedResource = Resource`. :299-305 |
| **PrioritizeBuilding** | require `BuildingType`; `nation.PrioritizedBuildingType = BuildingType`. :307-313 |
| **Idle** | no-op, Applied. :315 |
| **AttackCity** | require `TargetCityId`; find city across all nations; not own; must be at war with its nation; sets `nation.CurrentAttackOrder = {city,nation,center,IssuedAt,source}`. :142-182 |
| **Retreat** | `CurrentAttackOrder=null`; clear `_directedAttackTarget` on all military members. :184-194 |
| **Conscript** | `amount = Count ?? Amount ?? 4`; converts up to `amount` **Adult, Civilian, Male** members → Soldier, assigns to a Barracks; Deferred if none eligible. :196-233 |
| **ProposeTrade / Festival / ArrangeMarriage** | `Stub` ⇒ NotImplemented ("Reserved-Ready ... pending"). :38-40 |
| **FormAlliance / BreakAlliance / SendGift / DemandTribute / Surrender / AbandonCity / RelocateCapital / Fortify / Mourn / Observe** | `Stub` ⇒ NotImplemented ("Reserved-Future ..."). :45-54 |

**IMPORTANT replica note:** the Mock heuristic frequently *emits* `Surrender`, `Retreat`, `Observe`,
`AttackCity` etc., but **Surrender/Observe are stubs** (no effect beyond the logged decision event).
`Retreat`, `AttackCity`, `Conscript` are fully wired. So effectively the Mock's *actionable* outputs are:
DeclareWar, MakePeace, BuildArmy, DisbandArmy, FoundCity, Migrate, ExpandTerritory, NameHeir, Disinherit,
FocusResource, PrioritizeBuilding, Idle, AttackCity, Retreat, Conscript.

#### Dispatch-target details (how intents become world actions)
- **FoundCity** (`CityFoundingSystem.TryFoundForIntent` :201-295): up to `SEARCH_ATTEMPTS=200` random
  tiles; must be walkable Grassland/Beach; first 70% of attempts must match the requested `Direction`
  (N⇒y>capY, etc., :219-231); distance to own cities ∈[`MIN_CITY_DISTANCE=30`,`MAX_CITY_DISTANCE=80`]
  (Manhattan); ≥`MIN_CROSS_NATION_DISTANCE=50` from other nations' cities and not on foreign-owned tile;
  `IsSpawnZoneViable` (≥`VIABILITY_MIN_TILES=18` walkable within radius 2, ≥12 grassland, :393-406) and
  reachable from capital. On success: `NationManager.FoundCity(radius=CITY_RADIUS=5)`, emits
  `city_founded`, `CitiesFoundedCount++`, places TownHall + a Farm blueprint. (Autonomous founding also
  runs on a timer at `POP_THRESHOLD=60`, cooldown `FOUND_COOLDOWN_SECONDS=180`.)
- **Migrate** (`CityMigrationSystem` :35-89): `RequestMigrationBias` just stores a Direction. A periodic
  equalizer (`TICK_INTERVAL=3s`, `MIGRATIONS_PER_TICK=5`, `POP_DELTA_THRESHOLD=5`) moves members from
  cities above `totalPop/cityCount` to cities below; the bias re-sorts sinks so bias-direction cities
  fill first (`DirectionMatch`, :206-215), then the bias is consumed. Needs ≥2 cities.
- **ExpandTerritory** (`TerritoryExpansionSystem` :47-89): `RequestExpansionBias` stores a Direction;
  a tick (`TILES_PER_TICK=5`, wood-costed) claims frontier tiles, preferring bias-direction tiles.

---

## 4. THE MOCK HEURISTIC — `LLM/MockHeuristic.cs` (THE AI) — reproduce verbatim

`public static Intent Decide(Nation nation, NationStateDoc state, string playerMessage=null)` (`:38`).

**Determinism / RNG:** `seed = GetWorldSeed() ^ nation.Id ^ state.World.Tick`; `rng = new System.Random(seed)`
(`:41-42`). `GetWorldSeed()` reads `WorldBootstrap.GetParams().Seed` (cached; 0 if none, :19-27). **Use
.NET `System.Random` semantics** (`rng.Next(n)`=[0,n), `rng.NextDouble()`=[0,1)) to match sequences.

Every returned Intent is stamped with `MockRulePath` (the rule id). Coverage mode
(`CoverageMode`, default **false**) round-robins the 18-item `COVERAGE_POOL` for QA — **ignore for the
replica** (it's off in the shipped game).

### 4.0 The rule cascade (top-level `Decide`, `:44-81`) — FIRST MATCH WINS

```
Decide(nation, state, playerMessage):
    seed = worldSeed ^ nation.Id ^ state.World.Tick
    rng  = Random(seed)

    if CoverageMode and coverage slot open:  return BuildCoverageIntent(...)   # QA only, off by default

    # R0 — stalemate auto-peace
    r = TryStalemateMakePeace(nation, state);   if r: r.path="R0.stalemate_peace"; return r
    # R1 — crisis response (reads RecentEvents)
    r = TryCrisisResponse(nation, state, rng);  if r: r.path="R1.crisis"; return r
    # R2 — player message (natural-language command)
    if playerMessage nonempty:
        r = TryHandlePlayerMessage(nation, state, playerMessage, rng); if r: r.path="R2.player_msg"; return r
    # R3 — reactive war (only if currently at war)
    if state.Diplomacy.AtWarWith.Count > 0:
        r = TryWarIntent(nation, state, rng);    if r: r.path="R3.war"; return r
    # R3.5 — proactive war (start a war when at peace)
    r = TryProactiveWarIntent(nation, state, rng); if r: r.path="R3.5.proactive_war"; return r
    # R4 — succession (name an heir if none)
    r = TrySuccessionIntent(nation, state, rng); if r: r.path="R4.succession"; return r
    # R5 — personality default (peacetime economy/expansion)
    r = DefaultByPersonality(nation, state, rng); r.path="R5.default"; return r
```

### 4.1 R0 — `TryStalemateMakePeace` (`:84-107`)
```
if AtWarWith.Count == 0: return null
if Military.CasualtiesLast60Ticks >= 8: return null      # still actively bleeding → keep fighting
enemyId = AtWarWith[0]
enemy   = NationManager.GetNation(enemyId); if null: return null
warAge  = WarRegistry.GetWarAge(nation, enemy)           # real seconds since war start
if warAge <= 300f: return null
return MakePeace(target=enemyId, reason="This war drags on without progress. Better to seek peace.")
```

### 4.2 R1 — `TryCrisisResponse` (`:109-279`) — event-driven reflexes
```
if RecentEvents empty: return null
tick = state.World.Tick
personality = ParsePersonality(state.Leader?.Personality)

# mourning a predecessor: new leader with no LastIntent but a PreviousLeaderSummary
if Leader != null and Leader.LastIntent is empty and Leader.PreviousLeaderSummary nonempty:
    return Idle(reason="Mourning my predecessor. There is much to be done.")

for ev in RecentEvents:                 # newest-first not guaranteed; scans all
    if tick - ev.Tick > 30: continue    # only react to events within last 30 ticks
    switch ev.Type:
      "war_declared":
          if ev.arg "against" != nation.Id: break
          enemyEst = EstimateEnemySoldiers(state, rng)
          target   = max(Military.SoldiersCount + 8, enemyEst)
          return BuildArmy(TargetSize=target, reason=GetReason(BuildArmy,personality,rng))
      "combat" | "army_decimated":
          casualties = ev.arg "our_casualties" (0 if absent)
          cur = Military.SoldiersCount;  baseline = cur + casualties
          decimated = (ev.Type=="army_decimated") or (baseline>0 and casualties/baseline > 0.5)
          if decimated:
              target = (ev.Type=="army_decimated") ? 20 : max(cur*1.5, cur+4)
              return BuildArmy(TargetSize=target,
                       reason = decimated-army? "Our army was decimated. Rebuild." : GetReason(BuildArmy,...))
          break
      "building_destroyed":
          btype = ev.arg "building_type";  cityId = ev.arg "city_id"
          if btype in {"Castle","TownHall"}: return Idle(reason="Regrouping after the loss.")
          if Enum.TryParse<BuildingType>(btype): 
              return PrioritizeBuilding(BuildingType=btype, TargetCityId=(cityId>0?cityId:CapitalId),
                                        reason="We must rebuild what was lost.")
          break
      "city_lost":
          cityId = ev.arg "city_id"
          return PrioritizeBuilding(BuildingType=Barracks, TargetCityId=(cityId>0?cityId:CapitalId),
                                    reason=GetReason(PrioritizeBuilding,personality,rng))
      "food_crash":
          cityId = ev.arg "city_id";  targetCity = cityId>0?cityId:CapitalId
          if rng.Next(2)==0: return PrioritizeBuilding(Farm, targetCity, "Famine threatens the people.")
          else:              return FocusResource(Food, targetCity, "Food stores are dangerously low.")
      "city_founded": break        # no reflex
      "first_contact":
          if rng.Next(2)==0: return Observe(reason="Observing our new neighbor.")
          else:              return Idle(reason="Observing our new neighbor.")
      "royal_death":
          role = ev.arg "role";  if role=="Leader": break       # leader death handled by succession
          return Idle(reason="Grieving for one of our own.")
      "royal_birth": break
return null
```
Helpers used: `EstimateEnemySoldiers` (`:1061-1073`): over neighbors with `DiplomaticState=="at_war"`,
`max over them of (ownSoldiers * neighbor.RelativeStrength)`. `CapitalId` (`:1007-1012`): the city with
`IsCapital`, else `Cities[0].Id`, else 0.

### 4.3 R2 — `TryHandlePlayerMessage` (`:281-385`) — natural-language command parsing
Lowercases the message; keyword-matches (`ContainsAny`, ordinal substring). Then rolls **obedience**;
if the roll fails the command is **ignored** (returns null, cascade continues).
```
low = msg.ToLowerInvariant();  personality = ParsePersonality(...)
isWarMsg = false;  matched = null
if low contains "attack " or "war on ":
    isWarMsg = true; target = FindNeighborByName(state, low)   # substring match on neighbor.Name
    if target: matched = DeclareWar(target.NationId, GetReason(DeclareWar,...), RespondedToPlayer=true)
elif low contains "peace with" or "make peace":
    target = FindNeighborAtWar(state);  matched = MakePeace(target?.NationId, GetReason(MakePeace,...))
elif low contains "defend" or "army" or "recruit":
    matched = MakeBuildArmy(...)          # TargetSize = max(cur+8, EstimateEnemySoldiers)
elif low contains "found city" or "settle" or "new city":
    matched = MakeFoundCity(...)
elif low contains "migrate" or "move people":
    matched = MakeMigrate(...)
elif low contains "name heir" or "appoint ":
    matched = MakeNameHeir(state)         # may be null if no candidate
elif low contains "surrender" or "give up war":
    target = FindNeighborAtWar(state);  matched = Surrender(target?.NationId, "The cost of war is too great.")
else:
    b = ExtractBuildingFromMessage(low)   # first BuildingType whose name appears
    if b: matched = PrioritizeBuilding(b, CapitalId, GetReason(PrioritizeBuilding,...))
    else:
        r = ExtractResourceFromMessage(low)   # first ResourceType whose name appears
        if r: matched = FocusResource(r, CapitalId, GetReason(FocusResource,...))
if matched == null: return null
matched.RespondedToPlayer = true
if RollObey(personality, isWarMsg, rng): return matched
else: return null
```
**`RollObey`** (`:887-901`) — returns `rng.Next(100) < threshold`:
| Personality | war msg | non-war msg |
|---|---|---|
| Aggressive | 90 | 70 |
| Cautious | 30 | 50 |
| Expansionist | 60 | 75 |
| Mercantile | 40 | 60 |
| Pious | 50 | 65 |
| Cunning | 70 | 60 |
| Stoic | 40 | 50 |
| (default) | 50 | 50 |

### 4.4 R3 — `TryWarIntent` (`:387-537`) — only when `AtWarWith.Count>0`
```
personality = ParsePersonality(...)
ownSoldiers  = Military.SoldiersCount
enemySoldiers= EstimateEnemySoldiers(state, rng)
desired      = Military.DesiredSoldiers
casualties   = Military.CasualtiesLast60Ticks
losingBadly     = ownSoldiers < enemySoldiers * 0.4
cityUnderAttack = CityUnderDirectAttack(state)   # a "combat" event within last 5 ticks (:1075-1082)

# (a) losing badly
if losingBadly:
    if (personality in {Cautious,Pious}) and rng.Next(100) < 30:
        return Surrender(target=FindStrongestEnemy(state)?.NationId, "We cannot sustain these losses.")
    return BuildArmy(TargetSize = max(ownSoldiers*1.5, ownSoldiers+8), GetReason(BuildArmy,...))

# (b) a city is under direct attack
if cityUnderAttack:
    if personality in {Aggressive, Cunning}:
        return AttackCity(TargetCityId=FindEnemyCityToAttack(nation), "Strike while they are exposed.")
    return BuildArmy(TargetSize = max(nation.DesiredSoldiers, 12), "Reinforce the city under attack.")

# (c) already have a standing attack order → keep pressing
if nation.CurrentAttackOrder != null:
    return AttackCity(TargetCityId=nation.CurrentAttackOrder.TargetCityId, "Press the attack.")

# (d) army is ~ready (>=80% of desired) and no order yet → launch assault on nearest enemy city
if nation.CurrentAttackOrder == null:
    desiredN = nation.DesiredSoldiers
    actual   = ProfessionCapacity.CurrentCount(nation, Soldier)
    if desiredN>0 and actual >= desiredN*0.8:
        t = FindEnemyCityToAttack(nation)         # nearest enemy city by Manhattan (:1096-1124)
        if t.HasValue: return AttackCity(t, "army ready, attacking enemy capital")

# (e) below desired and have barracks → conscript a few
if desired>0 and ownSoldiers<desired and Military.BarracksCount>0:
    return Conscript(Count = min(4, desired-ownSoldiers), "Conscript to meet army target.")

# (f) build up (esp. if we've lost wars)
lostWarsRecently = Military.WarsLost > 0
targetSize = lostWarsRecently ? max(desired+8, 24) : desired+4
if desired==0 or ownSoldiers < desired*0.3 or lostWarsRecently:
    return BuildArmy(TargetSize=targetSize,
        reason = lostWarsRecently ? "Past defeats demand stronger legions." : GetReason(BuildArmy,...))

# (g) need more barracks (soldiers outgrew capacity: barracks*10 < soldiers, :1084-1091)
if NeedsMoreBarracks(state):
    return PrioritizeBuilding(Barracks, TargetCityId=WeakestCityId(state),   # fewest barracks (:1048-1059)
                              "We need more barracks to sustain our forces.")

# (h) no army and can't raise one → sue for peace
if ownSoldiers==0 and AtWarWith.Count>0:
    return MakePeace(AtWarWith[0], "We have no army and no means to raise one. Seek peace.")

# (i) heavy casualties → chance to pull back
if casualties>5 and rng.Next(100) < 40:
    return Retreat("Our forces need to recover.")

# (j) fallback: attack nearest enemy city if one exists, else null (cascade falls through)
t = FindEnemyCityToAttack(nation)
if !t.HasValue: return null
return AttackCity(t, GetReason(AttackCity,...))
```
Note `GetReason` has **no `AttackCity`/`Retreat`/`Surrender`/`Conscript`/`Observe` templates**, so those
fall to the `GetReason` fallback: first non-empty template of that IntentType, else `"No comment."`
(`:988-999`). (For AttackCity there is no entry at all ⇒ `"No comment."`.)

### 4.5 R3.5 — `TryProactiveWarIntent` (`:748-791`) — declare war during peace
```
if state == null: return null
if AtWarWith.Count > 0: return null
if HadRecentWar(state,120): return null          # peace_signed event within 120 ticks (:1238-1245)
if Military.SoldiersCount < 4: return null
for c in Cities: if c.Security?.Food == "Critical": return null   # never attack while starving
target = FindWeakNeighbor(state)                 # at_peace AND RelativeStrength < 1.2 (:1157-1164)
if target == null: return null
personality = ParsePersonality(...)
chance = { Aggressive:25, Cunning:15, Expansionist:15, Pious:8, _:0 }[personality]
if chance == 0: return null
if target.DistanceTiles < 80: chance += 10
if target.DistanceTiles < 40: chance += 10
if target.RelativeStrength < 0.7: chance += 10
if target.RelativeStrength < 0.5: chance += 10
if rng.Next(100) >= chance: return null
return DeclareWar(target.NationId, GetReason(DeclareWar,personality,rng))
```
(Cautious, Mercantile, Stoic never start a proactive war here.)

### 4.6 R4 — `TrySuccessionIntent` (`:539-556`)
```
if state.Heir == null and state.RoyalFamily != null:
    candidate = FindEldestRoyalChild(state)      # RoyalFamily member Role=="Royal", !IsHeir, max AgeTicks (:1177-1188)
    if candidate: return NameHeir(RoyalId=candidate.Id, "The line of succession must be clear.")
return null
```

### 4.7 R5 — `DefaultByPersonality` (`:558-681`) — peacetime default (always returns non-null)
```
personality = ParsePersonality(...)
atWar   = AtWarWith.Count > 0
desired = Military.DesiredSoldiers
capitalPop  = CapitalPopulation(state)              # population of IsCapital city (:1014-1019)
smallestPop = SmallestSatellitePopulation(state)    # min non-capital city pop, or -1 if <=1 city (:1021-1028)

# (1) peacetime disarmament
if !atWar and desired>0:
    if personality in {Cautious,Pious} and rng.Next(100)<20:
        return DisbandArmy("There is no need for such a force in times of peace.")

# (2) post-war behavior (peace_signed within 120 ticks)
if !atWar and HadRecentWar(state,120):
    if personality==Aggressive and rng.Next(100)<10:
        w = FindWeakNeighbor(state); if w: return MakeDeclareWar(...)
    if personality==Cautious and rng.Next(100)<30:
        return Idle("Preserving the peace after recent conflict.")

# (3) rebalance population if capital dwarfs a satellite (capitalPop > 2*smallestPop)
if smallestPop>=0 and capitalPop > smallestPop*2:
    if personality==Expansionist and rng.Next(100)<25: return MakeMigrate(...)
    if personality==Mercantile   and rng.Next(100)<15: return MakeMigrate(...)

# (4) rare unprovoked war ("floor" war): non-{Cautious,Stoic,Mercantile}, no recent war,
#     >=2 cities, at peace with everyone, rng.NextDouble() < 0.05
if personalityAllowsWar and !HadRecentWar(state,120) and Cities.Count>=2
   and AtWarWith empty and rng.NextDouble() < 0.05:
    t = FindWeakNeighbor(state); if t: return MakeDeclareWar(...)

# (5) Pious tiny idle chance
if personality==Pious and rng.Next(100)<2: return Idle("Reserved-Ready intent skipped.")

# (6) MAIN per-personality roll: roll = rng.Next(100), warCooldown = HadRecentWar(state,120)
switch personality:
  Aggressive:
     if !warCooldown and roll<15: w=FindWeakNeighbor; if w: return MakeDeclareWar
     if roll<25: return MakeFoundCity
     if roll<50: return MakePrioritizeBuilding
     if roll<60: return MakeFocusResource
     return MakeIdle
  Cautious:
     if roll<5:  return MakeFoundCity
     if roll<45: return MakePrioritizeBuilding
     if roll<65: return MakeFocusResource
     return MakeIdle
  Expansionist:
     if !warCooldown and roll<5: n=FindAnyNeighborAtPeace; if n: return MakeDeclareWar
     if roll<40: return MakeFoundCity
     if roll<65: return MakePrioritizeBuilding
     if roll<75: return MakeFocusResource
     return MakeIdle
  Mercantile:
     if roll<10: return MakeFoundCity
     if roll<35: return MakePrioritizeBuilding
     if roll<70: return MakeFocusResource
     return MakeIdle
  Pious:
     if !warCooldown and roll<5: n=FindAnyNeighborAtPeace; if n: return MakeDeclareWar
     if roll<10: return MakeFoundCity
     if roll<30: return MakePrioritizeBuilding
     if roll<40: return MakeFocusResource
     return MakeIdle
  Cunning:
     if !warCooldown and roll<10: w=FindWeakNeighbor; if w: return MakeDeclareWar
     if roll<25: return MakeFoundCity
     if roll<45: return MakePrioritizeBuilding
     if roll<60: return MakeFocusResource
     return MakeIdle
  Stoic (default):
     if roll<5:  return MakeFoundCity
     if roll<30: return MakePrioritizeBuilding
     if roll<40: return MakeFocusResource
     return MakeIdle
```

### 4.8 Intent-builder helpers (`Make*`, `:793-885`)
- `MakeDeclareWar(target)` → DeclareWar, TargetNationId=target.NationId, GetReason(DeclareWar).
- `MakeFoundCity` → FoundCity, Direction=`FirstOpenDirection(state)`, DistanceHint=Near. `FirstOpenDirection`
  (`:1190-1201`): first capital city's `FrontierOpenDirections[0]` parsed to Direction, else `N`.
- `MakeMigrate` → Migrate, FromCityId=`LargestCityId`, ToCityId=`SmallestCityId`, Count=10 (`:816-827`).
- `MakePrioritizeBuilding` → PrioritizeBuilding, BuildingType=`PickBuildingNeed(state)`, TargetCityId=CapitalId.
  `PickBuildingNeed` (`:1203-1216`): for the capital city — Food Critical/Insufficient⇒Farm;
  else Housing Critical/Insufficient⇒House; else Wood Critical/Insufficient⇒Sawmill; else **Farm**.
- `MakeFocusResource` → FocusResource, Resource=`MostScarceResource(state)`, TargetCityId=CapitalId.
  `MostScarceResource` (`:1218-1236`): sums each resource across all city stockpiles; returns the smallest
  in priority order Food→Wood→Planks→Stone→Iron (ties resolve to earlier in that order).
- `MakeIdle(personality)` → Idle, GetReason(Idle).
- `MakeBuildArmy` → BuildArmy, TargetSize=max(SoldiersCount+8, EstimateEnemySoldiers).
- `MakeNameHeir(state)` → NameHeir on `FindEldestRoyalChild`, or **null** if none.

### 4.9 Neighbor-selection helpers (all read `state.Neighbors`)
- `FindNeighborByName(state, low)`: neighbor whose lowercased `Name` is a substring of the message (:1129-1136).
- `FindNeighborAtWar(state)`: first neighbor whose id ∈ `Diplomacy.AtWarWith` (:1138-1144).
- `FindStrongestEnemy(state)`: `DiplomaticState=="at_war"` with max `RelativeStrength` (:1146-1155).
- `FindWeakNeighbor(state)`: first `at_peace` neighbor with `RelativeStrength < 1.2` (:1157-1164).
- `FindAnyNeighborAtPeace(state)`: first `at_peace` neighbor (:1166-1172).
- `FindEnemyCityToAttack(nation)`: nearest (Manhattan) city among nations currently at war with us (:1096-1124).
  (Note the `NationStateDoc` overload always returns null, :1093-1094 — the live-`Nation` overload is used.)

### 4.10 REASON_TEMPLATES (flavor text) — `:903-999`
`Dictionary<IntentType, Dictionary<Personality, string[]>>` with 3 lines each, for intents:
`DeclareWar, MakePeace, BuildArmy, FoundCity, Migrate, PrioritizeBuilding, FocusResource, Idle`
(each × all 7 personalities). `GetReason(type,personality,rng)` picks `templates[rng.Next(len)]`; if the
personality/type is missing it falls back to the first non-empty template, else `"No comment."`.
(Exact 168 strings are in the file; reproduce them literally for 1:1 flavor. Example — DeclareWar/Aggressive:
`{"Their lands shall be ours.","Time to test their steel.","Honor demands answer."}`.)

---

## 5. NATION-STATE SNAPSHOT — `LLM/NationStateSerializer.cs` (the heuristic's inputs)

`Serialize(nation, currentTick, playerMessage)` → `NationStateDoc` (`LLM/Schema/NationStateDoc.cs`,
`SchemaVersion="v2"`). The Mock reads a subset; these are the fields that drive decisions:

- **World** (`:45-53`): `Size=[480,480]`, `Tick=currentTick`, `Year=Tick/TICKS_PER_YEAR(60)`.
- **Leader** (`:84-98`): `Id=GetInstanceID()`, `Name`, `AgeTicks`, `Personality` (enum→string),
  `LastIntent=nation.LastLeaderIntent`, `LastReason`, `PreviousLeaderSummary=nation.LastLeaderContextSummary`.
  Null if no leader.
- **Consort/Heir/RoyalFamily** (`:100-144`): `RoyalMember = {Id, Role(str), AgeTicks, Personality, IsHeir}`;
  RoyalFamily includes only alive members with a Role. `HeirInfo` adds `IsAdult`.
- **Cities** (`CityInfo`, `:146-223`): `Id`, `Name="City_{id}"`, `IsCapital`, `Coord`, `Population`,
  `DominantBiome`+`BiomeMix` (30 random samples radius 10, seeded by center), `NearbyResources`
  {Trees,StoneNodes,IronNodes,Fauna} scanned within Manhattan radius 15, `FrontierOpenDirections`
  (which of the 8 dirs border an unowned tile, `:300-342`), `Stockpile` {Food,Wood,Planks,Stone,Iron,Gold},
  `Buildings` (name→count of constructed), `ConstructionQueue` (unconstructed names),
  `Security = {Food,Housing,Wood}` (see §5.1), `Professions` (name→count).
- **Neighbors** (`NeighborInfo`, `:344-403`): one per *other* nation. `NationId`, `Name`,
  `RelativeDirection` (8-way from capital-to-capital angle, `:405-420`), `DistanceTiles` (Manhattan
  capital-to-capital), `SharesBorder`, **`RelativeStrength`** (see §5.2), `ColorHex`, **`DiplomaticState`**
  ∈ {`"at_war"`,`"at_peace"`}, and a `cities` list.
- **Military** (`MilitaryInfo`, `:455-488`): `SoldiersCount` (alive military members), `SoldiersPerCity`,
  `BarracksCount` (constructed), `DesiredSoldiers=nation.DesiredSoldiers`,
  `CasualtiesLast60Ticks = NationMetricsTracker.DeathsCombatLast60Ticks`, `WarsWon`, `WarsLost`.
- **Diplomacy** (`DiplomacyInfo`, `:490-507`): `AtWarWith` / `AtPeaceWith` = lists of other nation ids,
  partitioned by `WarRegistry.IsAtWar`.
- **RecentEvents** (`:509-518`): `nation.RecentEvents.GetAll(currentTick)` — pruned copy (see §6).

### 5.1 Security levels — `Nations/CityStateEvaluator.cs`
`SecurityLevel` enum (`Nations/CityState.cs`): `{ Critical, Insufficient, Adequate, Surplus }`.
- **Food** (`ClassifyFood`, thresholds are food-per-member): `<0.3`⇒Critical, `<0.7`⇒Insufficient,
  `<1.5`⇒Adequate, else Surplus; 0 members ⇒ Surplus. (`FOOD_CRITICAL=0.3, FOOD_INSUFF=0.7, FOOD_ADEQUATE=1.5`.)
- **Housing** (`ClassifyHousing`): based on unmarried adults vs house couple-capacity (multi-branch, :62-108).
- **Wood** in the doc = `ResourceDeficit_Plank ? Insufficient : Adequate`, where deficit ⇔ `planks < 5`
  (`RESOURCE_DEFICIT_THRESHOLD=5`, serializer `:201-209`).

### 5.2 RelativeStrength (the war-appetite signal) — `NationStateSerializer.cs:365-373`
```
theirSoldiers = CountSoldiers(other);  theirPop = other.Members.Count
ourSoldiers   = CountSoldiers(self);   ourPop   = self.Members.Count
ourTotal = ourSoldiers + ourPop
RelativeStrength = ourTotal>0 ? round((theirSoldiers+theirPop)/ourTotal, 2) : 1.0
```
So RelativeStrength **<1 ⇒ neighbor weaker than us** (attractive target). `FindWeakNeighbor` uses `<1.2`.
`CountSoldiers` = alive members with `IsMilitary` (Soldier or General).

---

## 6. RECENT EVENTS — `Nations/RecentEventsBuffer.cs` + producers
`RecentEvent = { int Tick; string Type; Dictionary<string,object> Args; }` (`LLM/Schema/RecentEvent.cs`).
Buffer: `MAX_EVENTS=10`, `MAX_AGE_TICKS=60`; `Add` prunes by age then trims oldest beyond 10.
The Mock keys on `Type` string and `Args` values. Event producers and their args:

| Type | Args | Emitted by |
|---|---|---|
| `war_declared` | `by`, `against`, `reason` | WarRegistry.DeclareWar :88-92 (added to both nations) |
| `peace_signed` | `a`, `b` | WarRegistry.MakePeace :134-138 |
| `war_won` / `war_stalemate` | `vs` | WarVictorySystem :94-95 (winner) |
| `war_lost` | `vs` | WarVictorySystem :96-97 (loser) |
| `decision` | `intent,reason,source,status,result_reason` | IntentDispatcher.Apply :64-73 |
| `food_crash` | `city_id` | DecisionScheduler.CheckFoodCrash :213 |
| `army_decimated` | `soldiers_before, soldiers_after` | DecisionScheduler.CheckArmyDecimation :231 |
| `building_destroyed` / `city_lost` | `building_type`, `city_id` | DecisionScheduler.OnBuildingDestroyed :150-155 |
| `royal_death` | `royal_id, name, role` | SuccessionSystem.HandleAgentDied :80-85 |
| `royal_birth` | `name, is_heir` | RoyalReproductionSystem :196-200 |
| `leader_succession` | `previous_leader,new_leader,reason,case` | SuccessionSystem.ProcessSuccession :183-189 |
| `city_founded` | `city_id, coord_x, coord_y, source` | CityFoundingSystem :274 |
| `heir_without_fallback` | `leader_id` | SuccessionSystem.RedesignateHeir :137 |
| `combat` | (`our_casualties` expected by heuristic) | combat system (Economy/Combat) |
| `first_contact` | — | contact/exploration system |

`NationMetricsTracker` (`Nations/NationMetricsTracker.cs`): 60s sliding window of Birth/Death/DeathCombat
events → `NationMetrics{BirthsLast60Ticks, DeathsLast60Ticks, DeathsCombatLast60Ticks}`.
`DeathsCombatLast60Ticks` is what feeds `Military.CasualtiesLast60Ticks`.

---

## 7. SUCCESSION — `Agents/SuccessionSystem.cs` (+ heir logic in dispatcher/repro)

`MIN_WORKERS_FOR_SURVIVAL = 10`. On any tracked worker death (`HandleAgentDied`, `:68-114`):
1. If the dead worker had a `Role`: remove from `RoyalFamily`, emit `royal_death`
   (`{royal_id,name,role}`), strip `" (leader)"` name suffix if leader, notify scheduler
   (consort death ⇒ `consort_died` wakeup).
2. If dead worker `IsHeir` and not the Leader, and nation has a living leader ⇒ `RedesignateHeir`.
3. If dead worker was the **Leader** ⇒ queue succession (deduped per nation), processed in `LateUpdate`.

### 7.1 `RedesignateHeir` (`:116-141`) — auto-heir among leader's children
Pick the **eldest living child** of the current leader (`OrderByDescending(Age)`); clear all children's
`IsHeir`, set the chosen. If no child ⇒ emit `heir_without_fallback` + scheduler wakeup (lets the Mock's
R4 name a Royal instead).

### 7.2 `ProcessSuccession` (`:153-193`) — when a Leader dies
```
nation.LastLeaderContextSummary = GenerateLeaderSummary(nation, previousLeader)   # epitaph (see 7.4)
successor = FindSuccessor(nation, previousLeader)
if successor == null:
    totalAlive = count of alive Worker members
    if totalAlive < MIN_WORKERS_FOR_SURVIVAL(10):  DissolveNation(nation); return
    successor = SpawnNewRoyalAsLeader(nation)      # spawn a fresh adult, random personality
    if successor == null: return
case = ResolveSuccessionCase(nation, previousLeader, successor)   # 1..5 (see 7.3)
PromoteToLeader(nation, successor)
emit "leader_succession" {previous_leader, new_leader, reason=DetermineCause(prev), case}
fire OnLeaderSuccession(nation, prev, successor)   # → scheduler wakeup, immediate decision
```

### 7.3 `FindSuccessor` — the ORDER OF SUCCESSION (`:195-254`)
1. **Designated Heir** — `nation.Heir` if alive and ≠ previousLeader. → case 1.
2. **Eldest adult child** of previous leader (`Family.Children`, `Stage==Adult`, `OrderByDescending(Age)`);
   promotes it to heir first. → case 2.
3. **Youngest adult `Role==Royal`** (`OrderBy(Age)` — note: *youngest* royal). → case 3.
4. **A random ordinary adult worker** with `Role==null`, `Age < LifetimeSeconds*0.5` (young), and (if no
   consort exists) unmarried — `OrderBy(Random.value)`. If chosen and there's no consort and it has an
   ordinary spouse, that spouse becomes **Consort** (random personality, lifetime ×2.5, added to
   RoyalFamily). → case 4.
5. **The Consort** if alive. → case 5.
6. Else null ⇒ dissolve-or-spawn (above).

`ResolveSuccessionCase` (`:357-365`) returns 1/2/3/4/5 by the same categories (0 otherwise).

### 7.4 `PromoteToLeader` (`:256-304`) & summary
- Strips `" (leader)"` from the previous live leader's name. Sets `Role=Leader`, `IsHeir=false`.
- Renames to `Protagonist_{nationId}_{instanceId} (leader)` when coming from a non-protagonist/pooled.
- **If promoted from an ordinary worker (`oldRole==null`)**: `LifetimeSeconds *= 2.5` (`GetLifetimeMultiplier`).
- Adds to RoyalFamily, sets `nation.LeaderPromotionTime = realtime`, attaches/show a LeaderMarker.
- `BootstrapLeaderForNewNation` (`:306-312`) = same but assigns a random personality first (game start).
- `GenerateLeaderSummary` (`:330-342`): `"{name} reigned for {lifetime}s, {mothered|fathered} {N} children,
  won {WarsWon} wars and lost {WarsLost}, founded {citiesFounded} cities, died of {cause}."` — this string
  becomes the next leader's `PreviousLeaderSummary`, which triggers R1's "Mourning my predecessor" Idle.
- `DetermineCause` (`:344-355`): Killer is a protagonist ⇒ `"assassination"`; Killer other ⇒ `"combat"`;
  `Hunger>=100` ⇒ `"starvation"`; else `"old age"`.

### 7.5 Royal births & heir bootstrap — `Agents/RoyalReproductionSystem.cs`
- Timers: `PREGNANCY_DURATION=45s`, `ROYAL_COOLDOWN_POST_PARTUM=90s`, `TICK_INTERVAL=5s`.
- Leader+Consort couple auto-reproduces (female of the pair is mother) when off-cooldown, not pregnant,
  and `CanReproduce` (city Food not Critical and `Food >= max(10, members/8)`, `:227-235`).
- Non-heir adult female Royals with a spouse may bear **one** child (`_nonHeirChildrenBorn` cap 1).
- `OnRoyalChildBorn` (`:169-225`): child `Role=Royal`; **personality inheritance**: 50% inherit a parent's
  personality (50/50 father/mother), else 50% a uniformly random personality (`:172-183`); sets
  Father/Mother, `LifetimeSeconds *= 2.5`; **if `nation.Heir==null` ⇒ child.IsHeir=true**; emits
  `royal_birth` + scheduler wakeup.

### 7.6 Memory inheritance
There is **no per-agent episodic memory carried forward** beyond: (a) the textual
`PreviousLeaderSummary` epitaph passed leader→leader, and (b) the nation-level `RecentEvents` ring buffer
(shared, decays in 60 ticks / 10 entries). Personality is inherited genetically at birth (7.5), not learned.

---

## 8. WAR & DIPLOMACY RESOLUTION — `Nations/WarRegistry.cs`, `Economy/Combat/WarVictorySystem.cs`, `MilitaryMobilization.cs`

### 8.1 `WarRegistry` — the diplomacy state machine
- War state = a `HashSet<long>` of symmetric `EncodePair(min<<32|max)` keys (`:33-38`); only two states
  exist per pair: at-war or at-peace. **No alliances/tribute/trade are implemented** (those intents stub).
- `DeclareWar(a,b,reason)`: adds pair, records `_warStartTimes[key]=Time.time`, emits `war_declared` to
  both, fires `OnWarDeclared` (→ scheduler wakes both).
- `MakePeace(a,b)`: removes pair, emits `peace_signed` to both, fires `OnPeaceSigned` (→ SuccessionSystem
  tallies win/loss, §8.4), wakes both, and schedules a **DesiredSoldiers auto-reset to 0** after
  `PEACE_DESIRED_SOLDIERS_RESET_DELAY=50s` if still at peace (`:147-179`).
- `GetWarAge(a,b)` = real seconds since war start (used by R0 stalemate and WarVictorySystem).
- `IsAtWar(nation)` = at war with *anyone*; `EnemiesOf(nation)` enumerates current enemies.

### 8.2 Mobilization — `Economy/Combat/MilitaryMobilization.cs`
- `MIN_DEFENSIVE=6`, `MIN_AGGRESSIVE=12`. `ArmDefensively(n)`: `DesiredSoldiers=max(cur,6)`,
  ensure a Barracks blueprint exists (war priority), `HasActiveWarChest=true`. Called on both sides at
  war declaration. `Demobilize(n)`: `DesiredSoldiers=0`, clear attack order & war-priority & directed
  targets. Called on both sides at peace.
- `EnsureBarracksCapacity`: if capital has no Barracks, spawn a war-priority Barracks blueprint.

### 8.3 Conquest / victory — `Economy/Combat/WarVictorySystem.cs` (1 Hz `Tick`)
For each warring pair (`WAR_MAX_REAL_SECONDS=300`):
1. If a side has **0 cities** ⇒ other side `war_won`, force peace. (:55-62)
2. Else if a side has **<10 members** ⇒ other side `war_won` **and that side is dissolved** (case-B loser).
   (:64-73, :101-107)
3. Else if `GetWarAge > 300s` ⇒ `war_stalemate`, force peace (logged once). (:75-82)
On resolve: `MakePeace`, emit `war_won`/`war_stalemate` (`{vs}`) to winner and `war_lost` (`{vs}`) to loser
(:86-99). City **capture** proper is handled elsewhere (`WarRegistry.NotifyCityConquered`/`OnCityConquered`
consumers, and `NationManager.RemoveCity` when a city's TownHall/Castle falls — see §1); the strategic
outcome that the heuristic sees is the events above + shrinking `Cities`/`Members`.

### 8.4 Win/Loss tally — `SuccessionSystem.OnPeaceSigned` (`:367-381`)
On every peace: the side with **more (or equal) cities** gets `WarsWon++`, the other `WarsLost++`.
These feed `Military.WarsWon/WarsLost`, which R3 rule (f) uses ("Past defeats demand stronger legions").

### 8.5 Nation dissolution — `SuccessionSystem.DissolveNation` (`:383-437`)
Make peace with all enemies; migrate every living worker to the **nearest other nation** (reset Role/heir,
reassign city/house/nation); abandon all buildings; release all territory to unowned (-1); clear cities &
royal family; set `IsDissolved=true`; fire `OnNationDissolved` (scheduler purges its queued requests).

---

## 9. LLM PATH (present but OFF in the replica) — `LLM/LLMClient.cs`, `ProtagonistDecisionSystem.cs`
`ProtagonistDecisionSystem.GetIntent`: if `ActiveSource==LLM`, POST the serialized `NationStateDoc` (+
optional player message) to `http://127.0.0.1:8000/protagonist/decide` (120s timeout), parse the JSON
intent (`ParseIntent`, snake_case keys), and **on any failure/null fall back to `MockHeuristic.Decide`**
(`ProtagonistDecisionSystem.cs:62-67`). For the 1:1 replica with no server: `ActiveSource` resolves to
`Mock` and `LLMClient` is never exercised — **the Mock heuristic is the complete decision authority.**

---

## 10. Replica checklist (behavioral parity)
1. RNG: `System.Random(worldSeed ^ nationId ^ tick)`, .NET `Next`/`NextDouble` semantics — required for
   identical sequences.
2. Cascade order R0→R1→R2→R3(if at war)→R3.5→R4→R5, first non-null wins; R5 always returns.
3. Personality default ⇒ Stoic on unknown/empty.
4. Thresholds exactly as in §4 (all the `< 0.4`, `*0.8`, `*1.5`, `rng.Next(100)<X`, `NextDouble()<0.05`, etc.).
5. RelativeStrength = (theirSoldiers+theirPop)/(ourSoldiers+ourPop), rounded 2dp; <1.2 = "weak".
6. Event window: react only to RecentEvents with `tick-ev.Tick <= 30` (R1) / `<=5` (combat) / `<=120` (recent war).
7. Dispatcher validation + the "no living leader ⇒ Rejected" gate; Surrender/Observe/trade/alliance are
   NotImplemented stubs (emit decision event, no world change).
8. Scheduler cadence 45s/nation + the event wakeups list; Mock has no real-time cooldown.
9. Succession order (Heir→eldest adult child→youngest Royal→random young worker(+spouse→Consort)→Consort→
   spawn/dissolve), lifetime ×2.5 on promotion/birth, epitaph string, `DetermineCause` mapping.
10. War auto-resolution at 1 Hz: 0 cities / <10 members(+dissolve) / >300s stalemate; win/loss by city count.
```
