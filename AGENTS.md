# AGENTS.md — cs16-client

## Project

Fork of [Velaron/cs16-client](https://github.com/Velaron/cs16-client) — reverse-engineered Counter-Strike 1.6 client built on Xash3D FWGS engine.

**Fork goal:** Make it run on macOS Apple Silicon (arm64) and keep it as close to the original upstream as possible. Prefer upstream-compatible fixes over fork-specific workarounds.

## Build

```shell
# MUST clone with --recursive (4 git submodules)
git clone --recursive https://github.com/Velaron/cs16-client

# Configure (use presets, see CMakePresets.json for full list)
cmake --preset linux-release-amd64      # Linux amd64
cmake --preset linux-release-i386       # Linux i386 (cross-compile, uses toolchains/i386-linux-gnu.cmake)
cmake --preset win32-release-x86        # Windows x86
cmake --preset win32-release-amd64      # Windows amd64
cmake --preset psvita-release           # PS Vita (requires $VITASDK)

# Build & install
cmake --build build
cmake --install build --prefix <xash3d-install-dir>

# Android (separate Gradle build)
cd android && ./gradlew assembleGitDebug
```

### Build options

- `BUILD_CLIENT=ON` (default) — client library (`cl_dll/`)
- `BUILD_MAINUI=ON` (default) — menu UI library (`3rdparty/mainui_cpp/`)
- `BUILD_SERVER=ON` (default) — server library (YaPB bot + ReGameDLL_CS). PS Vita sets this OFF.

### Prerequisites

- Python interpreter required at configure time (runs `scripts/yapb_graph_dl.py` and `scripts/pack_extras.py`)
- Linux i386 build needs `gcc-multilib g++-multilib` and `libfontconfig-dev:i386`
- macOS build uses `gcc`/`g++` (not Apple Clang), requires `brew install cmake gcc ninja fontconfig`
- PS Vita needs VitaSDK (`$VITASDK` env var)

## Architecture

```
cl_dll/          → Client DLL (the main contribution — reverse-engineered CS 1.6 client)
  ├── hud/       → HUD elements (money, radar, radio, scoreboard, etc.)
  ├── events/    → Weapon event handlers (one file per weapon, e.g. event_ak47.cpp)
  ├── cs_wpn/    → Client-side weapon prediction
  ├── VGUI/      → VGUI2 menu panels (buy menu, class select, scoreboard, etc.)
  ├── input/     → Platform input (SDL, Xash3D)
  └── include/   → Client headers
dlls/            → Shared server headers + wpn_shared/ (weapon code shared between client & server)
common/          → Engine interface headers (from Xash3D SDK)
engine/          → Engine API headers
pm_shared/       → Player movement code (shared client/server)
game_shared/     → Voice status code (shared)
public/          → build.h (platform detection), utflib, cl_dll SDK headers
3rdparty/        → Git submodules (yapb, mainui_cpp, ReGameDLL_CS, miniutl)
                   + cs16client-extras/ (assets, not a submodule)
```

### Shared code pattern

Client and server share weapon and movement code:
- `dlls/wpn_shared/*.cpp` — compiled into BOTH client (`cl_dll/CMakeLists.txt` includes them) and server
- `pm_shared/*.cpp` — compiled into client and server
- `game_shared/*.cpp` — voice code compiled into both

When editing weapon behavior, check whether changes need to apply to both sides.

### Platform detection

`public/build.h` defines `XASH_*` macros for OS/arch detection at compile time. `cmake/LibraryNaming.cmake` reads these to set library output names with platform postfixes (e.g. `client_amd64`, `client_psvita_armv7hf`). Do not rename output libraries without updating this logic.

## Code Style

- `.clang-format` present: WebKit-based, **Allman braces**, **tabs for indentation**, pointer alignment right (`int *foo`), `SortIncludes: false`
- Defines: `CLIENT_DLL` and `CLIENT_WEAPONS` are set for client builds; `LINUX`/`_LINUX` on non-Windows
- Non-MSVC: `stricmp` → `strcasecmp`, `-fms-extensions` enabled

## Key Conventions

- No test suite exists — verification is build-only (`cmake --build`)
- No linter or typecheck command — formatting uses `.clang-format`
- CI builds all platforms on every push/PR; continuous releases from `main` branch
- `inputw32.cpp` and `ev_hldm.cpp` are explicitly excluded from builds (legacy HL code)
- Output libraries have no `lib` prefix on non-Android platforms (set by `set_target_postfix`)
