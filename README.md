# agents.exe

**A faithful re-implementation, in [LDP3](https://github.com/jvpts11/LDP3), of the generative-agents
civilisation sandbox *agents.exe*.**

The original agents.exe is João Victor Pereira Tavares's undergraduate project — a Unity game where a
world of deterministic worker agents is led by a handful of *protagonists* whose strategy comes from a
fine-tuned small language model, with a deterministic **Mock heuristic** as the fallback. This project
rebuilds that game in LDP3 — with **no engine underneath it**: the world simulation, the isometric
renderer, the agents, the economy, the nations and the UI are all built from scratch in the language,
partly as a stress-test that exercises LDP3 end to end. There is no LLM here; the Mock heuristic *is*
the strategic AI.

It is built in phases against a reverse-engineering of the original's source. See
[`docs/BLUEPRINT.md`](docs/BLUEPRINT.md) for the plan and the faithful game model, and
[`docs/reference/`](docs/reference/) for the code-grounded specs of each subsystem.

> **Status: playable.** The full core is in — isometric world, agents, economy, the strategic layer
> (Mock heuristic, war, conquest, succession), a Windows-95 HUD with a reflection-driven inspector, an
> interactive real-time shell, Worldbox god-powers, and nation territory. Ongoing polish: the menu
> flow and camera/rendering are being brought closer to the original.

## The game, in one paragraph

A **480×480 tile** world — six elevation biomes from a warped-Perlin / continent-mask heightmap —
simulated on a plain 2D grid but **rendered isometrically** (2:1 diamond tiles, depth-sorted, with
Mountain/Snow cliffs). Four nations, each led by a protagonist, grow populations of workers who forage,
farm, chop, mine, build, haul and fight under a behaviour FSM, tracking hunger, thirst and energy.
Every ~45 seconds each leader makes one strategic decision through the Mock heuristic — declare war,
seek peace, forge an alliance, raise an army, focus a resource, prioritise a building — and the realm
turns it into orders. Armies march and besiege; a broken realm is conquered and its survivors defect;
when a leader dies an heir takes the throne, sometimes with a new temperament. A Gold trade economy runs
alongside. The player watches in a Windows-95 UI, controls time, and clicks any worker or nation to
inspect it.

## Build & run

Needs the LDP3 toolchain (the sibling `../LDP3` dev build) and the sibling `../ldp3-opengl` library.
Asset and save paths resolve relative to the executable, so it runs from any location.

```
ldp3 build
build-output/AgentsExe.exe
```

## Controls

| Input | Action |
|-------|--------|
| **Enter** / click **New Game** | start a game (seed 42) |
| **WASD** / arrows | pan the camera |
| **scroll** / PgUp / PgDn | zoom |
| **Space** | pause / resume |
| **1 / 2 / 3** | speed 1× / 2× / 4× |
| **click** | select a worker or nation → inspector |
| **F5 / F9** | save / load (slot 1, seed-based) |
| **Esc** | main menu |
| **B / K** | conjure a worker / smite one, at the cursor |
| **G / Q / L** | drought / earthquake / flood (Worldbox god-powers) |

## Layout

- `src/world/` — grid, worldgen (noise), biomes, camera, flow-field pathfinding, resource seeding
- `src/people/` — nations and workers
- `src/economy/` — buildings, building types, stockpiles
- `src/sim/` — the Realm: the simulation container and its tick
- `src/strategy/` — the Mock heuristic, intents, personalities
- `src/gfx/` — the Batch2D quad batcher and the isometric renderers (world, territory, buildings, agents)
- `src/ui/` — the Windows-95 draw kit, HUD, font, reflection inspector
- `src/app/` — the interactive shell (Game), save/load, path resolution
- `assets/` — the sprite atlas (packed from the original's art)
