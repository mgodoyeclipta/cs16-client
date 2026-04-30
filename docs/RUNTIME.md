# cs16-client Runtime & Architecture

This document explains how cs16-client works from a user perspective: what each binary does, how they interact, and how to set up and run Counter-Strike 1.6 with this project.

## Project Purpose

cs16-client is a reverse-engineered Counter-Strike 1.6 client. It is **not** a standalone game — it is a set of plugin DLLs loaded by the Xash3D FWGS engine (a re-implementation of the GoldSrc/Half-Life 1 engine).

## The Three Layers

Running CS 1.6 with cs16-client requires three things:

### Layer 1: Xash3D FWGS Engine

The engine binary. Provides:

- **Launcher** (`xash3d`) — the executable the user runs
- **Engine core** (`libxash.dylib` / `libxash.so` / `libxash.dll`) — main loop, networking, sound, entity management, console/CVar system, physics skeleton, demo recording, VGUI2 panel root
- **Renderer** (`libref_gl.dylib` — OpenGL, `libref_soft.dylib` — software fallback)
- **Filesystem** (`filesystem_stdio.dylib`) — Virtual File System: mounts PK3/PAK/WAD/BSP files
- **Default menu** (`libmenu.dylib`) — generic engine menu UI

### Layer 2: Steam CS 1.6 Game Assets

The original game content from Steam. Two directories are required:

- `valve/` — base Half-Life content (engine requires this)
- `cstrike/` — Counter-Strike content (maps, models, sounds, sprites, wad files, etc.)

These come from a Steam installation of Half-Life + Counter-Strike 1.6.

### Layer 3: cs16-client Binaries (this project)

The reverse-engineered game DLLs that replace the original proprietary ones. They provide Counter-Strike 1.6 game logic:

| Binary | Replaces | Purpose |
|--------|----------|---------|
| `client_arm64.dylib` | Original `client.dll` | Client game logic: HUD (health, armor, money, timer, radar, radio, scoreboard, crosshair, sniperscope, night vision, scenario), weapon prediction (recoil, animation, fire rate), event handlers (muzzle flash, shell ejection, bullet impact — one file per weapon), VGUI2 menus (buy menu, team select, class select, spectator), player movement, input processing, view calculation |
| `cs_arm64.dylib` | Original `mp.dll` / `cs.dll` | Server game logic: round system, buy time, freeze time, bomb plant/defuse, hostage rescue, win conditions, weapon damage/accuracy/recoil, player spawn/death/money, entity logic (doors, triggers, hostages) |
| `menu_arm64.dylib` | Engine's `libmenu.dylib` | Custom CS 1.6 menu UI: main menu, server browser, options dialogs, game creation. Replaces the engine's generic menu. |
| `yapb_arm64.dylib` | No original equivalent | YaPB bot AI: navigation graphs, bot behavior, chatter system. Provides bot players for offline play. |

## Directory Layout After Setup

```
xash3d-fwgs-apple-arm64/          # Xash3D engine directory
├── xash3d                        # Engine launcher
├── libxash.dylib                 # Engine core
├── libref_gl.dylib               # OpenGL renderer
├── libref_soft.dylib             # Software renderer
├── filesystem_stdio.dylib        # VFS
├── libmenu.dylib                 # Default menu (replaced by cs16-client)
├── SDL2.framework/               # SDL2
├── valve/                        # Half-Life base content (from Steam)
│   ├── *.wad, maps/, models/, sound/, etc.
│   └── ...
└── cstrike/                      # Counter-Strike content (from Steam + cs16-client)
    ├── *.wad                     # WAD texture files (from Steam)
    ├── maps/                     # Map files (from Steam)
    ├── models/                   # Model files (from Steam)
    ├── sound/                    # Sound files (from Steam)
    ├── sprites/                  # Sprite files (from Steam)
    ├── gfx/                      # UI graphics (from Steam)
    ├── resource/                 # UI resources (from Steam)
    ├── events/                   # Event scripts (from Steam)
    ├── liblist.gam               # Mod configuration — tells engine to load client DLL
    ├── cl_dlls/                  # CLIENT DLLs go here
    │   ├── client_arm64.dylib    # cs16-client (replaces original client.dll)
    │   └── menu_arm64.dylib      # cs16-client (replaces engine's libmenu.dylib)
    ├── dlls/                     # SERVER DLLs go here
    │   ├── cs_arm64.dylib        # cs16-client (replaces original mp.dll)
    │   └── yapb_arm64.dylib      # cs16-client (bot AI)
    └── extras.pk3                # cs16-client (bot configs, nav graphs, extras)
```

## How to Set Up and Run

### Step 1: Get Xash3D FWGS

Download the latest developer build for your platform from [FWGS/xash3d-fwgs releases](https://github.com/FWGS/xash3d-fwgs/releases).

### Step 2: Get Steam CS 1.6 Assets

Copy `valve/` and `cstrike/` directories from your Steam installation:

- **Linux/Windows:** `~/.steam/steam/steamapps/common/Half-Life/`
- **macOS:** Steam installs CS 1.6 under the Steam content directory

### Step 3: Build and Install cs16-client

```shell
# Build (macOS ARM64 example)
cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=gcc-15 \
  -DCMAKE_CXX_COMPILER=g++-15 \
  -DCMAKE_CXX_STANDARD=11 \
  -DCMAKE_CXX_STANDARD_REQUIRED=ON

cmake --build build

# Install — copies DLLs into the Xash3D directory
cmake --install build --prefix /path/to/xash3d-fwgs
```

### Step 4: Run

```shell
cd /path/to/xash3d-fwgs
./xash3d -game cstrike
```

The `-game cstrike` flag tells the engine to load the `cstrike/` mod directory. The engine then:

1. Reads `cstrike/liblist.gam` — sees `cldll "1"` — loads `cstrike/cl_dlls/client_arm64.dylib`
2. Loads `cstrike/dlls/cs_arm64.dylib` for server logic
3. Loads `cstrike/cl_dlls/menu_arm64.dylib` for the menu UI
4. Mounts `extras.pk3` via the VFS

## The Plugin Architecture

cs16-client DLLs are **not** standalone programs. They are loaded by the engine via `dlopen()`/`LoadLibrary()` and communicate through a **bidirectional function table exchange** (the GoldSrc ABI).

### Engine to Client (~140 functions)

The engine provides `cl_enginefunc_t` to the client DLL:

- **Drawing:** `pfnSPR_Load`, `pfnSPR_Draw`, `pfnFillRGBA`, `pfnSetCrosshair`
- **Sound:** `pfnPlaySoundByName`, `pfnPlaySoundByNameAtPitch`
- **Console:** `Con_Printf`, `pfnConsolePrint`, `pfnCenterPrint`
- **CVars/Commands:** `pfnRegisterVariable`, `pfnGetCvarFloat`, `pfnAddCommand`
- **User Messages:** `pfnHookUserMsg` — client registers callbacks for server-to-client messages
- **Events:** `pfnPrecacheEvent`, `pfnPlaybackEvent`, `pfnHookEvent`
- **Entities:** `GetLocalPlayer`, `GetViewModel`, `GetEntityByIndex`
- **Prediction:** `PM_PointContents`, `PM_TraceLine`
- **Sub-APIs:** `pTriAPI`, `pEfxAPI`, `pEventAPI`, `pDemoAPI`, `pNetAPI`

### Client to Engine (~40 functions)

The client DLL exports `cldll_func_t` to the engine:

- `Initialize` — receives engine function table, checks interface version (7)
- `HUD_Init` / `HUD_VidInit` — initialization
- `HUD_Redraw` — **the main draw call every frame** — draws all HUD elements
- `HUD_UpdateClientData` — sync shared client data
- `CL_CreateMove` — build user command from input
- `V_CalcRefdef` — camera/view setup
- `HUD_AddEntity` / `HUD_CreateEntities` — entity rendering hooks
- `pClientMove` — player movement prediction
- `pStudioEvent` / `pStudioInterface` — model events

### The Handshake

1. Engine loads `client_arm64.dylib` via `dlopen()`
2. Engine calls `F()` — client fills in `cldll_func_t` with its function pointers
3. Engine calls `Initialize(cl_enginefunc_t*, 7)` — client copies engine functions to `gEngfuncs`
4. Engine calls `HUD_GetRenderInterface()` — Xash3D extension: client gets render API (version 37)
5. Engine calls `HUD_MobilityInterface()` — Xash3D mobile extension: touch/vibration API

### Per-Frame Call Sequence

Every frame, the engine calls into the client DLL:

1. `HUD_Frame(time)` — per-frame logic
2. `HUD_UpdateClientData(cdata, time)` — sync data
3. `CL_CreateMove(frametime, cmd, active)` — build user command from input
4. `V_CalcRefdef(ref_params)` — camera setup
5. `HUD_AddEntity(type, ent, modelname)` — entity rendering hook
6. `HUD_CreateEntities()` — client-side entities
7. `HUD_Redraw(time, intermission)` — **main draw call** — draws all HUD
8. `HUD_DrawNormalTriangles()` / `HUD_DrawTransparentTriangles()` — custom rendering
9. `HUD_TempEntUpdate(...)` — temp entities (muzzle flashes, etc.)
10. `HUD_StudioEvent(event, entity)` — model events

The client calls back into the engine using `gEngfuncs.*`:

- `gEngfuncs.pfnSPR_Draw()` — draw a sprite
- `gEngfuncs.Con_Printf()` — print to console
- `gEngfuncs.pfnHookUserMsg()` — register message handler
- `gEngfuncs.pfnPlaybackEvent()` — fire an event

## liblist.gam — The Mod Configuration

```ini
game "Counter-Strike"
cldll "1"                         # Tells engine to load client DLL
gamedll "dlls\mp.dll"             # Windows server DLL path
gamedll_linux "dlls/cs.so"        # Linux server DLL path
gamedll_osx "dlls/cs.dylib"       # macOS server DLL path
```

The `cldll "1"` line is critical — without it, the engine uses the built-in Half-Life HUD instead of the CS 1.6 client DLL.

The client DLL path follows a **convention** (not explicitly in liblist.gam): the engine looks for `cl_dlls/client.dylib` (macOS), `cl_dlls/client.dll` (Windows), or `cl_dlls/client.so` (Linux). It appends the architecture postfix itself (e.g., `_arm64` on Apple Silicon).

## Xash3D-Specific Extensions

cs16-client uses Xash3D extensions beyond the original GoldSrc ABI:

- **Render API** (`CL_RENDER_INTERFACE_VERSION 37`) — extended rendering functions
- **Mobile API** (`MOBILITY_API_VERSION 2`) — touch controls, vibration
- **Menu Extended API** (`MENU_EXTENDED_API_VERSION 1`) — menu-client communication

The client enforces a minimum Xash3D version: `MIN_XASH_VERSION 3137` (checked in `cdll_int.cpp`).

## Shared Code Pattern

Some code is compiled into **both** the client and server DLLs:

- `dlls/wpn_shared/*.cpp` — weapon behavior (shared client/server)
- `pm_shared/*.cpp` — player movement prediction (shared client/server)
- `game_shared/*.cpp` — voice status code (shared)

Changes to shared code affect both sides. This is critical for weapon balance: client-side prediction must match server-side validation.

## CVars (Console Variables)

cs16-client adds custom CVars for CS 1.6-specific features:

| CVar | Default | Description |
|------|---------|-------------|
| `cl_crosshair` | 1 | Crosshair type (0=none, 1=classic, 2=dot, 3=dynamic) |
| `cl_crosshair_color` | "50 250 50" | Crosshair RGB color |
| `cl_crosshair_translucent` | 1 | Translucent crosshair |
| `cl_crosshair_size` | "auto" | Crosshair size (auto/small/medium/large) |
| `hud_scale` | 0 | HUD scale factor |
| `hud_takesshots` | 0 | Auto-screenshot on map change |
| `cl_quakeguns` | 0 | Quake-style gun positioning |
| `cl_weapon_quality` | "high" | Weapon model quality |
| `cl_righthand` | 1 | Right-handed weapon models |

These are accessible via the console (`~` key) or in-game options menu.
