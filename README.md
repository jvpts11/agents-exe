# agents.exe — in pure LDP3

A headless reimplementation of **agents.exe** — a real-time god-game / civilisation sandbox — written
entirely in [LDP3](https://github.com/jvpts11/LDP3). Several AI-run nations grow, farm, build, raise
armies, wage war and conquer one another on a procedurally generated island — all autonomously, all
printed to the terminal as ASCII. **The LLM is deliberately left out**: where the original game asked a
fine-tuned Phi-4-mini to pick each nation's strategy, this build always routes to the deterministic
`MockHeuristic` rule cascade — the same scripted oracle agents.exe itself falls back to when the model
is unreachable.

```
map legend:  ~ ocean   . beach   (space) grass   ^ mountain   * snow
             1-4 a nation's people    H TownHall  F Farm  W saWmill  h House  B Barracks  ? blueprint
```

## The architecture (mirrors the original)

The simulation has **two decision layers**, exactly as agents.exe does:

1. **Tactical layer** — every person is a `Worker` finite-state machine (woodcutter, farmer, miner,
   builder, sawyer, soldier). Workers move by **flow-field** (a BFS flood-fill from the target over the
   walkable grid — no A* per agent), gather resources into the city stockpile, raise blueprints, and
   march to war. Driven by `Realm.tickWorker`. *This layer never touched the LLM.*
2. **Strategic layer** — every ~45 ticks each nation's **Leader** produces exactly one `Intent`
   (DeclareWar, MakePeace, BuildArmy, FoundCity, FocusResource, PrioritizeBuilding, AttackCity,
   Conscript, NameHeir, Retreat, Idle). This is the seat the LLM occupied. Here it is filled by
   `MockHeuristic.decide` — a faithful port of the original's **R0..R5 priority cascade** (stalemate
   peace → crisis response → war conduct → proactive war → succession → a personality-weighted
   default). `IntentDispatcher.apply` writes the Intent back onto the world.

Supporting systems, all self-scheduled off a single tick like the original: economy production
(farms → food, sawmills → planks), building growth, labour re-allocation, combat and dwell-based
conquest, nation dissolution, and a `SuccessionSystem` (heir → royal → raised commoner) so a realm
stays governed when its leader dies of old age.

## Source layout (`src/`)

| File | What lives there |
|---|---|
| `world.ldp3` | `Rng` (deterministic LCG), `Biomes`, `Grid` (continent-seed world generation) |
| `flow.ldp3` | `FlowField` — the BFS flow-field pathfinder |
| `economy.ldp3` | `Res`, `Stockpile`, `Bld`, `Building` |
| `nations.ldp3` | `Job`, `Trait`, `WState`, `Worker`, `Nation` |
| `strategy.ldp3` | `IType`, `Intent`, **`MockHeuristic`** (the scripted brain), `IntentDispatcher` |
| `sim.ldp3` | `ResourceNode`, **`Realm`** — the orchestrator: setup, tactical FSM, economy, growth, combat, the decision scheduler |
| `main.ldp3` | `Renderer` (ASCII map + dashboard + event feed) and `Main` |

## Build & run

Needs the LDP3 toolchain (the sibling `../LDP3` dev build, or an installed `ldp3` on `PATH`):

```
ldp3 build
build-output/AgentsExe.exe
```

It generates an island, seats four nations (each with a distinct personality — Aggressive, Cautious,
Expansionist, Mercantile…), then simulates ~15 in-game years, painting the world and a per-nation
dashboard every so often. Nations that lose their capital fall; the run ends early if one nation is
left standing.

## What is faithful, and what is simplified

Faithful: the two-layer decision model, the flow-field pathfinding, the resource/economy loop,
building costs and caps, the MockHeuristic rule cascade and its personality roll tables, war →
conquest → dissolution, and royal succession.

Simplified for a headless build: one city per nation (satellite cities are recorded, not seated);
combat is resolved by proximity-to-capital attrition rather than per-tile duels; there are no births
(population is held stable by replacing the naturally deceased); and the world is smaller than the
original 480×480. None of the runtime depends on the LLM, by design.
