# agents.exe

**A faithful re-implementation, in [LDP3](https://github.com/jvpts11/LDP3), of the generative-agents
civilisation sandbox *agents.exe*.**

The original agents.exe is João Victor Pereira Tavares's undergraduate project — a Unity game where a
world of deterministic worker agents is led by a handful of *protagonists* whose strategy comes from a
fine-tuned small language model, with a deterministic **Mock heuristic** as the fallback. This project
rebuilds that game 1:1 in LDP3 — with **no engine underneath it**: the world simulation, the isometric
renderer, the agents, the economy, the nations and the UI are all built from scratch in the language,
partly as a stress-test that exercises LDP3 end to end. There is no LLM here; the Mock heuristic *is*
the strategic AI.

It is being rebuilt in phases against a reverse-engineering of the original's source. See
[`docs/BLUEPRINT.md`](docs/BLUEPRINT.md) for the plan and the faithful game model, and
[`docs/reference/`](docs/reference/) for the code-grounded specs of each subsystem.

> **Status: Phase 0 — reset.** The engine primitives (a GPU quad batcher, a pixel painter, vectors, a
> seeded RNG) are in place on the plugged-in [LDP3-OpenGL](https://github.com/jvpts11/ldp3-opengl)
> back end; the faithful game layer is being built on top, phase by phase (isometric world → agents →
> economy → nations & Mock heuristic → Win95 UI → shell → the parts the original left unfinished).

## The game, in one paragraph

A **480×480 tile** world — six elevation biomes from a Perlin/continent-mask heightmap — simulated on a
plain 2D grid but **rendered isometrically** (2:1 diamond tiles, depth-sorted). Several nations, each
led by a protagonist, grow populations of workers who forage, farm, chop, mine, build, haul and fight
under a hierarchical behaviour FSM. Every ~45 seconds each leader makes one strategic decision through
the Mock heuristic — declare war, found a city, prioritise a resource, name an heir — and the engine
turns it into orders its workers carry out. The player watches in a Windows-95-styled UI, controls
time (pause / 1× / 2× / 4×), and clicks any agent, building or nation to inspect it.

## Build & run

Needs the LDP3 toolchain (the sibling `../LDP3` dev build) with the sibling `../ldp3-opengl` library:

```
ldp3 build
build-output/AgentsExe.exe
```
