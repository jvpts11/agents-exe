# agents.exe ORIGINAL — Faithful Program Map (from canonical ZIP source)

Source: `Downloads\agents-exe-source.zip` → `unity-project/Assets/Scripts` (149 .cs, ~19.6k lines).
This is the source of truth for the LDP3 replica. All facts quoted from code.

---

## 0. WINDOW / IDENTITY
- Window **1280×720**, windowed, resizable. Product `agents.exe`, company `IPCA MEDJD`, "a Worldbox-style civilization simulator".

## 1. SCENE WIRING (Editor/BuildGameSceneMenu.cs) — the master architecture
Systems added to Game scene, in order (execution order roughly follows):
Camera(+CameraController) → Grid + Tilemap_Terrain(TilemapPainter) + Tilemap_Territory_Fill + Tilemap_Territory_Border
→ WorldBootstrap → NationManager → WarRegistry → CombatSystems(ConquestSystem, AttackOrderSystem, GeneralPromoter, CombatEventRecorder)
→ TerritoryPainter → AgentSpawner → NationStockpileOverlay → WarOverlay → ResourceNodeManager → ResourceNodeSeeder
→ BuildingManager → TimestampedLogger → HungerSystem → MarriageMatchmaker → PregnancySystem → RoyalReproductionSystem
→ CityFoundingSystem → BuildingGrowthSystem → TerritoryExpansionSystem → CityMigrationSystem → SuccessionSystem
→ NationMetricsTracker → ProtagonistDecisionSystem → DecisionScheduler → DecisionToastController → DeathNotification
→ SettingsMenu → GameSpeedHud → NationStateLogger → CityStatusLogger → CityScreenshotCapture

- Resources: **Food, Wood, Stone, Iron, Gold** (5).
- Professions: **Civil, Soldier, Scout, Protagonist** (sprites NE/SE/SW/NW each).
- Buildings (13): Castle, TownHall, Farm, Sawmill, HuntersLodge, FishingHut, Smithy, Bakery, Brewery, Forge, Marketplace, Barracks, House.
- Resource nodes: Tree(Pine/Oak/Palm), Bush, Stone, Iron, Fauna(Deer).

## 2. CAMERA (BuildGameSceneMenu + Player/CameraController.cs)
- Orthographic, initial **size 130**, position **(0, 120, -10)**, bg **(0.045,0.05,0.085)**, near/far −100/100.
- Zoom clamp **[1.5, 250]**; panSpeed 20, zoomSpeed 5; pan scales `size/10`; zoom step `sign(scroll)*5*(size*0.05)`.
- WASD: W=+y(up), S=−y(down), D=+x(right), A=−x(left). transparencySortMode=CustomAxis axis(0,1,0) → sort by world Y.
- Camera stays at authored (0,120); does NOT recenter on world.

## 3. GRID / RENDER (isometric)
- Grid cellLayout=IsometricZAsY, **cellSize (1, 0.5, 0.25)** (2:1 diamonds; Z step 0.25 = elevation), swizzle XYZ.
- 3 stacked tilemaps: Terrain(sort 0), Territory_Fill(sort 1), Territory_Border(sort 2). All Chunk/TopRight.
- Tile PPU **64** → diamond 64px wide × 32px tall. Cliff art 48px tall, pivot (0.5, 16/48). Flat pivot (0.5, 0).

## 4. WORLDGEN (World/HeightmapGenerator + BiomeMapper + TilemapPainter) — [AGENT 1 ✓]
- **480×480** tiles, seed default 42. Menu overrides ONLY seed; rest = WorldGenParams.Default.
- Noise: Unity `Mathf.PerlinNoise` + fBm. RNG = `System.Random(seed)`, draw order load-bearing.
- Pipeline: 6 RNG offsets (ox,oy,warpOx,warpOy,boundaryOx,boundaryOy each in ±100000) → PlaceContinentSeeds (4 continents grid 2×2 + jitter±30%, 7 islands) → per-cell: domain warp(oct3) + base fBm(oct6,pers0.5,lac2,scale180) + boundary noise(low oct2 scale38 str0.45 + high oct2 scale12 str0.22) + continent mask(inverse smoothstep, strength0.65) → normalize [0,1].
- **Biome by normalized elevation ONLY** (no moisture/rivers/lakes):
  - DeepOcean v<0.252 | Ocean <0.42 | Beach <0.46 | Grassland <0.62 | Mountain <0.78 | Snow ≥0.78.
- Biome enum: DeepOcean=0,Ocean=1,Beach=2,Grassland=3,Mountain=4,Snow=5.
- Elevation stacking: elevByBiome {0,0,0,0,1,2}. Mountain adds z=1 tile, Snow z=2. Under raised terrain the z=0 ground is forced to Grassland flat.
- Cliffs: compare SE neighbor (x,y-1) & SW neighbor (x-1,y) elevation; both lower→Cliff_Both, SE→Cliff_SE, SW→Cliff_SW, else flat.
- **Per-tile variant deterministic by position hash**: `h=(x*73856093)^(y*19349663); idx=((h%len)+len)%len`. NOT per-frame random (avoids flicker). color=white (no tint).
- Biome COLORS not in code (external PNGs, absent) → replica chooses faithful palette.
- Passability: ground walkable ONLY on Grassland/Beach; ocean/mountain/snow impassable.

## 5. MENUS (Win95 aesthetic; Editor/BuildMainMenuScene + BuildUIScenes)
Palette: C_GRAY(192,192,192) face, C_GRAY_DK(128) bevel-shadow, C_WHITE bevel-hi, titlebar(8,26,147), logo GOLD(228,185,60), desktop BG(26,58,90), C_BLUE_DK(0,0,128).
Bevel: 2px top/left white + bottom/right dark = raised; inverted = sunken.
- **MainMenu** 426×413 (offset y −20): titlebar "agents.exe — Main Menu" (fs17), [X] close btn, logo "agents.exe" fs58 GOLD + subtitle fs16, divider@130, 4 btns 240×40 gap11 first@153 (New Game/Load Game/Options/Quit) label fs22 black, footer "v0.1 prototype • TEIA 2026 • IPCA" + "João V. P. Tavares  21871", taskbar h25 (Start btn 68w + clock 70w "21:42"), top-right "1920×1080 • 16 bpp • agents.exe".
- **NewGame** 720×510: logo "New Game", divider@90, Seed: label+sunken input(default"42", ph"Enter seed..."), toggle "Randomize personalities", toggle "Use LLM (requires server.py running)", Start(br)/Back(bl).
- **LoadGame** 800×510: header "Choose a save to continue", 5 slots "Slot N — empty"+DEL, Back.
- **Options** 614×480: tabs Gameplay/Display/Audio/Debug; AI Mode section (radio "• Phi-4-mini LoRA (real fine-tuned model via FastAPI)" / "• Heuristic mock AI (rule-based fallback, no LLM)"), "Master volume:" slider(blue fill 0,0,180), Use LLM toggle, Apply/Back.
- **Pause** overlay 320×340 (Esc; backdrop black a160): "Day X • Population: N • Nations: N", btns Resume/Save Game/Load Game/Options/Quit to Menu; SavePanel sub-screen w/ slots.
Flow: MainMenu→(NewGame|LoadGame|Options)→Game; GameSessionState static carrier (PendingWorldGen/PendingDecisionSource/PendingLoadSlotPath/MenuOverrideActive). Pause via PauseMenuSpawner (Esc, timeScale 0/1).

## 6. AGENTS — SOCIAL / LIFECYCLE (Agents/*) — [AGENT 4 ✓]
- **4 nations** (NationManager numNations=4). Capitals: capitalSeed=7, min-distance 100 Manhattan, need ≥400 reachable tiles in radius 20, reachable from capital[0]. Initial territory = disk radius **10**.
- **Per nation: 50 workers (gender strictly alternated → 25M+25F) + royal couple (Leader+Consort) = 52. Total start = 208.** spawnSeed=42. Workers spawn on random walkable tile in territory (fallback capital); royals at capital center.
- Worker lifetime **360–540s** random; initial adults force-aged 60–90s. Protagonists lifetime **×2.5**.
- Tick: round-robin **4 groups**, one group/frame (52 agents/frame).
- Natural (non-combat, non-royal) death → respawn replacement adult into same city. Combat/protagonist death → no respawn.
- **LifeStage**: Child (Age<60s) / Adult (≥60s). **Elder defined but NEVER reached.** Children only wander (no profession).
- **Death**: old age (Age≥lifetime) | starvation (hunger→100 ⇒ hp0) | combat (TakeDamage). IsHungry hunger>70; reproduce blocked hunger≥30; hp/hungerMax=100.
- **7 Personalities** (categorical): Aggressive,Cautious,Expansionist,Mercantile,Pious,Cunning,Stoic. **Affect ONLY the nation Leader's strategy** (MockHeuristic DefaultByPersonality thresholds for war/foundCity/build/focus/idle). Plain workers default Aggressive, personality irrelevant to worker behavior. Royals random; child 50% inherit/50% random.
- **12 Professions**: Civilian,Woodcutter,Farmer,Hunter,Builder,Sawyer,TreePlanter,StoneMiner,IronMiner,Trader,Soldier,General. Capacities: Woodcutter=TownHalls×4, Sawyer=Sawmills×5, TreePlanter=Sawmills×1, Farmer=Farms×2, Hunter=Lodges×3, StoneMiner=TownHalls×2, IronMiner=TownHalls×2, Trader=(cities≥2?max(4,cities×4):0), Builder=blueprints×2, Soldier=Barracks×12, Civilian=∞, General=promote-only. **Females never Soldier/General.** Pick priority: (male+atWar+slot)→Soldier, blueprints→Builder, food-insecure→Farmer else Hunter, else best score. Re-eval every 10s, switch margin 0.40.
- Sprites: only **Civil** (all economic) + **Soldier** (Soldier+General) sets exist, 4 dirs each. Protagonists → Royal sprites, scaled; royal child 0.7×.
- **Marriage** (MarriageMatchmaker, tick 2s/nation): unmarried male needs free constructed House in city; pair with unmarried non-ancestor female (BFS depth≤6); both adults; mixed royal/commoner promotes commoner (+×2.5 life, random personality).
- **Pregnancy** (tick 1s): female+adult+spouse+house; conception needs both hunger<30 + spouse house + CanReproduce (city food≥max(10,members/8), not Critical) + 0.5 roll; duration **45s**; birth 1 child (gender 50/50, Civilian, inherits mother house/family).
- **Royal reproduction** (tick 5s): Leader+Consort auto; post-partum cooldown 90s; non-heir royal females ≤1 child; royal child ×2.5 life, becomes Heir if none.
- **Roles**: Leader/Consort/Royal + separate IsHeir flag. **Succession** on leader death: Heir → oldest adult child → youngest adult Royal → random ordinary adult(<half life) → Consort → else if <10 workers **dissolve nation** (migrate to nearest capital) else spawn new leader.
- Overhead UI: WorkerBubble (alert/work/sleep, sort10), WorkerCarryIcon (sort11), WorkerDamageFlash (red 0.18s), SoldierHpBar (military only, green/yellow/red, maxHp100).

## 7. NATIONS / TERRITORY / CITIES / WAR (Nations/*, World/TerritoryPainter) — [AGENT 2 ✓]
### Nations
- 4 nations, names "Nation 1".."Nation 4". Capitals SEEDED (capitalSeed=7): shuffle walkable tiles, truncate 2000, pick first tile with Manhattan≥100 to others + ≥400 reachable in radius20 + reachable from capital[0].
- **EXACT nation colors** (NationPalette, alpha 1.0), id-indexed wrap 16:
  0 red (0.85,0.20,0.20)#D93333 | 1 blue (0.20,0.45,0.85)#3373D9 | 2 green (0.30,0.75,0.30)#4CBF4C | 3 yellow (0.95,0.85,0.25)#F2D940.
  (5+: purple(.65,.30,.75), teal(.25,.80,.80), orange(.95,.55,.20), pink(.95,.45,.70), brown(.50,.30,.10), lime(.55,.85,.25)...)
### Territory model
- **int grid TileNation[x,y]**, -1=unowned, ≥0=nation id. Parallel Nation.Territory/City.Territory HashSets + cellToCity dict.
- Initial: **filled disk radius 10** around capital (ignores biome, only clipped to bounds → capitals can own ocean).
- **Growth** (TerritoryExpansionSystem, tick **5s/nation**, TILES_PER_TICK=5/city): 4-connected frontier (von Neumann) into in-bounds + **walkable + unowned + Grassland/Beach**. Costs city wood `2+floor(tiles/50)`; stop if unaffordable. Housing gate: if planks<5 AND houses<pop/4 → no expand. Pick random among top-8 nearest/bias. Never overwrites owned; contested only via conquest. Rebuild flowfield ≤ every 20s.
- OnTerritoryChanged fires per ownership change (when _ready) → **full-map repaint**.
### Territory RENDERING (TerritoryPainter — 2 tilemaps) ★ fixes the border bug
- **Fill pass**: every owned cell gets tileFill; then `fillTilemap.SetColor = GetTerritoryFillColor(id)` = nation RGB **@ alpha 0.40** (translucent).
- **Border pass**: `PickBorder` gives ONE directional sprite per boundary cell (first match): (x,y-1) border→**SE**, (x-1,y)→**SW**, (x+1,y)→**NE**, (x,y+1)→**NW**; else none. Border tiles keep **solid nation color (alpha 1.0)** — SetColor never called on border layer. So: translucent fill + solid thin directional edge.
- `IsBorder(nx,ny)` = neighbor OOB, OR different nation (incl -1), OR different CITY of same nation (→ internal city partition lines too).
### Cities
- Capital = Cities[0]. Found (CityFoundingSystem tick 5s/nation): pop≥60 + constructed Farm + homeless≥8 + cooldown 180s; site walkable Grassland, dist [30,80] to own cities, ≥50 cross-nation; radius 5; +100 wood, Farm blueprint, 10 adults (5M/5F).
- Migration (tick 3s): if >1 city, move ≤5 agents/nation/tick from pop>avg+5 to pop<avg-5 (single adult or whole family).
- Dissolution (tick 5s): city with buildings but no constructed TownHall → RemoveCity (tiles→-1); orphans migrate or **found new nation** (radius 8, next id/color). Nation with 0 cities → IsDissolved (id kept stable).
### War / Conquest
- WarRegistry: symmetric war set; DeclareWar/MakePeace + events; post-peace DesiredSoldiers→0 after 50s.
- **ConquestSystem** (tick 1s): count enemy military within Euclidean radius 5 of city.Center; if ≥2 attackers dwell **15s continuous** → FlipOwnership: move city to attacker, reassign buildings, **transfer every city tile** OwnTile(attacker), re-tint members; capital loss rebuilds castle at new Cities[0]. 30s post-flip grace.
### Support
- ResourceStockpile: Food,Wood,Stone,Iron,Gold,Planks. Add(>0), TrySpend, ForceSpend(floor -999).
- CityStateEvaluator food per-member: CRITICAL 0.3 / INSUFF 0.7 / ADEQUATE 1.5; ExpansionPressure=members/territory.

## 8. WORKER CORE / MOVEMENT / RENDER (Agents/Worker.cs, Pathfinding) — [AGENT 3 ✓]
- **Position = continuous world Vector2** (z always 0). Grid cell derived via WorldToCell. Cell→world via Grid.GetCellCenterWorld (iso projection lives in Unity Grid, NOT code: cellSize (1,0.5,0.25) from scene builder).
- **TWO-PHASE UPDATE** (flicker-critical):
  - `Tick()` = AI/decision, called round-robin **1 of every 4 frames** (tickGroups=4). Sets `_moveVelocity` + `Direction`; does NOT move transform. Velocity persists between ticks.
  - `Update()` every frame integrates: `pos += moveVelocity * SpeedMultiplier * Time.deltaTime`. No FixedUpdate. Frame-rate independent.
- Speed **1.5** u/s (wander 1.5, war-march ×1.3, engage-strafe ×0.30). SpeedMultiplier ≈ always 1.
- **Pathfinding = FlowField BFS** (8-connected, no corner-cutting), per-target cached. FlowFieldVelocity aims at **next cell's center** → continuous re-aim = the built-in smoothing. Walkable = Grassland/Beach only. Soldiers within 15u + clear path steer straight.
- Per-frame wall-slide (check midpoint+full; else slide X or Y). Stuck after **300 blocked frames** → reset to Wandering + teleport to walkable ring (radius 2+2·resets, cap 12) + 20s cooldown.
- Wander: bias to city center, radius 5, repick dir every 2–4s or on wall probe.
- **Direction = pure sign quadrant** (Direction.cs): x≥0,y≥0→NE; x≥0,y<0→SE; x<0,y<0→SW; else NW. Set from velocity only when |vel|²>0.001. ★**NO hysteresis → flicker when velocity near-cardinal.** Replica: match sign rule OR add small dead-zone to look better.
- Render: 4 static directional sprites (NE/SE/SW/NW) per profession, flipX always false, no animation. **Nation tint** on white art (SetTint nation.Color). Sort: layer Default order 1, **sortPoint=Pivot → depth by Y (iso)**. Bubble order10, Carry order11. Scale 1. Damage flash red 0.18s.
- **State machine** (per profession): Civilian(wander/bush), Woodcutter(chop 5s→10), Stone/IronMiner(mine 5s→10, r80), Farmer(tend 10s), Hunter(hunt 5s→5, r80), Builder(construct 8s), Sawyer(saw 4s→4 planks, wood→planks), TreePlanter(plant 4s), Trader(carry 25, surplus>20/demand<5), Soldier/General(seek barracks→idle→moving→engaging). Node search radii tree50/miner80/bush60/fauna80. Deposit at town hall.

## 9. COMBAT / CONQUEST (Economy/Combat, Worker soldier FSM) — [AGENT 6 ✓]
- **ALL agents 100 HP** (soldier/worker/general; the SOLDIER_HP=150 constant is DEAD/unused). Damage **flat 15 vs any agent, 10 vs building** (no armor/variance/crit/ranged).
- Attack cadence **0.5s** (2 hits/s = 30 dmg/s to agents, 20 to buildings; ~4 hits kills an agent).
- Range: enter engage dist≤1.5 (sq 2.25), break if drifts >2.0 (sq 4.0) — hysteresis. Strafe perpendicular while engaging (repick CW/CCW 0.8–1.5s).
- Target (CombatTargeting, search radius 30, retarget 3s): nearest enemy **military within radius 8**, else nearest enemy **building**, else nearest **civilian**; anti-stacking claim penalty spreads soldiers. Only vs nations at war.
- Building HP: House/Farm/Lodge/Sawmill 100, **Barracks 200, TownHall 300 (INDESTRUCTIBLE — floors at 1 HP; ≤50 HP triggers 5s forced conquest dwell), Castle 500**.
- Mobilization: DesiredSoldiers; barracks cap ×12; SoldiersNeeded = war max(desired,8) (×attack-order 12) / peace barracks×4; BuildArmy intent cap 60. GeneralPromoter (1s): oldest soldier→General per barracks (1/barracks), the rally anchor.
- AttackOrder (1s, repath 5s): general-anchored rally (24 fixed offsets around general); target = **Manhattan-nearest enemy city** (tie: lowest id). Orphans rally to primary or march to own nearest.
- **ConquestSystem** (1s): ≥2 enemy military within Euclidean radius 5 of city.Center, continuous **15s dwell** → FlipOwnership (city+buildings+all tiles+members→attacker, TownHall HP reset 300, capital loss rebuilds castle at new Cities[0]); 30s post-flip grace.
- **WarVictorySystem** (1s): loser Cities==0 → war_won; loser Members<10 → war_won + **DissolveNation**; war age >300s → stalemate forced peace.
- Death: combat (Killer set) | starvation (Hunger≥100→hp0) | old age (Age≥lifetime). **Combat death PERMANENT (no respawn); natural death respawns** — war attrits pop toward the <10 dissolution condition.

## 10. HUD / CAMERA / MENU FLOW (UI/HUDController, Player/CameraController) — [AGENT 8 ✓]
- ★ **UICleanupBootstrap DISABLES the legacy IMGUI overlays** (NationStockpileOverlay, WarOverlay, GameSpeedHud, SettingsMenu, DecisionToastController). **Live HUD = HUDController (Canvas/TextMeshPro) + DeathNotification banner + PauseMenuController.** Do NOT replicate the IMGUI overlays as visible.
- Win95 palette (same as menus): face #C0C0C0, bevel white/#808080, navy #000080 title/header bars, body #DCDCDE, black text. Raised bevel (top/left white, bottom/right gray) for panels/buttons; sunken (inverted) for inputs/bodies. TMP default font. Sizes: bar 18, buttons 20, inspector title 20, body/side 17.
- Canvas: ScreenSpaceOverlay, sortingOrder 50, **ScaleWithScreenSize ref 1280×720 match 0.5**. All coords in 1280×720 space.
- **Top bar** h36 full-width: `"Year {tick/60}   ·   Tick {round(Time.time)}   ·   Source: {Mock/LLM/None}"` (+ queued msgs if any). 60 real s = 1 year.
- **Inspector** right column 360w, 8px from right, top 42 / bottom 70 (clears bars); navy title + [X] close; portrait box 80×80 (#282832) if sprite; body text 17. Describes Worker (name/nation/profession/activity/carrying/age/lifespan/hp/stage/family/royal/last-decision), Building (type/cell/city/nation/constructed), Nation (name/id/pop/cities/wars won-lost/leader/personality/last-decision/cities-list). Closed → "▸ Inspector" tab top-right (150×30 at -8,-42).
- **Bottom bar** h64: 4 speed buttons {Pause,1x,2x,4x}→timeScale {0,1,2,4}, 70×42 at x=12,87,162,237. Prompt bar (input + Send) between x=315 and inspector — **visible only in LLM mode**. Keys: Space toggle pause, 1→1×, 2→2×, 3→4× (when input not focused).
- **Left collapsible panels** (280w): "Active Wars" at (8,-36) [`• A vs B` / "(peace — no active wars)"]; "Recent Intents" at (8,-222) [feed max 6, `[Nation] intent  reason`]. Header click collapses (− / +).
- Refresh every 0.4s unscaled (works when paused).
- **Map click** (not over UI): ScreenToWorldPoint → worker within 0.5u → building within 1.5u → nation tile via TileNation. Opens inspector.
- **Camera** (CameraController): pan WASD/arrows W+y S−y D+x A−x, `pos += dir.normalized * panSpeed(20) * (size/10) * unscaledDt`; zoom scroll `size -= sign(scroll)*5*(size*0.05)`, clamp **[1.5, 250]** (initial 130 from scene, pos (0,120,-10)). Runs while paused. World extent ~**480×240 units** (WORLD_MAX_X 479, MAX_Y 239.5). transparencySortAxis (0,1,0).
- **DeathNotification** (ACTIVE): centered banner 500×80, dark-red (0.2,0.05,0.05,0.9), white bold 16, 5s: `"{prev} of {nation} has died ({cause}). {new} assumes leadership."`
- **Pause** (Esc via PauseMenuSpawner, timeScale 0/1): panel 320×340 "Day X • Population • Nations", Resume/Save Game/Load Game/Options/Quit to Menu; inline Save panel (5 slots) + inline Options panel (volume slider + Use LLM toggle + Apply/Back, built in code).
- **Menu flow**: MainMenu → NewGame(seed+randomize+LLM toggles)/LoadGame(5 slots)/Options → Game via GameSessionState static carrier.

## 11. DECISION AI / MOCK CASCADE (LLM/*) — [AGENT 7 ✓]
- **Per-nation decision** (DecisionScheduler): cadence **45s** (staggered by id), Mock no real cooldown / LLM 15s cooldown; ≤2 decisions/frame, 1/nation/frame. Frozen if source==None.
- Auto-emitted crisis events: **food_crash** (city food≤0, throttle 60s), **army_decimated** (soldiers < baseline×0.5). Plus event-driven wakeups (war_declared, succession, building_destroyed, consort_died).
- **MockHeuristic.Decide** (deterministic RNG = `worldSeed ^ nationId ^ tick`). Cascade **first-match-wins R0→R5**:
  - **R0 stalemate_peace**: at war + casualties<8 + war age>300s → MakePeace.
  - **R1 crisis** (RecentEvents within 30 ticks): war_declared vs me→BuildArmy(max(soldiers+8,enemyEst)); combat/army_decimated (>50% loss)→BuildArmy; building_destroyed→PrioritizeBuilding(that type) [Castle/TownHall→Idle]; city_lost→PrioritizeBuilding Barracks; food_crash→50% Farm / 50% FocusResource Food; first_contact→Observe/Idle; royal_death(non-leader)→Idle; mourning override→Idle.
  - **R2 player_msg** (LLM mode): keyword match (attack/war→DeclareWar, peace→MakePeace, defend/army→BuildArmy, found/settle→FoundCity, migrate→Migrate, name heir→NameHeir, surrender→Surrender) THEN **obedience roll by personality** (Aggressive 90/70, Cautious 30/50, Expansionist 60/75, Mercantile 40/60, Pious 50/65, Cunning 70/60, Stoic 40/50 war/nonwar) — fail = leader ignores, fall through.
  - **R3 war** (at war): losingBadly(<0.4×enemy)→Surrender(Cautious/Pious 30%)/BuildArmy; cityUnderAttack→AttackCity(Aggr/Cunning)/BuildArmy; CurrentAttackOrder→AttackCity; army≥80% desired→AttackCity enemy capital; soldiers<desired→Conscript(≤4); BuildArmy; NeedsMoreBarracks→PrioritizeBuilding Barracks; no army→MakePeace; casualties>5→Retreat 40%; else AttackCity nearest.
  - **R3.5 proactive_war**: not at war + no recent war(120t) + soldiers≥4 + no food-critical city → DeclareWar on weak neighbor (RelStrength<1.2), chance by personality (Aggressive 25, Cunning/Expansionist 15, Pious 8, else 0) + distance/strength modifiers.
  - **R4 succession**: no Heir → NameHeir(eldest non-heir Royal).
  - **R5 default_by_personality** (always returns): roll 0-100 thresholds (cumulative <):
    Aggressive war<15/city<25/build<50/focus<60; Cautious —/5/45/65; Expansionist war<5/40/65/75; Mercantile —/10/35/70; Pious war<5/10/30/40; Cunning war<10/25/45/60; Stoic —/5/30/40; else Idle. Plus peace-disband, migrate-surplus, floor-war 5%.
- **IntentType** (28; implemented: DeclareWar, MakePeace, BuildArmy, DisbandArmy, FoundCity, Migrate, ExpandTerritory, NameHeir, Disinherit, FocusResource, PrioritizeBuilding, Idle, AttackCity, Retreat, Conscript). Stubs=NotImplemented (ProposeTrade, Festival, ArrangeMarriage, Surrender, Observe, FormAlliance...). **Mock CAN emit Surrender/Observe but they're no-ops.**
- **IntentDispatcher.Apply** executes: DeclareWar→WarRegistry+ArmDefensively both; BuildArmy→DesiredSoldiers=max(cur,min(req,60)); AttackCity→CurrentAttackOrder; Conscript→convert ≤N adult male civilians→Soldier; FoundCity/Migrate/ExpandTerritory→bias into their systems; NameHeir/Disinherit→IsHeir; FocusResource/PrioritizeBuilding→nation fields. Sets LastLeaderIntent/Reason, appends "decision" event.
- **Mock is BOTH the default mode AND the universal fallback** when LLM throws/returns null. NationStateDoc = full serialized state (world/nation/leader/consort/heir/royalFamily/cities/neighbors/military/diplomacy/recentEvents/playerMessage). LLM POSTs to 127.0.0.1:8000. Default source Mock.

## 12. ECONOMY / BUILDINGS / RESOURCES (Economy/*) — [AGENT 5 ✓]
- **Resources**: Food, Wood, Stone, Iron, Gold, Planks. Per-CITY stockpiles (nation-level fallback). **Gold is INERT** (never produced/spent — don't implement a gold economy). Starting resources **0** everywhere except **+100 Wood** to newly-founded secondary cities.
- **Nodes** (ResourceNodeKind): Tree→Wood, Bush→Food, StoneNode→Stone, IronNode→Iron, Fauna→Food. Caps: Tree/palm 10, Bush 3, Stone 5, Iron 5, Fauna 5. **No regrowth** except TreePlanter replants Trees (cap 10). Single reservation per node.
- **Seeding** (deterministic worldSeed, once at ready): Pass A forest clusters (5–25 trees, centers ≥12 apart, not within Chebyshev-8 of capital); Pass B scatter grassland (Tree 3% / Bush 4% / Stone 4% / Iron 2%), beach Tree(palm) 4%; Pass C fauna 1.5% grassland. Node lookup = spatial hash bucket 32.
- **Buildings**: 15 enum but only **9 registered/functional**: Farm, HuntersLodge, Sawmill, House, HouseMedium, HouseLarge, Barracks, Castle, TownHall. (FishingHut/Smithy/Bakery/Brewery/Forge/Marketplace = unused placeholders — don't implement.)
  - Costs (Planks/Stone/Iron ONLY): Farm 8/4/0, Lodge 8/4/3, Sawmill 6/6/3, House 5/3/0, HouseMed 12/6/0, HouseLarge 25/12/0, Barracks 10/8/6; Castle & TownHall free.
  - Footprints: Castle 3×3, TownHall/Farm/Sawmill/Lodge 2×2, House 1×1, HouseMed 2×2, HouseLarge 3×3, Barracks 3×3.
  - HP: Castle 500, TownHall 300, Barracks 200, else 100 (TownHall indestructible, floors 1, ≤50→capture).
  - Biome: HuntersLodge grass/beach, others Grassland only; beach needs 3-tile water buffer; separation 1 Chebyshev (Barracks 0). Must be inside nation territory.
  - PerCityCap: Farm 6, Lodge 4, Sawmill 3, Barracks 2, Castle/TownHall 1, Houses ∞. Prereqs: House/Lodge need Farm, Barracks needs Castle.
- **Production** (executed by Worker AI, not building classes):
  - Woodcutter 10 Wood/5s (r50), Stone/IronMiner 10/5s (r80), Farmer 3 Food/10s tend (2/farm min 2 to operate), Hunter 5 Food/5s (r80 fauna), Sawyer **1 Wood→4 Planks**/4s, TreePlanter plant/4s (r30), Trader 25/trip (surplus>20, demand<5, priority Planks→Food→Wood→Stone→Iron), Builder construct 8s (2/blueprint). Bush 3 Food/3s. Deposit at TownHall/capital.
  - Housing: House 1 couple (2 adults), Medium 2, Large 4.
- **Growth system** (BuildingGrowthSystem, tick 5s, poorest-infra city first, ≤12 placements/city/tick): priority = Barracks(capital+war+0) → Farm/Lodge(food critical/insuff) → House(housing low) → Barracks(war) → Sawmill → Lodge(tier≥1) → House. Pooled cost across nation's cities; refund on fail. **No in-place upgrades** (bigger house = new blueprint). Blueprints built by Builders at 50% alpha.
- **Initial buildings/nation** (constructed at start): Castle (capital−(1,1)), TownHall, Farm, Sawmill (placed near most trees), Barracks.
- **Hunger** (HungerSystem, 30 workers/frame): +1/7s to 100; feed at ≥30 (Adult only) costs **1 Food**→hunger 0; die at 100 (0→100 = 700s). IsHungry flag >70. Children accrue but never fed.
- **Fauna** (FaunaWander): wander radius 5, speed 0.4, move 1–3s repick 3–6s; stops when reserved by hunter.

---
## ★ USER-COMPLAINT → FAITHFUL FIX MAP
1. **Fronteiras erradas/gigantes** → territory is a growing int-grid (disk r10 + 4-conn wood-costed expansion 5/tick), rendered as **translucent fill (alpha 0.40) on every owned cell + ONE solid directional border tile (NE/SE/SW/NW) on boundary cells** (2 layers). NOT a fixed nearest-capital circle, NOT a full-diamond strong tint.
2. **"De longe não renderiza bem / fica lento"** → iso 2:1 (64×32px tile), 480×480 world, camera ortho size 130 init clamp [1.5,250]; 3 stacked tilemaps; per-tile variant by position HASH (deterministic, no flicker); chunked. Replica needs viewport culling + deterministic tile variants.
3. **Flicker dos agentes** → direction is pure-sign quadrant with NO hysteresis + 2-phase update (decide 1/4 frames, integrate every frame, velocity persists). Fix replica: persist velocity, integrate by dt every frame, compute facing from retained velocity with a small dead-zone.
4. **WASD invertido** → W=+y up, S=−y down, D=+x right, A=−x left; pan scales size/10.
5. **Menus não batem** → exact Win95 layouts captured (MainMenu 426×413 gold logo + 4 buttons + taskbar; NewGame/LoadGame/Options/Pause). Palette + sizes in §5.
6. **Textos fora de posição** → HUD is Canvas 1280×720 ScaleWithScreenSize match 0.5; TopBar h36 / Inspector 360 right / BottomBar h64 speed buttons; left panels at (8,-36)/(8,-222). Legacy IMGUI overlays are DISABLED.

## ✅ ALL 8 SUBSYSTEM SCANS COMPLETE — full program mapped from canonical ZIP source.
