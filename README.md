# agents.exe

**The first game written in [LDP3](https://github.com/jvpts11/LDP3).**

agents.exe is a real-time civilisation sandbox: several AI-run nations grow, farm, build, raise
armies, wage war and conquer one another on a procedurally generated island — all autonomously,
painted to the terminal. It began as João Victor Pereira Tavares's undergraduate project; this is
that world brought to life as the first full game implemented in LDP3, and it doubles as a proving
ground that exercises the language end to end — enums, records, generics, `match`, operator
overloading, regions and ownership, `async`, `Result`/`Option`, reflection, persistents and more.

```
map legend:  ~ ocean   . beach   (space) grass   ^ mountain   * snow
             1-4 a nation's people    H TownHall  F Farm  W saWmill  h House  B Barracks  ? blueprint
```

## Architecture (two decision layers)

1. **Tactical layer** — every person is a `Worker` finite-state machine (woodcutter, farmer, miner,
   builder, sawyer, soldier) that moves by a BFS flow-field and runs the economy.
2. **Strategic layer** — every ~45 ticks each nation's Leader emits one `Intent` (DeclareWar,
   BuildArmy, FoundCity, FocusResource, PrioritizeBuilding, AttackCity, NameHeir, …) through a
   scripted decision cascade; `IntentDispatcher` applies it, nudging the tactical layer.

Supporting systems run off a single tick: world generation, resource gathering, building growth,
combat and dwell-based conquest, royal succession, and nation dissolution.

## Layout

The code is organised as a flagship — one type per file, grouped into folders, each folder its own
namespace under the `Agents` bundle:

| Namespace | Folder | Holds |
|---|---|---|
| `Agents.Math` | `src/math/` | value-type maths (`Vec2`) |
| `Agents.World` | `src/world/` | world generation, biomes, the grid, resource nodes, flow-field |
| `Agents.Economy` | `src/economy/` | resources, stockpiles, buildings |
| `Agents.People` | `src/people/` | workers, professions, nations, the royal line |
| `Agents.Strategy` | `src/strategy/` | intents and the Leader's decision cascade |
| `Agents.Sim` | `src/sim/` | the `Realm` orchestrator that runs every system |
| `Agents.App` | `src/app/` | the terminal renderer and entry point |

## Build & run

Needs the LDP3 toolchain (the sibling `../LDP3` dev build, or an installed `ldp3` on `PATH`):

```
ldp3 build
build-output/AgentsExe.exe
```

It generates an island, seats four nations of distinct personality (Aggressive, Cautious,
Expansionist, Mercantile…), and simulates ~15 in-game years, painting the world and a per-nation
dashboard as it goes.
