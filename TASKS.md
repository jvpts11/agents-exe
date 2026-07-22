# agents.exe (LDP3) — Rebuild Task Tracker

Durable checklist for the faithful 1:1 rebuild. Plan + faithful model: `docs/BLUEPRINT.md`.
Code-grounded reference specs: `docs/reference/replica_*.md`. Check items off as they land; keep this
current so progress survives across sessions.

Legend: `[x]` done · `[~]` in progress · `[ ]` todo.

---

## Phase 0 — Reset ✅
- [x] Strip the wrong game layer; keep engine primitives (Vec2, Rng, Batch2D, Painter/GlPainter/SoftPainter)
- [x] Blueprint + 6 reference specs committed; 1280×720 window skeleton builds (commit 0493991)

## Phase 1 — Isometric world
- [ ] `enum Biome` (Java-style: DeepOcean/Ocean/Beach/Grassland/Mountain/Snow + color + walkable + threshold)
- [ ] `struct Tile` + the 480×480 grid (flat array, arr[x,y]) with biome/walkable/territory
- [ ] Worldgen: warped multi-octave Perlin + continent/island smoothstep masks + boundary noise → heightmap
- [ ] BiomeMapper: heightmap → biome via thresholds 0.42/0.46/0.62/0.78
- [ ] Iso coord type + transform (grid→world `((gx-gy)*0.5,(gx+gy)*0.25)`, inverse for picking), `comptime` consts
- [ ] Isometric tile render: 64×32 diamonds, bottom-center pivot, **depth-sort by world-Y**, Mountain/Snow z-stack
- [ ] Resource nodes (Tree/Bush/Stone/Iron/Fauna) seeded on Grassland/Beach; forests = BFS clusters; render
- [ ] Camera: pan (WASD/arrows) + zoom (scroll), grid↔screen mapping
- [ ] Verify: GL readback → PPM → PNG reads as the original's isometric map

## Phase 2 — Agents
- [ ] Worker model (HP/Direction/nationId/Lifetime/Hunger/Profession/Role/Family/Carrying, Stage)
- [ ] Hierarchical FSM (Dead→not-ready→Child→job); harvest & soldier state machines
- [ ] Hunger survival (+1/7s, death@100, eat@≥30); old-age lifetime roll
- [ ] 12 professions + scored selector under building caps + demotion grace + hysteresis
- [ ] 4-direction iso sprites + per-nation tint + overlays; object pooling
- [ ] 3 tick cadences (AI 1/4 staggered, hunger 30/frame, movement every frame)
- [ ] Flow-field pathfinding (BFS) + movement integration + wall-slide; children random-walk
- [ ] Reproduction (housing-gated)

## Phase 3 — Economy
- [ ] Resources (Food/Wood/Stone/Iron/Gold/Planks) + per-city stockpiles
- [ ] `catalog` for BuildingRegistry + ProfessionRegistry/Capacity (code-as-data)
- [ ] 9 buildable buildings (costs/caps/footprints/HP/prereqs) + production/refinement (Wood→Planks)
- [ ] BuildingGrowthSystem (5s, priority ladder, ≤12/tick, 8s build, ≤2 builders)
- [ ] Construction blueprints + builder FSM; secondary-city seeding

## Phase 4 — Nations & Mock heuristic
- [ ] Nation model (capital/cities/stockpiles/members/relations/personality/war)
- [ ] Protagonist (Worker + Role + Personality[7]); 28 IntentTypes; Intent payload (union/record)
- [ ] MockHeuristic cascade R0..R5 (exact thresholds from replica_nations.md) + DecisionScheduler (45s)
- [ ] IntentDispatcher (exhaustive match) → worker orders
- [ ] Succession (heir / auto-promote elite); War (WarRegistry + victory rules); conquest

## Phase 5 — UI & Inspector
- [ ] Win95 draw kit (1×1 tint + bevels) on Batch2D; palette + font
- [ ] HUD: top bar (Year/Tick/Source/queued), Wars panel, Recent-Intents panel
- [ ] Inspector (360px, click worker/building/nation) — via reflection over fields
- [ ] Speed controls (Pause/1x/2x/4x) + keyboard (Space, 1/2/3)

## Phase 6 — Shell
- [ ] Real-time scaled clock (tick=round(scaled time), year=tick/60, timeScale 0/1/2/4)
- [ ] Main Menu, New Game (seed 42, toggles), Load (5 slots+DEL), Options, Pause overlay
- [ ] Save/load (seed-based, save_v1, 5 slots)
- [ ] Bootstrap: pre-place 4 nations, 50 workers + royal couple, seed 42

## Phase 7 — Complete what the original left unfinished
- [ ] Worldbox god-powers: paint terrain/biome, add/remove resources, spawn/kill, events (drought/flood/quake) + event stream
- [ ] Full worldgen: tectonic plates, hydraulic erosion, rivers, latitude/humidity biomes, separate terrain+biome layers
- [ ] Survival: thirst + energy/sleep; fuller action vocab (swim/flee/follow/drink/rest/patrol) + fish node
- [ ] Economy: the 6 extra buildings (FishingHut/Smithy/Bakery/Brewery/Forge/Marketplace) + a real Gold economy
- [ ] Diplomacy: stubbed intents (Surrender/Observe/alliance/tribute/trade/ambassador)
- [ ] Nation-origin modes (pre-placement vs organic emergence); full-state save

---

### Working notes (update as you go)
- Repo: `github.com/jvpts11/agents-exe`; commit identity jvpts11; build under vcvars64 (`ldp3 build`).
- Verify the GL path with readPixels→PPM→PNG (env has real GL 4.6). `open()` returns non-zero on success.
- No inline `{}` blocks in LDP3; multi-line always.
