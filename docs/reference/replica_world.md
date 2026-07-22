# agents.exe — World & World-Generation Subsystem (faithful-replica spec)

Reverse-engineered from the shipped Unity C# source. Everything below is grounded in the
actual code under `unity-project/Assets/`. Paths are relative to that root unless absolute.

**Scope of this subsystem:** the tile grid, the height/terrain layer, the biome layer, the
procedural world-generation algorithm, on-map resources, and how the world is rendered
(isometric). Cross-cutting files outside `Scripts/World/` that are load-bearing for the world
model are included (Pathfinding, Economy resource seeding, Player camera, Agents rendering).

> **Two headline facts a re-implementer MUST get right (details in §1 and §6):**
> 1. **Rendering is ISOMETRIC, not top-down flat.** There are two coordinate systems: an
>    integer 2-D grid for *all simulation logic*, and a Unity **isometric** projection for
>    *rendering only*. Tiles are 2:1 diamonds (64×32 px footprint, 64×48 px art).
> 2. **The proposal's numbers are wrong.** It says "4×4 px tiles, ~240×135". The shipped code
>    uses a **480×480** grid of **64-px-wide isometric diamonds** (`WorldGenParams.Default`,
>    `WorldGenParams.cs:49-50`; PPU 64, `Terrain_Iso/*.png.meta`). There is **no** separate
>    terrain enum and **no** desert/tundra/forest biome — there is a single 6-value height-based
>    `Biome` enum, and "forest" is a *resource cluster*, not a biome.

---

## 0. File inventory

### `Scripts/World/` (11 .cs)
| File | Role |
|------|------|
| `Biome.cs` | The `Biome` enum (6 values). |
| `BiomeMapper.cs` | Height float → `Biome` via thresholds. |
| `HeightmapGenerator.cs` | The actual worldgen: warped fractal Perlin + continent masks. |
| `WorldGenParams.cs` | All tunables + `Default` preset (the shipped values). |
| `WorldBootstrap.cs` | Orchestrates generate→map→paint; owns `Biome[,]`; logging. |
| `BiomeRandomTile.cs` | `TileBase` that hash-picks a sprite variant per cell. |
| `TilemapPainter.cs` | Writes biomes onto the Unity isometric Tilemap incl. cliff/elevation stacking. |
| `TerritoryPainter.cs` | Paints nation territory fill + borders on two more tilemaps. |
| `TerritoryTile.cs` / `TerritoryTileSet.cs` | Territory `TileBase` + sprite set. |
| `WorldMapLogger.cs` | Dumps world/city/building events as JSONL (the game's data export). |

### World-related files elsewhere
- `Scripts/Pathfinding/PassabilityMap.cs` — biome → walkable bool grid.
- `Scripts/Economy/ResourceNodeSeeder.cs` — trees/ore/fauna placement by biome.
- `Scripts/Economy/ResourceNodeKind.cs`, `ResourceNode.cs` — node kinds & yields.
- `Scripts/Economy/BuildingBiomeRules.cs` — which buildings on which biome.
- `Scripts/Player/CameraController.cs` — orthographic pan/zoom + iso sort axis.
- `Scripts/Agents/Agent.cs`, `AgentRenderer.cs`, `Worker.cs`, `Direction.cs` — grid↔world for units.
- `Scripts/UI/GameSessionState.cs`, `UI/NewGameMenuController.cs` — seed/params handoff.
- Scene `Scenes/Game.unity` — the Grid + 3 Tilemap components (isometric settings).
- Tile assets: `Tiles/Biomes/*.asset`, `Tiles/Territory/*.asset`; art `Art/Tilesets/Terrain_Iso/*`.

### Python worldgen
`python-project/src/worldgen/` contains **only `.gitkeep`** — it is an empty stub. The entire
procedural pipeline is implemented in C# (`HeightmapGenerator.cs`). There is nothing to consume
from Python. (`ENVIRONMENT.md` mentions the folder but no code exists.)

---

## 1. ★ THE TWO COORDINATE SYSTEMS (read this first) ★

### 1a. Simulation grid space (integer cells)
All game logic works in an integer grid `Vector2Int (x, y)`, `x∈[0,W)`, `y∈[0,H)`,
origin at cell (0,0), **+x and +y increasing**. Everything is stored as `T[W, H]` 2-D arrays:
- Height: `float[,]` (`HeightmapGenerator.Generate`).
- Biome: `Biome[,]` (`BiomeMapper.Map`).
- Walkable: `bool[,]` (`PassabilityMap.ForGroundUnits`).
- Territory ownership: `int[,] _tileNation` (`NationManager.cs:22,66`), `-1` = unclaimed.

Indexing convention is uniformly `arr[x, y]` (column-major: `GetLength(0)=W=x`,
`GetLength(1)=H=y`), e.g. `BiomeMapper.cs:7-8,17`, `TilemapPainter.cs:23-24,36`.

### 1b. Isometric render space (Unity `Grid` component)
The world is drawn by a Unity **isometric** `Grid` (scene `Game.unity`, `Grid:` at line 1001):
```
m_CellSize:   {x: 1, y: 0.5, z: 0.25}
m_CellLayout: 3            # 3 = IsometricZAsY
m_CellSwizzle: 0
```
Three child Tilemaps hang off this Grid (see §6c). Sprites are imported at **PPU 64**
(`spritePixelsToUnits: 64`, `Terrain_Iso/*.png.meta`), so **1 world unit = 64 px**.

**There is NO hand-written iso math class** (no IsoUtils/GridToWorld/CoordinateSystem exists —
verified by search). All grid↔world conversion goes through Unity's isometric `Grid` API:
`grid.GetCellCenterWorld(Vector3Int)`, `grid.CellToWorld`, `grid.WorldToCell(Vector3)`.
A re-implementation must reproduce Unity's isometric transform explicitly.

### 1c. The grid → world (iso) transform  — EXACT
Given cell `(gx, gy, gz)` and `cellSize = (cw, ch, cz) = (1, 0.5, 0.25)`, Unity's
**Isometric / IsometricZAsY** `CellToWorld` produces world units:

```
worldX = (gx - gy) * cw * 0.5            = (gx - gy) * 0.5
worldY = (gx + gy) * ch * 0.5 + gz * cz  = (gx + gy) * 0.25 + gz * 0.25
```
- `gz` (the tilemap z-layer) is folded into world **Y** — that is what "IsometricZAsY" (layout 3)
  means, and it is how elevation/cliffs are raised visually (§6d). `cz = 0.25` per z-step.
- `GetCellCenterWorld` = `CellToWorld` + the cell-center offset (≈ `(0, ch*0.5) = (0, 0.25)`),
  i.e. the middle of the diamond. The code always uses `GetCellCenterWorld` to place objects
  (`ResourceNodeSeeder.cs:253`, `Worker.cs:340,548,1990`).

**Pixel form** (multiply units by PPU = 64; diamond footprint 64×32 px):
```
screenX_px = (gx - gy) * 32          # tileW/2 = 32
screenY_px = (gx + gy) * 16 + gz*16  # tileH/2 = 16 ; tileH(footprint)=32
```
This is the classic 2:1 isometric layout. (Camera transform/zoom then applies on top, §1f.)

Tile **art** is 64×48 px (`Terrain_Iso/*.png.meta`: sprite `rect width:64 height:48`), with
pivot `{x:0.5, y:0}` (bottom-center). The top diamond face is 64×32; the extra 16 px is the
cube's vertical side (for beach/mountain/snow depth). Draw at the diamond's bottom-center.

### 1d. The world → grid inverse (mouse-picking) — EXACT
Inverting the XY projection (ignore z / center offset), for world units `(wx, wy)`:
```
gx = floor(wx + 2*wy)
gy = floor(2*wy - wx)
```
(Derivation: `gx-gy = 2*wx`, `gx+gy = 4*wy`.) In code this is done via `grid.WorldToCell`:
mouse-pick flow in `HUDController.HandleMapClick` (`UI/HUDController.cs:133-210`):
```
screenPos = Mouse.current.position
world     = Camera.main.ScreenToWorldPoint(screenPos)      # 137
cell      = grid.WorldToCell(new Vector3(world.x, world.y, 0))  # 207
x = cell.x; y = cell.y                                      # 208
```
then bounds-checks against `NationManager.TileNation.GetLength(0/1)`. `Agent.CellPosition`
(`Agents/Agent.cs:35-44`) uses the same `LocalGrid.WorldToCell(transform.position)`.

### 1e. Agent grid → screen every frame
Agents do **not** store grid cells; they store a **world-space** `Vector2 _position`
(`Agent.cs:11`). `SyncTransform()` (`Agent.cs:113-116`) writes it straight to the transform:
```
transform.position = new Vector3(_position.x, _position.y, 0f)
```
Movement logic (`Worker.cs`) works like this:
1. Pick a target **cell** (Vector2Int) from flow-field/BFS in grid space.
2. Convert it to a world point: `_grid.GetCellCenterWorld(new Vector3Int(cell.x, cell.y, 0))`
   (`Worker.cs:340,548,1154,1990`, etc.).
3. Move `transform.position` toward that world point at `speed` (default 1.5 units/s,
   `Worker.cs:38`).
4. Re-derive the current cell when needed via `_grid.WorldToCell(transform.position)`
   (`Worker.cs:527,610,633,...`).

So the per-frame screen position **is** the iso-projected world position of the diamond center
(step 2 formula in §1c). A flat re-implementation must project the agent's continuous position
through the same iso transform, not draw it at `(x,y)` pixels.

Facing: `Direction` enum `{ NE=0, SE=1, SW=2, NW=3 }` (`Agents/Direction.cs`), chosen from the
world-space movement vector by `DirectionExtensions.FromMovement` (`Direction.cs:16-22`):
```
v.x>=0 && v.y>=0 -> NE ;  v.x>=0 && v.y<0 -> SE ;  v.x<0 && v.y<0 -> SW ;  else NW
```
`AgentRenderer` holds 4 sprites indexed by this enum (`AgentRenderer.cs:9,24-33`; sprite set
order is `ne, se, sw, nw`, `SetSprites` line 35-38). (A *separate* 8-way `Direction` enum exists
for the LLM layer, `LLM/Direction.cs` — not used for rendering.)

### 1f. Camera (screen ↔ iso/grid) — `Player/CameraController.cs`
- Orthographic 2-D camera. **Isometric depth sorting is set here** (`Awake`, lines 20-21):
  ```
  _cam.transparencySortMode = TransparencySortMode.CustomAxis;
  _cam.transparencySortAxis = new Vector3(0f, 1f, 0f);   # sort by world Y
  ```
  → among sprites of equal `sortingOrder`, **higher world Y draws behind** (further back).
- Pan: WASD / arrows move `transform.position` in world XY (line 30-38), speed scaled by zoom:
  `panSpeed(20) * (orthographicSize/10) * unscaledDeltaTime`.
- Zoom: mouse scroll changes `orthographicSize`, clamped **[1.5, 250]** (lines 44-49):
  `zoomDelta = sign(scroll) * zoomSpeed(5) * orthographicSize*0.05`.
- Screen→world for picking is plain `Camera.ScreenToWorldPoint` then `grid.WorldToCell` (§1d).

---

## 2. The tile grid

- **Dimensions (cells):** `Width × Height`. Shipped default **480 × 480**
  (`WorldGenParams.Default`, `WorldGenParams.cs:49-50`). Inspector range `[16, 4096]`
  (`WorldGenParams.cs:12-13`). The New-Game menu only overrides **Seed**, never W/H
  (`NewGameMenuController.OnStartClicked`, lines 28-33), so **480×480 is the actual runtime
  size**. (Proposal's 240×135 is not used anywhere.)
- **Pixel size per tile:** 64×48 px art, 64×32 px diamond footprint, PPU 64 (see §1c).
  Full map footprint in world units spans roughly x∈[−W·0.5, W·0.5], y∈[0, (W+H)·0.25].
- **Storage:** dense 2-D managed arrays `T[Width, Height]`, no chunking, no tile struct — a tile
  is just its index into parallel arrays (height `float`, biome `Biome`, walkable `bool`,
  owner `int`). The only per-cell object instances created are **resource nodes** and
  **buildings** (separate GameObjects), not tiles.
- **Coordinate system:** integer `(x,y)`, origin (0,0), `arr[x,y]`, as in §1a.
- **Chunking:** none in simulation. The render Tilemap uses Unity's internal chunk mode
  (`m_Mode: 0` = Chunk, `Game.unity`), invisible to logic.

---

## 3. Terrain / height layer

**There is no discrete "terrain type" enum** (no water/plains/low-hills/hills/mountain type).
The terrain layer is a continuous **normalized heightmap** `float[,]`, values in `[0,1]` after
normalization (`HeightmapGenerator.Generate`, `HeightmapGenerator.cs:57-64`). Height is then
collapsed into the `Biome` enum (§4); the game keeps the biome grid, not the float grid, at
runtime (`WorldBootstrap._biomes`, line 20-21; the heightmap is discarded after mapping).

**Traversal cost / passability is BINARY, not per-terrain-cost.** `PassabilityMap.ForGroundUnits`
(`Pathfinding/PassabilityMap.cs:8-22`) is the only passability rule:
```
walkable[x,y] = (biome == Grassland) || (biome == Beach);
```
So ground units can only stand on **Grassland** and **Beach**; **water (DeepOcean/Ocean),
Mountain, and Snow are impassable**. There is no weighted movement cost per terrain — pathing
is over the boolean grid (flow fields / BFS in the Agents/Pathfinding subsystem).

Visual **elevation** (distinct from passability) is a small per-biome int used only for
rendering cliffs (`TilemapPainter.elevationByBiome = {0,0,0,0,1,2}`, `TilemapPainter.cs:16`):
DeepOcean/Ocean/Beach/Grassland = 0, **Mountain = 1, Snow = 2**.

---

## 4. Biome layer

### 4a. The enum — `World/Biome.cs`
```csharp
public enum Biome {
    DeepOcean = 0,
    Ocean     = 1,
    Beach     = 2,
    Grassland = 3,
    Mountain  = 4,
    Snow      = 5,
}
```
Exactly **6 biomes**, ordered by ascending elevation. Note: **no desert, tundra, forest, or
"green plains vs forest" distinction exists.** The biome is a pure function of normalized height,
i.e. it is really an elevation band, not a climate biome. There is **no latitude/humidity model**.

### 4b. Height → biome mapping — `World/BiomeMapper.cs`
`BiomeMapper.Map(float[,] heightmap, WorldGenParams p)` (lines 5-27):
```
deepOceanCutoff = p.OceanThreshold * 0.6          // line 11
for each (x,y): v = heightmap[x,y]
  v <  deepOceanCutoff       -> DeepOcean
  v <  p.OceanThreshold      -> Ocean
  v <  p.BeachThreshold      -> Beach
  v <  p.MountainThreshold   -> Grassland
  v <  p.SnowThreshold       -> Mountain
  else                       -> Snow
```
With `Default` thresholds (`WorldGenParams.cs:69-72`): OceanThreshold **0.42**, BeachThreshold
**0.46**, MountainThreshold **0.62**, SnowThreshold **0.78**, so:
`DeepOcean [0,0.252) · Ocean [0.252,0.42) · Beach [0.42,0.46) · Grassland [0.46,0.62) ·
Mountain [0.62,0.78) · Snow [0.78,1]`.

### 4c. What the biome governs
The biome grid is the single authority consumed everywhere:
- **Passability** (§3): Grassland/Beach walkable only.
- **Resource placement** (§5): trees/bushes/ore on Grassland, palms on Beach, ore requires
  Mountain neighbours (see `ResourceNodeSeeder`).
- **Building placement** — `Economy/BuildingBiomeRules.cs`:
  ```
  HuntersLodge -> {Grassland, Beach}
  FishingHut   -> {Beach}
  everything else -> {Grassland}
  ```
  Enforced in `BuildingManager` (lines ~391-393, 557-614): rejects Ocean/DeepOcean/Mountain/Snow,
  special-cases Beach for water-adjacent buildings.
- **City founding / territory expansion** — only Grassland (and Beach for expansion) tiles count
  (`CityFoundingSystem.cs:116,214,401-403`; `TerritoryExpansionSystem.cs:136-137`).
- Representation per tile: a single `Biome` value in `Biome[,]`; no per-tile struct.

---

## 5. Resources on the map — `Economy/ResourceNodeSeeder.cs`

### 5a. Node kinds & yields
`Economy/ResourceNodeKind.cs`: `enum ResourceNodeKind { Tree, StoneNode, IronNode, Bush, Fauna }`.
`ResourceNode.Init` maps kind → harvested `ResourceType` (`ResourceNode.cs:38-42`):
```
Tree->Wood(cap 10)  Bush->Food(cap 3)  StoneNode->Stone(cap 5)
IronNode->Iron(cap 5)  Fauna->Food(cap 5, mobile)
```
Capacities are passed at spawn (`SpawnNode(kind, cell, capacity, sprite)`): Tree 10, Bush 3,
Stone 5, Iron 5, Fauna 5. Nodes are individual GameObjects with a `SpriteRenderer`
(sortingOrder 2, `spriteSortPoint = Pivot`, `ResourceNodeSeeder.cs:244-246`), positioned at
`grid.GetCellCenterWorld(cell)` (line 253). Fauna additionally gets a `FaunaWander` component
and `IsStationary=false` (lines 273, 284-285).

### 5b. Placement algorithm (seeded, deterministic)
Runs once when nations are ready (`OnNationsReady`, lines 63-234). RNG = `new System.Random(worldSeed)`
(same seed as worldgen, line 79-80), so placement is deterministic per seed.

Constants (lines 30-37): `CAPITAL_CLEAR_RADIUS=8`, `MOUNTAIN_RADIUS=3`, `FOREST_MIN_DIST=12`,
`CLUSTER_FILL_MAX=25`, `CLUSTER_MIN=5`.

**Pass 1 — forests (tree clusters):** (lines 86-154)
- Candidate cells = Grassland ∧ walkable ∧ not within `CAPITAL_CLEAR_RADIUS` (Chebyshev) of any
  nation capital (`NearAnyCapital`, lines 318-328).
- Shuffle candidates (Fisher–Yates, line 93/309-316). Greedily pick forest centers keeping every
  pair ≥ `FOREST_MIN_DIST` (12) apart (squared-distance test, lines 95-107).
- Each center grows a BFS blob of `rng.Next(5, 26)` trees over same-biome, walkable, unoccupied,
  non-duplicate cells (lines 109-154). Tree sprite alternates by parity:
  `(x+y)%2==0 ? treePine : treeOak` (line 130). Cluster cells recorded in `_formerForestTiles`.

**Pass 2 — scattered nodes on every remaining walkable, non-capital, unoccupied cell:**
(lines 156-208) One `rng.NextDouble()` roll per Grassland cell (not already in a cluster):
```
roll < 0.03                 -> Tree  (pine/oak by parity), cap 10     # 3%
roll < 0.07                 -> Bush,  cap 3                            # +4%
roll < 0.11                 -> StoneNode, cap 5                        # +4%
roll < 0.13                 -> IronNode,  cap 5                        # +2%
else                        -> nothing
```
Beach cells: `rng.NextDouble() < 0.04` → Tree with **palm** sprite, cap 10 (lines 199-206).

**Pass 3 — fauna:** (lines 210-231) On Grassland, walkable, non-capital, unoccupied, empty cells,
`rng.NextDouble() < 0.015` → `SpawnFauna` (mobile Food node) (lines 224-228).

Notes: ore is placed purely by the 4%/2% rolls on Grassland — the `HasMountainNeighbor` helper
(`MOUNTAIN_RADIUS=3`, lines 330-343) exists but is **not called** in the shipped seed loop (dead
code; a re-implementer can omit it). There is **no fish node** kind and no ocean resource; "fish"
economy comes from FishingHut buildings on Beach, not map nodes. Runtime tree replanting:
`PlantTreeAt`/`IsValidPlantCell` (lines 288-307) let workers regrow trees on valid Grassland.

Art (`Art/Props/`): `Tree_Pine.png`, `Tree_Oak.png`, `Tree_Palm.png`, `Bush.png`,
`Stone_Node.png`, `Iron_Node.png`, plus unused `Flowers.png`, `Oil_Seep.png`,
`RareElement_Outcrop.png` (no code references — future/cut content). Fauna art: `Art/Fauna/Iso/`.

---

## 6. World generation algorithm — `World/HeightmapGenerator.cs`

`HeightmapGenerator.Generate(WorldGenParams p)` returns `float[,] [W,H]`. Pipeline:

### 6a. RNG & offsets (lines 10-17)
`rng = new System.Random(p.Seed)`. Draw six large noise offsets via
`NextOffset = rng.NextDouble()*200000 - 100000` (line 149-152): base `(ox,oy)`, domain-warp
`(warpOx,warpOy)`, boundary `(boundaryOx,boundaryOy)`.

### 6b. Continent seeds (`PlaceContinentSeeds`, lines 87-128)
- `n = max(1, NContinents)` (default **4**); lay them on a `ceil(sqrt(n))×ceil(sqrt(n))` grid of
  normalized cells, each seed at cell-center + jitter `±0.3*cell` (lines 98-114).
- Plus `NIslands` (default **7**) seeds at random normalized positions in `[0.1, 0.9]`
  (lines 116-125).
- Each seed stores world-space center `(X,Y)` and radius `R`:
  continents `R = rand(ContinentRadiusMin..Max) * min(W,H)` (default 0.16..0.26),
  islands `R = rand(IslandRadiusMin..Max) * min(W,H)` (default 0.025..0.06).
  `Seed { float X, Y, R }` (line 86).

### 6c. Per-cell height (double loop, lines 25-56)
For each `(x,y)`:
1. **Domain warp** (fractal Perlin, 3 oct, persistence 0.5, lacunarity 2), scaled by
   `WarpScale`(110) and `WarpStrength`(18); second component offset by +5000 (lines 30-35):
   `wx, wy`.
2. **Base height** = `FractalNoise((x+ox+wx)/Scale, (y+oy+wy)/Scale, Octaves, Persistence,
   Lacunarity)` — default Octaves **6**, Persistence **0.5**, Lacunarity **2.0**, Scale **180**
   (lines 37-39).
3. **Boundary noise** = two extra fractal layers added together (lines 41-47):
   `bnLow` (2 oct, pers 0.6, lac 2, scale `BoundaryLowScale`=38, strength 0.45) +
   `bnHigh` (2 oct, pers 0.5, lac 2.4, scale `BoundaryHighScale`=12, strength 0.22, offset +3000).
4. **Continent mask** `mask = ContinentMaskValue(x,y,seeds,bn)` (lines 49, 130-147): for each
   seed, normalized distance `d = sqrt((dx/R)²+(dy/R)²) + boundaryOffset`; if `d<1`, smoothstep
   falloff `m = 1 - (3d² - 2d³)`, take the max over seeds. Then
   `v -= (1 - mask) * ContinentMaskStrength` (default **0.65**) — i.e. push ocean where outside
   all continents (line 50).
5. Track min/max.

### 6d. Normalize (lines 57-64)
Rescale the whole map to `[0,1]`: `map = (map - min)/(max - min)` (guarded by range>1e-6).

### 6e. `FractalNoise` (lines 69-84)
Standard multi-octave `Mathf.PerlinNoise`, remapped to `[-1,1]` per octave
(`Mathf.PerlinNoise(...)*2-1`), amplitude `*=persistence`, frequency `*=lacunarity`, normalized
by `maxAmp`. **This is Perlin (Unity `Mathf.PerlinNoise`), not Simplex.**

### 6f. What is / isn't implemented
- **Implemented:** multi-octave Perlin, domain warping, continent/island seed masks with
  smoothstep falloff, two-layer boundary perturbation, global normalization, threshold biomes.
- **NOT implemented (proposal mentions but code lacks):** tectonic plates, hydraulic/thermal
  erosion, rivers, latitude/temperature, humidity, rainfall, or any climate biome assignment.
  Biomes are pure elevation bands. No Simplex noise. Don't add these — replica must match.

### 6g. `WorldGenParams.Default` (the shipped preset) — `WorldGenParams.cs:46-73`
```
Seed=42  Width=480  Height=480
Octaves=6  Persistence=0.5  Lacunarity=2.0  Scale=180
WarpScale=110  WarpStrength=18
NContinents=4  NIslands=7  ContinentMaskStrength=0.65
ContinentRadiusMin=0.16  ContinentRadiusMax=0.26
IslandRadiusMin=0.025    IslandRadiusMax=0.06
BoundaryLowScale=38  BoundaryLowStrength=0.45
BoundaryHighScale=12 BoundaryHighStrength=0.22
OceanThreshold=0.42  BeachThreshold=0.46  MountainThreshold=0.62  SnowThreshold=0.78
```
Seed handoff: menu → `GameSessionState.PendingWorldGen` (`GameSessionState.cs:10`) →
`WorldBootstrap.Awake` copies it (`WorldBootstrap.cs:32`); `MenuOverrideActive` gates the
inspector seed-override logic (lines 49-55).

### 6h. Determinism requirement
Exact replication requires matching **Unity's `Mathf.PerlinNoise`** and **.NET `System.Random`**
bit-for-bit, plus the exact draw order (6 offsets, then continent jitters, then island positions).
If the noise/RNG implementations differ, the same seed yields a different map. If bit-exactness
isn't feasible, document that seeds won't match across implementations.

---

## 7. Rendering of the world (isometric)

### 7a. The three Tilemaps — `Scenes/Game.unity`
All parented to the isometric `Grid` (§1b). Sorting orders (from scene, `m_SortingOrder`):
| GameObject | m_SortingOrder | Painted by |
|-----------|----------------|-----------|
| `Tilemap_Terrain` | 0 (base) | `TilemapPainter` |
| `Tilemap_Territory_Fill` | 1 | `TerritoryPainter` |
| `Tilemap_Territory_Border` | 2 | `TerritoryPainter` |

All three `TilemapRenderer`s use `m_Mode: 0` (Chunk) and `m_SortOrder: 3` (= **TopRight**, the
correct tile draw order for this isometric layout). Resource-node/agent SpriteRenderers live on
the Default layer at sortingOrder 2 / 1 respectively and inter-sort with the world by the
camera's CustomAxis-Y rule (§1f).

### 7b. Biome → tile (terrain) — `TilemapPainter.Paint(Biome[,])`
Serialized tile arrays, one entry per biome index (`TilemapPainter.cs:11-14`):
`flatTilesByBiome[6]`, `cliffSETilesByBiome[6]`, `cliffSWTilesByBiome[6]`, `cliffBothTilesByBiome[6]`.
Loop (lines 32-64):
- Base (flat) layer at **z=0**: tile = `flatTilesByBiome[idx]` (for elevated biomes it draws the
  Grassland flat underneath, `z0Idx = elev==0 ? idx : grasslandIdx`, line 44).
- **Cliff / elevation stacking** (the reason for IsometricZAsY z-layers): if the biome's
  `elevationByBiome[idx] ≥ 1`, stack an extra tile at **z=1** (mountain body); if `≥ 2`, another at
  **z=2** (lines 48-62). Each stacked tile is raised `0.25` world-Y per z (§1c) to read as height.
- **Cliff variant selection** (`ChooseVariant`, lines 68-76): compares this cell's elevation to its
  SE neighbour (`biomes[x, y-1]`) and SW neighbour (`biomes[x-1, y]`) (lines 39-42). If the
  neighbour is lower on that side, use the SE-facing / SW-facing / both cliff sprite; else flat.
  This is what produces the drawn cliff faces on the SE/SW edges of mountains & snow.
- Batched into `tilemap.SetTiles(positions[], tiles[])` (line 65).

### 7c. `BiomeRandomTile` (per-cell sprite variety) — `World/BiomeRandomTile.cs`
Each biome tile asset is a `BiomeRandomTile : TileBase` holding a `Sprite[] sprites` + `fallback`
(`Tiles/Biomes/*.asset`; e.g. `Grassland.asset` has **5** sprite variants). In `GetTileData`
(lines 14-30) it deterministically hash-picks a variant from the cell position:
```
h   = unchecked((position.x * 73856093) ^ (position.y * 19349663));   // line 20
idx = ((h % len) + len) % len;                                        // line 21 (safe modulo)
sprite = sprites[idx];
tileData.color = Color.white;  flags = LockTransform;  collider = None;
```
So terrain color comes entirely from the **sprite art** (tinted white), not code colors. The
matching art lives in `Art/Tilesets/Terrain_Iso/` with 5 variants each:
`Grassland_1..5`, `Ocean_1..5`, `Deep_Ocean_1..5`, `Beach_1..5`, `Mountain_1..5`, `Snow_1..5`,
plus cliff sets `Mountain_Cliff[_SE|_SW|_Both]_1..5` and `Snow_Cliff[...]_1..5`. Tile assets:
`Tiles/Biomes/{DeepOcean,Ocean,Beach,Grassland,Mountain,Snow}.asset` and the cliff variants
`Mountain_Cliff_{SE,SW,Both}.asset`, `Snow_Cliff_{SE,SW,Both}.asset`.

### 7d. Territory rendering — `TerritoryPainter` + `TerritoryTile`
Separate from terrain. Reads `NationManager.TileNation` (`int[,]`, `-1`=unowned). For each owned
cell it writes a **fill** tile (`Tiles/Territory/TerritoryTile_Fill`) on `Tilemap_Territory_Fill`
and, on edges, a directional **border** tile (NE/SE/SW/NW) on `Tilemap_Territory_Border`
(`TerritoryPainter.cs:64-97`). Border chosen by comparing the 4-neighbour owner / city
(`PickBorder`/`IsBorder`, lines 99-121). Fill color set per-cell via
`fillTilemap.SetColor(pos, NationPalette.GetTerritoryFillColor(nationId))` (line 95).
`TerritoryTile.GetTileData` tints the sprite by `nation.Color` (`TerritoryTile.cs:23-24`).
`TerritoryTileSet` holds `Fill/BorderNE/SE/SW/NW` sprites (`Art/Tilesets/Territory/`).
Repaints on `NationManager.OnNationsReady` / `OnTerritoryChanged` (lines 31-35).

### 7e. GPU instancing / atlas
No custom shader or GPU instancing — standard Unity 2-D sprite Tilemap rendering (chunk-batched).
Sprites are individual PNGs (single-sprite mode `spriteMode:1`), not a packed atlas in-project
(Unity may atlas at build). Re-implementation should treat each terrain cell as: pick biome →
pick variant by the hash → blit the 64×48 diamond at the iso-projected bottom-center, with cliff
overlays stacked for Mountain/Snow, then territory fill/border tints on top.

---

## 8. World data export — `World/WorldMapLogger.cs`

Not gameplay, but it is the canonical serialized view of the world (useful to validate a replica).
Writes JSONL to `<project>/../Logs/world_map_run_<timestamp>.jsonl` (lines 285-300). First line is
the world dump (`DumpWorld`, lines 119-158):
```json
{"type":"world","tick":<int>,"width":W,"height":H,"seed":S,
 "biome_enum":{"0":"DeepOcean","1":"Ocean","2":"Beach","3":"Grassland","4":"Mountain","5":"Snow"},
 "rows":["<H strings of W digits>", ...]}
```
Each row is a string of `W` digit-chars, `char('0'+biomeIndex)`, one row per `y` (lines 140-152).
Subsequent lines: `city`, `building_placed/constructed/destroyed`, `city_conquered` events with
`cell`/`center`/`footprint` in **grid coords**. This confirms the grid is the source of truth and
`(x,y)` cells are the serialized unit.

---

## 9. Re-implementation checklist (world subsystem)

1. **Two coord systems.** Keep all logic in an integer 480×480 grid; project to a **2:1 isometric
   diamond** only for rendering. Use the exact transforms in §1c/§1d.
   `worldX=(gx-gy)*0.5, worldY=(gx+gy)*0.25 (+gz*0.25)` in world units (×64 for px).
2. **Iso depth sort** by world Y (CustomAxis (0,1,0)) plus per-layer sortingOrder
   (terrain 0 < fill 1 < border 2 < nodes/agents), sprite pivot at bottom-center.
3. **Elevation z-stacking** for Mountain(1)/Snow(2) with SE/SW cliff variant selection.
4. **6 elevation-band biomes only** (no climate biomes); exact thresholds 0.42/0.46/0.62/0.78 with
   the DeepOcean = 0.6×Ocean sub-cutoff.
5. **Binary passability** (Grassland/Beach only); no per-terrain movement cost.
6. **Worldgen** = warped multi-octave **Perlin** + continent/island smoothstep masks + 2-layer
   boundary noise + normalize. No plates/erosion/rivers/climate. Match `Default` params in §6g.
7. **Resources** seeded from the same seed with the exact probabilities/clustering in §5.
8. Determinism hinges on matching `Mathf.PerlinNoise` + `System.Random` and their draw order (§6h).

### ★ The single most important thing
**The world is an isometric projection of a plain integer grid, not a flat top-down map.** Every
tile is a 2:1 diamond (64×32 px footprint / 64×48 px art, PPU 64) placed via
`worldX=(gx−gy)·0.5, worldY=(gx+gy)·0.25` world units (grid→iso), with the inverse
`gx=⌊wx+2·wy⌋, gy=⌊2·wy−wx⌋` for mouse-picking, and draw-order sorted by world-Y. Get this
grid↔iso transform (and the bottom-center pivot + Y-depth sort + Mountain/Snow z-stacking) right
and everything else — cliffs, agents, resources, territory, clicking — lines up; get it wrong and
the whole map renders as a wrong-shaped flat grid (the previous replica's mistake).
