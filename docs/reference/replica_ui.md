# agents.exe — UI & Player-Interaction Subsystem (faithful-replica spec)

Reverse-engineered 1:1 from the shipped Unity C# source. All facts cite
`unity-project/Assets/...` with `file:line`. Where the assignment brief assumed a
feature that the code does **not** implement, that is flagged explicitly as
**[NOT IN CODE]** so the re-implementation stays faithful to what actually ships.

Project identity: `productName: agents.exe`, `companyName: IPCA MEDJD`,
reference/native resolution **1280×720**, windowed (`fullscreenMode: 3`)
— `ProjectSettings/ProjectSettings.asset`.

---

## 0. Global visual style ("Windows 95/98" chrome)

Every screen is built in code from a single white 1×1 sprite (`White1x1.png`) tinted
per element, plus 1–2 px bevel strips to fake raised/sunken 3D chrome. There are **no
textured UI skins** — the entire look is flat rectangles + bevels + TextMeshPro.

Shared palette (identical constants appear in `HUDController.cs:18-23`,
`PauseMenuController.cs:21-25`, `BuildMainMenuScene.cs:18-27`, `BuildUIScenes.cs:16-23`):

| Name | RGBA | Use |
|------|------|-----|
| C_GRAY | 192,192,192 | panel/button face |
| C_GRAY_DK | 128,128,128 | bevel shadow, disabled text |
| C_GRAY_LT | 223,223,223 | clock inset |
| C_WHITE | 255,255,255 | bevel highlight |
| C_BLACK | 0,0,0 | body text |
| C_BLUE_DK | 0,0,128 (title bars use **8,26,147**) | title bars / headers |
| C_GOLD | 228,185,60 | logo wordmark |
| C_BG | 26,58,90 (desktop blue) | menu background fill |

**Bevels** — a raised box draws white top+left, dark-gray bottom+right strips; a sunken
box inverts it (`AddBevelRaised`/`AddBevelSunken` in `HUDController.cs:862-884`;
`AddBorderBevel`/`AddBevelSmall`/`AddBevelInSmall` in the Editor builders). Border bevel
= 2 px; button/inset bevel = 1 px.

**Buttons** (`MakeWinButton` `HUDController.cs:832-860`; `MakeWin95Button`
`BuildUIScenes.cs:731-775`): gray face, raised bevel, bold black centered label.
ColorBlock: normal `white`, highlighted `208-220,208-220,215`, pressed `168,168,175`,
disabled `185,185,188,200`.

**Fonts**: TextMeshPro, no custom font asset is ever assigned in code, so everything
renders in the TMP default **LiberationSans SDF**
(`Assets/TextMesh Pro/Resources/Fonts & Materials/LiberationSans SDF.asset`).
Sizes are set per-element in points (see each screen). Titles/labels/buttons are
`FontStyles.Bold`; placeholders are `Italic`.

**Canvas**: `ScreenSpaceOverlay`, `CanvasScaler.ScaleWithScreenSize`, reference
`1280×720`, `matchWidthOrHeight = 0.5`. Menu canvases sortingOrder 0; in-game HUD
canvas sortingOrder 50 (`HUDController.cs:231`); pause overlay canvas sortingOrder 100
(`BuildUIScenes.cs:459`).

**Scenes / build order** (`BuildUIScenes.cs:1036-1042`):
`MainMenu → NewGameMenu → LoadGameMenu → OptionsMenu → Game`.
(`SampleScene.unity` is a leftover, unused.)

---

## 1. Menu screens

All four menu scenes share a chrome idiom: full-screen desktop-blue `Background`, a
centered gray "application **Window**" with a blue title bar reading
`agents.exe — <Screen>` and a fake **X** close button (decorative — not wired), and a
bottom **Taskbar** (25 px) with a raised **Start** button (68 px) at the left and an
inset **SystemClock** reading a hard-coded **`21:42`** at the right
(`MakeTaskbar` `BuildUIScenes.cs:689-729`).

### 1.1 Main Menu — `MainMenu.unity` / `MainMenuController.cs`, built by `BuildMainMenuScene.cs`

Window **426×413**, offset y −20 (`:82-93`). Title bar `agents.exe — Main Menu`
(font 17 bold). Logo area: gold wordmark **"agents.exe"** at font **58** bold
(`:164-169`) over subtitle **"a Worldbox-style civilization simulator"** font 16
(`:178`). Divider line at 130 px from window top.

Four stacked buttons, 240×40, 11 px gap, first at 153 px from window top, label font 22
bold (`:196-258`):

| Button | Handler | Action |
|--------|---------|--------|
| **New Game** | `OnNewGameClicked` | load `NewGameMenu` (else `Game`) `MainMenuController.cs:20-25` |
| **Load Game** | `OnLoadGameClicked` | load `LoadGameMenu` `:27-31` |
| **Options** | `OnOptionsClicked` | load `OptionsMenu` `:33-37` |
| **Quit** | `OnQuitClicked` | `Application.Quit()` (editor: stop play) `:39-46` |

Footer divider 24 px from bottom; footer-left `"v0.1 prototype  •  TEIA 2026  •  IPCA"`,
footer-right `"João V. P. Tavares  21871"` (font 13, gray) `:263-291`. Top-right desktop
label `"1920×1080  •  16 bpp  •  agents.exe"` `:349`. `Start()` calls
`GameSessionState.ClearPendingLoad()` `:14-18`.

```
+----------------------------------------------------------+
| agents.exe — Main Menu                             [ X ] | blue title bar
+----------------------------------------------------------+
|                                                          |
|                    agents.exe                            | gold, 58pt
|          a Worldbox-style civilization simulator         | 16pt
|  ------------------------------------------------------  | divider
|                 +------------------------+               |
|                 |       New Game         |               | 240x40
|                 +------------------------+               |
|                 |       Load Game        |               |
|                 +------------------------+               |
|                 |        Options         |               |
|                 +------------------------+               |
|                 |         Quit           |               |
|                 +------------------------+               |
|  ------------------------------------------------------  |
| v0.1 prototype • TEIA 2026 • IPCA     João V.P.T. 21871  |
+----------------------------------------------------------+
[ Start ]                                            [ 21:42 ]   <- taskbar
```

### 1.2 New Game — `NewGameMenu.unity` / `NewGameMenuController.cs`, built in `BuildUIScenes.cs:38-141`

Window **720×510**. Title `agents.exe — New Game`; gold "New Game" logo; divider at 90 px.
Controls (left column, x=30, first at y=105) `BuildUIScenes.cs:79-124`:

- **Seed:** label + sunken white input field (280×24). Default text **"42"**, placeholder
  "Enter seed..." (`NewGameMenuController.cs:20-24`, default from
  `WorldGenParams.Default.Seed`). Font 17.
- **Randomize personalities** toggle — default **ON** (`:22`; toggle default on `BuildUIScenes.cs:845`).
- **Use LLM (requires server.py running)** toggle — default **OFF** (`:23`).
- **Start** button (120×34, bottom-right) → `OnStartClicked`; **Back** (bottom-left) →
  `OnBackClicked` → `MainMenu`.

`OnStartClicked` (`NewGameMenuController.cs:26-40`): parse seed into
`WorldGenParams` → `GameSessionState.PendingWorldGen`; set
`PendingDecisionSource = useLlm ? LLM : Mock`; `MenuOverrideActive = true`; clear pending
load; `SceneManager.LoadScene("Game")`.

> **[NOT IN CODE]** The brief's *"nation-origin choice: pre-placement vs organic
> emergence"* control **does not exist**. New Game has exactly three inputs: Seed,
> Randomize-personalities, Use-LLM. Nation origin is not a player choice anywhere in the
> menu code.

```
+-----------------------------------------------------------+
| agents.exe — New Game                               [ X ] |
|                        New Game                           | gold logo
| --------------------------------------------------------- |
|  Seed:                                                     |
|  +-------------------------------+                         |  sunken input "42"
|  | 42                            |                         |
|  +-------------------------------+                         |
|  [x] Randomize personalities                               |  default ON
|  [ ] Use LLM (requires server.py running)                  |  default OFF
|                                                            |
| [ Back ]                                        [ Start ]  |
+-----------------------------------------------------------+
[ Start ]                                            [ 21:42 ]
```

### 1.3 Load Game — `LoadGameMenu.unity` / `LoadGameMenuController.cs`, built in `BuildUIScenes.cs:143-250`

Window **800×510**. Header "Choose a save to continue" (font 14 bold). **5 save-slot
rows**, each row = a wide left slot button + a **70 px "DEL"** button
(`:181-219`). Slot row: 52 px tall, 6 px gap, first at y=82.

`RefreshSlots()` (`LoadGameMenuController.cs:17-30`): per slot i (1..5),
`SaveSystem.ReadSlotMeta` →
label `"Slot {n} — {meta.label} (tick {meta.tick})"` when a save exists, else
`"Slot {n} — empty"`. Empty slots' load **and** DEL buttons are set non-interactable.
Slot click → `LoadSlot` (load `SaveFile`, stash into `GameSessionState`, load `Game`).
DEL → `DeleteSlot` (`SaveSystem.DeleteSlot` + refresh). **Back** (110×32, bottom-left) →
`MainMenu`.

```
+---------------------------------------------------------------+
| agents.exe — Load Game                                  [ X ] |
|  Choose a save to continue                                    |
|  +------------------------------------------------+ +-------+  |
|  | Slot 1 — Save 21:07 (tick 4210)                | |  DEL  |  |
|  +------------------------------------------------+ +-------+  |
|  | Slot 2 — empty                                 | |  DEL  |  | (greyed if empty)
|  +------------------------------------------------+ +-------+  |
|  | Slot 3 — empty ...                             | |  DEL  |  |
|  | Slot 4 — empty ...                             | |  DEL  |  |
|  | Slot 5 — empty ...                             | |  DEL  |  |
| [ Back ]                                                      |
+---------------------------------------------------------------+
[ Start ]                                              [ 21:42 ]
```

### 1.4 Options (standalone scene) — `OptionsMenu.unity` / `OptionsMenuController.cs`, built in `BuildUIScenes.cs:252-447`

Window **614×480**. **Four tabs** across the top — `Gameplay | Display | Audio | Debug`
(86×20 each). Only **Gameplay** is drawn active/flush; the others are decorative
**[NOT WIRED]** — no tab has a click handler (`:287-316`). A sunken **ContentPanel** holds:

- **"AI Mode"** heading + subtitle "Choose how protagonist agents make decisions".
- A sunken **AIBox** with two **radio-style text lines** (bullets, not real toggles —
  purely descriptive labels) `:356-359`:
  - `• Phi-4-mini LoRA  (real fine-tuned model via FastAPI)`
  - `• Heuristic mock AI  (rule-based fallback, no LLM)`
- Divider, then **"Master volume:"** + a real horizontal **Slider** (200×16, blue fill,
  default **0.7**) `:365-417`.
- **"Use LLM (requires server.py running)"** toggle `:420`.

Buttons bottom-right: **Apply** (`OnApplyClicked`) + **Back** (`OnBackClicked`)
(`:433-439`).

`OptionsMenuController` persistence (`OptionsMenuController.cs`): keys `opt_volume`
(float, default 0.7) and `opt_source` (string "Mock"/"LLM"); Apply writes PlayerPrefs and
sets `AudioListener.volume`. Only `volumeSlider` and `useLlmToggle` are wired; the tabs
and AI-mode radio lines are cosmetic.

```
+-------------------------------------------------------------+
| agents.exe — Options                                  [ X ] |
| [Gameplay] Display  Audio  Debug          (only Gameplay live)|
| +---------------------------------------------------------+ |
| | AI Mode                                                 | |
| | Choose how protagonist agents make decisions            | |
| |  +----------------------------------------------------+ | |
| |  | • Phi-4-mini LoRA  (real fine-tuned model / FastAPI)| |  (labels only)
| |  | • Heuristic mock AI (rule-based fallback, no LLM)   | | |
| |  +----------------------------------------------------+ | |
| |  --------------------------------------------------     | |
| |  Master volume:                                         | |
| |  [=========|-----------]  (slider, def 0.7)             | |
| |  [ ] Use LLM (requires server.py running)               | |
| +---------------------------------------------------------+ |
|                                       [ Back ]  [ Apply ]   |
+-------------------------------------------------------------+
```

### 1.5 Pause Menu — `PauseMenuOverlay.prefab` / `PauseMenuController.cs`

Spawned at runtime, not a scene. `PauseMenuSpawner` (`PauseMenuSpawner.cs`) loads
`Resources/UI/PauseMenuOverlay` at Awake, instantiates it hidden, and toggles it on **Esc**:
Esc when hidden → `Time.timeScale = 0` + show + `controller.OnOpen()`; Esc when shown →
`Time.timeScale = 1` + hide (`:31-50`). Prefab built by `BuildUIScenes.cs:449-574`:
full-screen dim **Backdrop** `(0,0,0,160)`, centered **320×340** window,
title `agents.exe — Pause`, sortingOrder 100.

**MainPanel** (`:476-508`): status line `"Day 0   •   Population: 0   •   Nations: 0"`
(static placeholder text in prefab), divider, then five 240×36 buttons:

| Button | Handler | Notes |
|--------|---------|-------|
| **Resume** | `OnResumeClicked` | `timeScale=1`, hide overlay (`PauseMenuController.cs:44-48`) |
| **Save Game** | `OnSaveClicked` | opens the Save sub-panel `:50-56` |
| **Load Game** | *(null)* | **[NOT WIRED]** — method arg is `null` `BuildUIScenes.cs:493` |
| **Options** | `OnOptionsClicked` | opens inline Options panel `:90-97` |
| **Quit to Menu** | `OnQuitToMenuClicked` | `timeScale=1`, load `MainMenu` `:99-103` |

**SavePanel** (`BuildUIScenes.cs:510-553`, hidden by default): header "Choose a slot to
save:", 5 slot buttons `"Slot n — empty"` / `"Slot n — {label} (overwrite)"`
(`PauseMenuController.RefreshSlots :58-69`) → `SaveToSlot(n)` saves as
`"Save {HH:mm}"` at current tick (`:77-82`), plus a **Back** button.

**Inline Options panel** (`PauseMenuController.BuildOptionsPanelInline :139-257`): a
separate **420×280** window built in code (not the OptionsMenu scene), with title
"Options", a **Master volume** slider (sunken track, blue fill, handle, default 0.7),
a **Use LLM** checkbox + label "Use LLM (requires server.py running)", and **Apply**
(`OnApplyOptions` — applies **live**: sets `AudioListener.volume` and
`ProtagonistDecisionSystem.SetActiveSource(LLM/Mock)`) + **Back** buttons.

```
        +--------------------------------------+
        | agents.exe — Pause             [ X ] |   (dim backdrop behind)
        | Day 0 • Population: 0 • Nations: 0    |
        | ------------------------------------ |
        |        +----------------------+      |
        |        |       Resume         |      |
        |        |      Save Game       |      |
        |        |      Load Game       |  (not wired)
        |        |       Options        |      |
        |        |    Quit to Menu      |      |
        |        +----------------------+      |
        +--------------------------------------+
```

---

## 2. In-game HUD — `HUDController.cs` (Game scene)

The entire in-game HUD is **built procedurally at Awake** by `HUDController.BuildUI()`
(`:224-244`); nothing is authored in the scene. It first `EnsureEventSystem()` (spawns an
`EventSystem` + `InputSystemUIInputModule` if missing, `:80-85`), makes the 1×1 white
sprite, then builds: TopBar, Inspector, InspectorOpenTab, BottomBar, WarsPanel,
IntentsFeedPanel. Layout constants (`:25-28`): `TOP_BAR_H = 36`, `BOTTOM_BAR_H = 64`,
`INSPECTOR_W = 360`; side panels `SIDE_PANEL_W = 280`, header 28, body 150 (`:246-248`).

Refresh loop: `Update()` accumulates `Time.unscaledDeltaTime`; every `REFRESH_S = 0.4 s`
(`:61,103-107`) it calls `UpdatePromptBarVisibility()` + `RefreshTexts()`.

### 2.1 Top bar (`BuildTopBar :329-345`, `RefreshTexts :547-557`)

Full-width, 36 px tall, gray, raised bevel, docked to top. Single left-aligned bold
label (font 18, `FONT_BAR_INFO`) reading:

`Year {year}   ·   Tick {tick}   ·   Source: {src}{queued}`

where `tick = round(Time.time)`, `year = tick/60`, `src` = `ProtagonistDecisionSystem
.ActiveSource` (`None`/`Mock`/`LLM`), and `{queued}` appends `"   queued msgs: {n}"` when
`PlayerMessageQueue.Count > 0` (`:549-557`). So the brief's "Source Mock|LLM" and "queued
msgs" both live in this one top-bar string.

### 2.2 Wars panel — top-left, collapsible (`BuildWarsPanel :317-321`)

A `BuildCollapsibleSidePanel` (`:250-315`) anchored top-left at `(8, -(36+0))`, width 280,
header 28 + body 150. Blue header bar is a **Button**: clicking collapses/expands the body
and flips the toggle glyph **−/+** (`:306-312`). Title **"Active Wars"**.
Body (`RefreshWarsPanel :761-775`): iterates `WarRegistry.Instance.GetActiveWars()`,
one line per war `"• {A.Name}  vs  {B.Name}"`; shows `"(peace — no active wars)"` when
none, or `"(WarRegistry not ready)"` before init.

### 2.3 Recent Intents panel — left, below Wars (`BuildIntentsFeedPanel :323-327`)

Same collapsible widget, stacked directly under Wars (topOffset = header+body+8 = 186).
Title **"Recent Intents"**. Body (`RefreshIntentsFeed :777-802`): watches every nation's
`LastLeaderIntent`; when a nation's intent changes it prepends
`"[{Nation}] {intent}  \"{reason≤60}\""` to a feed capped at **6** lines
(`INTENT_FEED_MAX = 6`, `:54`). Empty → `"(no decisions yet)"`.

### 2.4 Inspector — right side (`BuildInspector :347-413`, refreshed `:562-608`)

Gray panel anchored to the **right edge**, width **360**, spanning from
`TOP_BAR_H+6` down to `BOTTOM_BAR_H+6` (`:351-357`). Blue **TitleBar** (30 px) with a
title label + a small **X** button (`btnClose`) that calls `SetInspectorOpen(false)`
(`:373-379`). Sunken body holds an optional **80×80 portrait box** (dark `40,40,50`
inset, top-left, shown only when the selected thing has a `SpriteRenderer`,
`:387-403`, `:598-606`) and a top-left wrapped text block (font 17, `FONT_INSPECTOR`).

When closed, a small **InspectorOpenTab** (150×30, label "▸  Inspector") appears at the
top-right and reopens it (`BuildInspectorOpenTab :415-442`, `SetInspectorOpen :218-222`).

Default/empty text: *"Click anything on the map to inspect: · agent → full status ·
building → state + city · território → owning nation"* (`:593`).

**Worker inspector** (`DescribeWorker :625-686`) — title is
`"Royal · {Role}"` for a protagonist else the `Profession`. Body lines:
`Name`, `Nation: #id`, `Profession`, `Activity` (`CurrentActivity`), `Carrying: {type} ×n`
(if carrying), `Age: {s}s` + `Lifespan`/`~left` (if finite), `HP`, `Stage`. A **[Family]**
block (spouse + children list, deceased flagged). For protagonists a **[Royal]** block:
`Role`, `Personality`, `Is Heir`, `Father`, `Mother`. For a **Leader** protagonist a
**[Last decision]** block: `Intent: {LastLeaderIntent}` + `Reason: "{≤200}"`.

> **[PARTIALLY IN CODE]** The brief lists worker HP/hunger/thirst/age/action/nation and
> protagonist "+ last prompt, raw LLM response, episodic memory, executing intent,
> message box". The actual worker panel shows **HP, Age, Activity, Nation, Profession,
> Stage, carrying, family** — there is **no thirst field, no separate hunger field** in
> the inspector (hunger exists on the model, `DeathNotification.cs:84`, but isn't printed
> here). The protagonist panel shows **Role/Personality/heir/parents + last Intent+Reason**
> only. **No raw LLM prompt, no raw response, no episodic-memory dump, and no per-inspector
> message box** are rendered — the "message box" is the shared bottom Send bar (§2.6).

**Building inspector** (`DescribeBuilding :688-703`) — title "Building". Lines:
`Type`, `Cell: (x, y)`, and if it has a `City`: `City: City_{id}`, `City pop`,
`Nation: {name} (#id)`, and `Constructed: yes / no (blueprint)`.

**Nation/territory inspector** (`DescribeNation :715-751`) — title "Nation". Lines:
`Name`, `ID: #id`, `Population` (sum of city pops), `Cities`, `Wars won/lost`,
`Leader` + `Personality` (or "(vacant)"), a **[Last decision]** block, and a **[Cities]**
list (`City_{id} (capital) — pop n`).

### 2.5 Speed controls — bottom bar left (`BuildBottomBar :444-472`)

Full-width gray bar, **64 px**, raised bevel, docked bottom. Four 70×42 buttons from
x=12, 75 px pitch: **Pause | 1x | 2x | 4x** → set `Time.timeScale` to `0 / 1 / 2 / 4`
(`labels`/`speeds` arrays `:455-456`). Note the fourth button is labelled **4x** and maps
to timeScale **4** (there is no 3x).

### 2.6 Send-Message box — bottom bar right, **LLM-only** (`BuildBottomBar :474-525`)

A **PromptBar** child spans the bottom bar between x=315 and `INSPECTOR_W+16` from the
right. It contains a sunken white **TMP_InputField** (placeholder italic
*"Message to Protagonist..."*) and a 110×42 **Send** button. Submit (Enter) or Send →
`OnSendClicked` (`:527-536`): non-blank text → `PlayerMessageQueue.Enqueue(text)`, clear,
re-focus. `UpdatePromptBarVisibility` (`:538-545`) shows the whole PromptBar **only when
`ProtagonistDecisionSystem.ActiveSource == LLM`** — in Mock/None mode it is hidden.

Queued text is consumed by `ProtagonistDecisionSystem` (`PlayerMessageQueue.Dequeue()`,
`ProtagonistDecisionSystem.cs:41`) and injected into the next leader decision.
While typing, keyboard game-shortcuts are suppressed via `InputFieldFocused()`
(`HUDController.cs:95,122-125`).

### 2.7 HUD ASCII layout

```
+==============================================================================+
| Year 12  ·  Tick 740  ·  Source: LLM      queued msgs: 2                     |  top bar (36px)
+---------------+--------------------------------------------+-----------------+
| Active Wars − |                                            | Inspector   [X] |  <- 360px
| • Redharbor   |                                            |  +----+         |     right panel
|   vs Kaeloria |                                            |  |port|  Royal  |
| • ...         |            (isometric world map:           |  +----+ ·Leader |
+---------------+             tilemap terrain + territory     | Name: King...  |
| Recent        |             fill/border, agents, buildings, | Nation: #0     |
| Intents  −    |             fauna, leader markers)          | Activity: Idle |
| [Kael] Build  |                                            | HP: 100        |
|  army "..."   |                                            | [Royal]        |
| [Red] Migrate |                                            | [Last decision]|
| ...(≤6)       |                                            |  Intent: ...   |
+---------------+--------------------------------------------+-----------------+
| [Pause][1x][2x][4x]   [ Message to Protagonist... ] [ Send ]                 |  bottom bar (64px)
+==============================================================================+
       (Send box + input shown ONLY when Source = LLM)
```

---

## 3. Selection / player interaction (the only implemented "powers")

**Click-to-inspect** is the sole map interaction (`HUDController.Update :87-108`,
`HandleMapClick :132-149`). On left-mouse-down **not over UI**
(`IsPointerOverUI` via `EventSystem.IsPointerOverGameObject`, `:127-130`):
1. `Camera.main.ScreenToWorldPoint` → world point.
2. `FindWorkerNear(wp, 0.5)` — nearest alive worker within **0.5** world units
   (`CLICK_RADIUS_AGENT`, `:29,160-177`), scanning `NationManager.Nations[].Members`.
3. else `FindBuildingNear(wp, 1.5)` — nearest non-destroyed building within **1.5**
   (`CLICK_RADIUS_BUILDING`, `:30,179-199`), scanning nations' cities' buildings.
4. else `FindNationAtTile(wp)` — `Grid.WorldToCell` → `NationManager.TileNation[x,y]`
   → owning nation (`:201-216`).
`Select()` stores the pick and force-opens the Inspector (`:151-158`). Selection is
cleared implicitly when the picked worker dies / building is destroyed (`:564-565`).

There is **no multi-select, no drag-select, no hover highlight, no brush**.

---

## 4. "Worldbox-style powers" — **[NOT IMPLEMENTED IN SHIPPED CODE]**

This is the single biggest divergence from the brief and must be recorded faithfully.

The brief describes a god-power tool palette (paint/remove 5 terrains, paint 4 biomes,
add/remove resources, spawn/kill workers & protagonists, trigger drought/flood/earthquake,
brush size, etc.). **No runtime C# implements any of this.** Evidence:

- A repo-wide search for `brush|PaintTerrain|drought|flood|earthquake|PowerTool|
  ToolPalette|SpawnWorker|Worldbox` matches **zero gameplay scripts** — only Python
  dataset/tooling files, the MainMenu subtitle string, and art `.meta` files.
- The powers exist **only as 16×16 art icons** in `Assets/Art/UI/Powers/`
  (`Brush, Plant, Spawn_Worker, Spawn_Protagonist, Spawn_Building, Spawn_Fauna, Kill,
  Heal, Promote, Select, Toggle_Territory, Toggle_Pathing, Toggle_Intents, Pause, Play,
  Step, Save, Speed_1x/2x/4x`) generated by `python-project/tools/gen_god_power_icons.py`.
  That script groups them as design intent for **"Fase 3"**: *1. Tempo/sistema · 2.
  Inspeção/overlays · 3. Edição mundo (Brush, Plant) · 5. Spawning/manipulação
  (Spawn_*, Kill, Heal, Promote)* (`gen_god_power_icons.py:297-306`).
- **No script loads, instantiates, or references these Powers sprites** (grep for "Powers"
  across `Scripts/` = 0 hits). There is no toolbar GameObject, no brush-size field, no
  event trigger.

So for a faithful replica: the tool palette is a **planned-but-absent** feature. The
terrain/biome data it *would* paint does exist in the world model —
`Biome` enum = `DeepOcean, Ocean, Beach, Grassland, Mountain, Snow`
(`Scripts/World/Biome.cs:3-`; 6 biomes, not 4), painted by `TilemapPainter`; buildings
(`BuildingType`: Castle, TownHall, Farm, Sawmill, HuntersLodge, FishingHut, Smithy,
Bakery, Brewery, Forge, Marketplace, Barracks, House); professions (`Profession` enum,
12 values incl. Soldier/General); resource nodes seeded by `ResourceNodeSeeder`
(trees pine/oak/palm, bush, stone/iron nodes, deer fauna — `BuildGameSceneMenu.cs:168-174`)
— but **the player cannot edit any of it at runtime**. Replicating "faithfully" means
replicating the *absence* of these tools (or implementing them as an explicit new feature).

---

## 5. Camera & input

**CameraController** (`Player/CameraController.cs`, on the Game `Main Camera`,
orthographic, `orthographicSize` 130 initial per `BuildGameSceneMenu.cs:38`):

- **Pan**: `W/A/S/D` **or** arrow keys → move `transform.position` by
  `dir.normalized * panSpeed(20) * (orthoSize/10) * unscaledDeltaTime` — pan speed scales
  with zoom so it feels constant (`:24-39`). Works while paused (uses unscaled time).
- **Zoom**: mouse **scroll wheel** → `orthographicSize -= sign(scroll) * zoomSpeed(5) *
  (orthoSize*0.05)`, clamped **[1.5, 250]** (`minZoom`/`maxZoom`, `:41-51`).
- Isometric setup: `transparencySortMode = CustomAxis`, axis `(0,1,0)` (`:20-21`);
  world is an isometric `Grid` (`IsometricZAsY`, cellSize `(1, 0.5, 0.25)`,
  `BuildGameSceneMenu.cs:49-52`).

**Keyboard shortcuts** (new Input System, `HUDController.Update :95-101`, when the send
field is **not** focused):

| Key | Action |
|-----|--------|
| **Space** | `ToggleSpeed()` — pause ↔ resume at last non-zero speed (`:117-121`) |
| **1** | `SetSpeed(1)` → 1× |
| **2** | `SetSpeed(2)` → 2× |
| **3** | `SetSpeed(4)` → **4×** (digit 3 maps to 4×, matching the "4x" button) |
| **Esc** | toggle Pause overlay (`PauseMenuSpawner.cs:31-50`) |

**Mouse**: left-click = select/inspect (§3); scroll = zoom. No right-click or
middle-drag panning is implemented.

---

## 6. Legacy / disabled overlays — `UICleanupBootstrap.cs`

Several older IMGUI (`OnGUI`) overlays still exist as components in the Game scene but are
**disabled for the shipped build** by `UICleanupBootstrap.Awake()` (`:21-29`), which finds
each by type and sets `enabled = false`. Flags (`:9-19`):

| Overlay (type) | Killed? | What it was |
|----------------|---------|-------------|
| `NationStockpileOverlay` | yes | resource debug, top-corner |
| `WarOverlay` | yes | war list, bottom-corner |
| `GameSpeedHud` (`Player/GameSpeedHud.cs`) | yes | **legacy** OnGUI speed label top-right: `"Speed: 1x  [1] [2] [3] [Space pause]"`, orange `"PAUSED …"` when paused (`:73-106`) — **replaced by** the HUDController bottom bar |
| `DeathNotification` (`Player/DeathNotification.cs`) | **no (kept)** | centered 500×80 dark banner for 5 s on leader/heir death, e.g. `"{name} of {nation} has died (combat/starvation/old age). {heir} assumes leadership."` (`:74-110`) |
| `SettingsMenu` (`Player/SettingsMenu.cs`) | yes | legacy Esc IMGUI box (decision source Mock/LLM/None, tick-interval slider, debug toggles) — replaced by PauseMenuOverlay |
| `DecisionToastController` (`Player/DecisionToastController.cs`) | yes | top-center nation-colored toast on notable intents/combat — replaced by the Inspector's Last-decision + Recent-Intents panel |

So the **only** always-on runtime HUD is `HUDController`; the **only** kept IMGUI popup is
the central `DeathNotification` banner. `LeaderMarker` (`Player/LeaderMarker.cs`) is a
world-space sprite child that floats a colored marker above a nation's leader (not screen UI).
Background loggers (`CityStatusLogger`, `CityScreenshotCapture`) write to disk and draw
nothing.

`GameSpeedHud` also has a quirk: `OnDestroy` resets `Time.timeScale = 1` (`:108-112`) — a
faithful port should keep pause/speed state in one owner (HUDController) to avoid the two
speed systems fighting.

---

## Reference — exact file inventory

UI (`Assets/Scripts/UI/`): `GameSessionState.cs`, `HUDController.cs` (the whole in-game
HUD), `LoadGameMenuController.cs`, `MainMenuController.cs`, `NewGameMenuController.cs`,
`OptionsMenuController.cs`, `PauseMenuController.cs`, `PauseMenuSpawner.cs`,
`PlayerMessageQueue.cs`, `UICleanupBootstrap.cs`.
Player (`Assets/Scripts/Player/`): `CameraController.cs`, `CityScreenshotCapture.cs`,
`CityStatusLogger.cs`, `DeathNotification.cs`, `DecisionToastController.cs`,
`GameSpeedHud.cs`, `LeaderMarker.cs`, `SettingsMenu.cs`.
Scene builders (`Assets/Editor/`): `BuildMainMenuScene.cs`, `BuildUIScenes.cs`
(New Game / Load Game / Options / Pause prefab), `BuildGameSceneMenu.cs` (Game scene —
note: stale; it omits the HUD/Pause/Cleanup components that the shipped `Game.unity`
actually contains — each verified present by GUID).
Scenes: `MainMenu`, `NewGameMenu`, `LoadGameMenu`, `OptionsMenu`, `Game` (+ unused
`SampleScene`). Prefab: `Resources/UI/PauseMenuOverlay.prefab` (loaded by name at runtime).
Art: `Assets/Art/UI/Powers/*` (icons, unused by code), `Assets/Art/UI/*` (bubbles, resource
chips, selection ring), font = TMP default LiberationSans SDF.
