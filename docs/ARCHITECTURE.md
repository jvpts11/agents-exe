# Architecture

agents.exe is a real-time civilisation sandbox. Seven autonomous nations grow, farm, build, sail,
colonise and wage war on a procedurally generated archipelago, painted every frame. This document
describes how it is put together and — because the game exists partly to prove out
[LDP3](https://github.com/jvpts11/LDP3) — which language features each part exercises.

## The shape of a tick

Everything advances off one clock. `Realm.advance()` runs, in order:

1. **Each nation ticks** (`tickNation`): its people run their tactical FSMs, its economy produces,
   labour is reassigned, births and deaths are resolved, colonies may be founded, the dead are
   reaped from the roster.
2. **Combat** (`combat` → `siege`): armies within reach of an enemy capital batter its town hall and
   trade casualties.
3. **Wars resolve** (`resolveWars`): a nation whose town hall falls, or whose people are gone, is
   dissolved.
4. **Strategy** (`runDecisions`): any nation whose decision timer is up produces one `Intent` from
   its Leader and applies it.

The whole simulation is deterministic from the world seed: same seed, same fifteen-year history.

## Two decision layers

The original game seated a fine-tuned language model in the Leader's chair, falling back to a
deterministic rule cascade whenever the model failed. **This build has no model** — it always runs
the cascade, so every choice is reproducible.

### Tactical — `Agents.People` + the FSMs in `Realm`

Every person is a `Worker`: a small finite-state machine with a job, a position, a carry slot and a
few timers. `Realm.tickWorker` routes each to its behaviour:

- **Gatherers** (woodcutter, stone/iron miner, hunter) walk a cached flow-field to the nearest
  *reachable* resource node, harvest it, and carry the yield home.
- **Farmers** tend the nation's fields; **fishers** walk to the shore and carry back a catch;
  **traders** work the marketplace; **tree-planters** reforest around the capital.
- **Builders** raise blueprints into buildings.
- **Soldiers and generals** march on an enemy capital along *its* flow-field and lay siege.
- **Boat crews** cross open water — which walls off every land unit — to plant a colony.

Movement is by **flow-field**, not per-agent A*: one unweighted BFS floods out from each capital
(`FlowField`), and agents just step downhill. An attacker reuses the *enemy's* field to march on the
enemy seat; a gatherer skips any node its home field cannot reach, so no one sets out for a target
across a mountain lake or the sea and stalls there.

### Strategic — `Agents.Strategy`

Every ~45 ticks, a nation's Leader produces one `Intent` through `MockHeuristic.decide` — a cascade
of rules, seeded per call from the world seed so it stays reproducible:

- **R0** stalemate peace, **R1** crisis (famine, heavy losses), **R3** war conduct, **R3.5**
  proactive war on a weak *reachable* neighbour, **R4** name an heir, **R5** a personality-weighted
  default (the Aggressive muster armies, the Expansionist found cities, the Mercantile tend trade).

`IntentDispatcher.apply` turns the `Intent` into state the tactical layer reads: `desiredSoldiers`,
`focusedRes`, `prioBuilding`, `attackTarget`, war declarations. The two layers never call each
other directly — strategy sets knobs, tactics obey them.

## The systems

| System | Where | What it does |
|---|---|---|
| World generation | `World.Grid` | radial height blobs → biomes (ocean/beach/grass/mountain/snow), all integer, deterministic |
| Pathfinding | `World.FlowField` | one BFS flood per city, reused by every agent heading there |
| Resources | `World.ResourceNode` | trees, stone, iron, bush, fauna scattered on the land; reserved while harvested |
| Economy | `Economy.*` | six goods, twelve building types with costs/caps/housing, passive + worker-driven production |
| Population | `People.Nation` | the roster, the royal line, war state, era and knowledge |
| Combat | `Realm.siege` | soldiers near an enemy hall drain its HP and trade blows; a fallen hall dissolves the nation |
| Colonisation | `Realm.tickBoat` | boats cross the sea and found a second seat on a far shore |
| Technology | `Realm.advanceTech` | knowledge accrues from people, works and wealth; crossing a threshold advances the era and boosts output |

## Rendering — `Agents.Gfx`

Drawing goes through one interface so the CPU and GPU share a single scene:

```
Painter (interface: fillRect in 0-255 RGB)
├── GlPainter   → Batch2D → LDP3-OpenGL   (the real front end, one draw call per frame)
└── SoftPainter → a byte buffer → PPM      (a headless CPU rasteriser, for verification)
```

`Scene.paint(Painter, Realm)` draws the whole game once — terrain with per-cell noise shading, the
harvestable nodes, the ruins of fallen realms, each nation's buildings and people, and a HUD of
population, army and era. `main` opens a GL window and runs the live loop through `GlPainter`; where
no GL context exists it falls back to `SoftPainter` and writes frames to disk — the same picture,
proof the scene is right without a screen.

## Layout

One type per file, grouped into folders, each folder a namespace under the `Agents` bundle:

```
src/
├── main.ldp3            App        — entry point, the GL / software loop
├── math/     Vec2       Math       — value-type maths
├── world/    Rng Biomes Grid ResourceNode FlowField
├── economy/  Res Stockpile Bld Building
├── people/   Job Trait WState Worker Nation
├── strategy/ IType Intent MockHeuristic IntentDispatcher
├── sim/      Realm       — the orchestrator that runs every system
└── gfx/      Painter SoftPainter GlPainter Batch2D Scene
```

## What it exercises in LDP3

The game is a stress test of the language across levels:

- **OOP core** — classes, single inheritance, an `interface` (`Painter`) with an `override`,
  polymorphic dispatch (one `Scene` drawn through either back end), destructors that free owned
  memory (`Grid`, `FlowField`, `SoftPainter`).
- **Value semantics & memory** — `on stack` / `on heap` placement, `T*` to share, `delete`, manual
  ownership of every allocation with no leaks over a long run.
- **Generics** — `ArrayList<T>` throughout, over both value and pointer element types.
- **Enums-as-ordinals** — jobs, buildings, biomes and intent types.
- **String interpolation** — `$"{n.name} enters the {n.eraName()} Age!"` for the chronicle.
- **The plug system** — LDP3-OpenGL is a separate library, plugged in for the graphics, imported by
  its own `opengl.*` bundle path.
- **Low-level access** — `Batch2D` builds its vertex buffer through `System.Memory` (`address`,
  `Memory.write<float>`), the same primitives a systems language needs, inside an OOP program.
- **The standard library** — `String`, `ArrayList`, `File` (PPM output), `Console`.
