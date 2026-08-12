# agents.exe

**A god-game built from nothing, in [LDP3](https://github.com/jvpts11/LDP3): a world that is
generated, populated and then left to get on with it, while somebody watches.**

Everything here is written in LDP3 — the worldgen, the 3D renderer, the simulation — with **no engine
underneath it**. The only thing borrowed is OpenGL itself, through a sibling library that is also
LDP3. That is half the point: the game is the language's stress test, and every hole it walks into
gets fixed *in the compiler* rather than worked around in the game.

> **Status: the ground and the fauna are alive.** A world of 2560×1440 cells is generated — plates,
> coastlines, climate, rivers, biomes, forests, ore bodies — and about fifteen thousand animals live
> on it: they graze, keep to their herds, cross ridges to new pasture, run from predators, hunt,
> starve, breed, grow old, and go extinct for good. **The people are next**, and after them the
> nations, work, war, and the player's powers.

## What is actually running

**The ground** is searched for rather than rolled: a world is generated, measured against criteria
(one continent, not fourteen islands), and rejected if it fails. Tectonic plates give the relief,
climate follows the relief, rivers follow the climate, biomes follow both, and forests and ore bodies
follow the biomes.

**The fauna** is seventeen species on a table of real numbers — how fast, how shy, how long they
live, how many run together, what ground they can live on. Nothing above it is scripted:

- a **herd thinks and a beast follows**, which is a few thousand decisions and fifteen thousand moves
  rather than fifteen thousand decisions;
- **flight** is one comparison against `SHY`, and running costs stamina;
- **the hunt has no chase routine at all** — prey flees in bursts and blows, a pack walks and does
  not tire, and the animal that falls behind is the animal that gets caught;
- **a species can end**, permanently, and the world says which of five things did it.

**The window is where it is judged.** There is no headless mode and no smoke test standing in for a
run: development is verified by playing `AgentsExe game`, watching it, and reading what it says on
the way out — how many ticks it ran, what each stretch of the frame cost, how many animals are alive,
how far they walked, how many were drawn.

## Build & run

Needs the LDP3 toolchain (the sibling `../LDP3` dev build) and the sibling `../ldp3-opengl` library.
Asset paths resolve relative to the executable, so it runs from any location.

```
ldp3 build
build-output/AgentsExe.exe game
```

`game` — or no argument at all — plays. It takes two options and no more, because a game is not
configured from a command line:

| option | what it does |
|---|---|
| `gear=N` | open at speed N (1..6). Left out, it opens at the ordinary speed and the player drives |
| `for=S` | stop after S seconds. For anything that is not a hand: a gate, a measurement, a run left going |

The seed is 42 and stays 42: one world, so that two runs are the same world.

## Controls

| Input | Action |
|-------|--------|
| **drag** / arrows | rotate the camera |
| **scroll** | zoom |
| **WASD** (Shift for fast) | pan |
| **1**…**6** | speed: real time up to three hundred ticks a frame |
| **P** | pixel filter on/off |
| **Ctrl-C** | end the run and report, rather than being killed mid-frame |
| **Esc** | quit |

## Layout

- `src/world/` — the grid and its layers, and `worldgen/`: plates, climate, rivers, biomes, flora, ore
- `src/sim/` — the world's tick: `Sim` owns the order things happen in; `Wild`, `Herd`, `Beast`, the
  block index, the fauna's rules, and the chronicle that says what happened in each tick
- `src/gfx/` — the 3D renderer: two-level LOD, instanced meshes, cel shading, the frame meter
- `src/app/` — the window, the options, the interrupt
- `src/test/` — the suite that runs inside the program
- `assets/models/` — the meshes, three flat tones each

## How it is built

One slice at a time, and **a slice is not done until it is drawn**. Every slice ends with the real
game running, a gate that fails on behaviour rather than on matching output text, and a ledger line
for every hole found on the way — including the ones that turned out to be the compiler's.
