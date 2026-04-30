# cs16-client Architecture

Fork of [Velaron/cs16-client](https://github.com/Velaron/cs16-client) — a reverse-engineered Counter-Strike 1.6 client built on the Xash3D FWGS engine. The fork's goal is to run on macOS Apple Silicon (arm64) while staying as close to upstream as possible.

---

## Directory Structure

```
cs16-client/
├── cl_dll/                 # Client DLL — the main contribution
│   ├── hud/                # HUD elements (money, radar, radio, scoreboard, etc.)
│   ├── events/             # Weapon event handlers (one file per weapon, e.g. event_ak47.cpp)
│   ├── cs_wpn/             # Client-side weapon prediction
│   ├── VGUI/               # VGUI2 menu panels (buy menu, class select, scoreboard, etc.)
│   ├── input/              # Platform input (SDL, Xash3D)
│   └── include/            # Client headers
├── dlls/                   # Shared server headers + wpn_shared/ (shared weapon code)
├── common/                 # Engine interface headers (from Xash3D SDK)
├── engine/                 # Engine API headers
├── pm_shared/              # Player movement code (shared client/server)
├── game_shared/            # Voice status code (shared)
├── public/                 # build.h (platform detection), utflib, cl_dll SDK headers
├── 3rdparty/               # Git submodules + extras
│   ├── cs16client-extras/  # Game assets (NOT a submodule, tracked in repo)
│   │   ├── BotChatter.db   # Bot chatter database
│   │   ├── BotProfile.db   # Bot profile database
│   │   ├── gfx/            # Graphics assets
│   │   ├── maps/           # Map files
│   │   ├── media/          # Media files
│   │   ├── sound/          # Sound assets
│   │   ├── touch_default/  # Default touch config
│   │   ├── touch_presets/  # Touch preset configs
│   │   ├── touch.cfg       # Touch configuration
│   │   └── userconfig.d/   # User config defaults
│   ├── mainui_cpp/         # Git submodule: VGUI2 menu UI library
│   ├── miniutl/            # Git submodule: Mini utility library
│   ├── ReGameDLL_CS/       # Git submodule: Reverse-engineered game DLL
│   └── yapb/               # Git submodule: YaPB bot (Yet Another PodBot)
├── scripts/                # Build scripts
│   ├── yapb_graph_dl.py    # Downloads bot navigation graphs at configure time
│   └── pack_extras.py      # Packages extras into extras.pk3
├── cmake/                  # CMake modules
│   └── LibraryNaming.cmake # Platform detection + library output naming
├── toolchains/             # Cross-compilation toolchains
│   └── i386-linux-gnu.cmake # Linux i386 cross-compile toolchain
├── android/                # Android build (Gradle)
└── .github/workflows/     # CI workflows
```

---

## Shared Code Pattern

Client and server share weapon and movement code. This is a fundamental architectural pattern — changes to shared code affect both sides simultaneously.

### `dlls/wpn_shared/*.cpp` — Shared Weapon Code

Compiled into **both** the client (`cl_dll/CMakeLists.txt` includes them) and the server. Contains weapon behavior logic used by both sides. Any weapon balance change here applies consistently across client and server.

### `pm_shared/*.cpp` — Shared Player Movement

Player movement prediction code compiled into both client and server. Ensures client-side movement prediction matches server-side validation, preventing desync between what the player sees and what the server calculates.

### `game_shared/*.cpp` — Shared Voice Code

Voice status code compiled into both client and server.

> **Critical:** When editing weapon behavior, changes in `dlls/wpn_shared/` affect both client and server. Weapon balance changes must be applied consistently to avoid client-server desync.

---

## Platform Detection System

`public/build.h` (274 lines) is the central compile-time platform detection header. All macros are defined to positive values when active, and left undefined when inactive.

### Apple Silicon (M1/M2/M3/M4) Detection Chain

```
__APPLE__ defined         → XASH_APPLE = 1
__aarch64__ || _M_ARM64   → XASH_64BIT = 1, XASH_ARM = 8, XASH_ARMv8 = 1
Little endian             → XASH_LITTLE_ENDIAN = 1
```

**Result:** `XASH_POSIX`, `XASH_APPLE`, `XASH_64BIT`, `XASH_ARM=8`, `XASH_ARMv8`, `XASH_LITTLE_ENDIAN`

### Intel Mac Detection Chain

```
__APPLE__                 → XASH_APPLE = 1
__x86_64__ || _M_X64      → XASH_AMD64 = 1, XASH_64BIT = 1
XASH_LITTLE_ENDIAN = 1
```

### Full Macro Categories

**Platform macros:**

| Macro | Platform |
|-------|----------|
| `XASH_WIN32` | Windows |
| `XASH_ANDROID` | Android |
| `XASH_LINUX` | Linux |
| `XASH_FREEBSD` | FreeBSD |
| `XASH_APPLE` | macOS |
| `XASH_IOS` | iOS |
| `XASH_NETBSD` | NetBSD |
| `XASH_OPENBSD` | OpenBSD |
| `XASH_HAIKU` | Haiku |
| `XASH_SERENITY` | SerenityOS |
| `XASH_IRIX` | IRIX |
| `XASH_NSWITCH` | Nintendo Switch |
| `XASH_PSVITA` | PS Vita |
| `XASH_WASI` | WASI |
| `XASH_SUNOS` | SunOS |
| `XASH_EMSCRIPTEN` | Emscripten |
| `XASH_DOS4GW` | DOS4GW |
| `XASH_TERMUX` | Termux |

**Architecture macros:**

| Macro | Architecture |
|-------|-------------|
| `XASH_64BIT` | 64-bit (any) |
| `XASH_AMD64` | x86_64 |
| `XASH_X86` | x86 (32-bit) |
| `XASH_ARM` | ARM (value 4-8 indicates version) |
| `XASH_MIPS` | MIPS |
| `XASH_PPC` | PowerPC |
| `XASH_E2K` | Elbrus 2000 |
| `XASH_RISCV` | RISC-V |
| `XASH_JS` | JavaScript |
| `XASH_WASM` | WebAssembly |

**ARM variant macros:**

`XASH_ARM_HARDFP`, `XASH_ARM_SOFTFP`, `XASH_ARMv4` through `XASH_ARMv8`

**RISC-V variant macros:**

`XASH_RISCV_DOUBLEFP`, `XASH_RISCV_SINGLEFP`, `XASH_RISCV_SOFTFP`

**Endianness macros:**

`XASH_LITTLE_ENDIAN`, `XASH_BIG_ENDIAN`

**Mobile macro:**

`XASH_MOBILE_PLATFORM` — auto-set for Android, iOS, Switch, Vita, and Sailfish.

---

## Library Naming System

`cmake/LibraryNaming.cmake` (222 lines) handles output library naming across platforms.

### How It Works

1. Uses `check_symbol_exists` against `build.h` macros at CMake configure time
2. Detects platform and sets `BUILDOS` variable
3. Detects architecture and sets `BUILDARCH` variable
4. Computes `POSTFIX` as `_{BUILDOS}_{BUILDARCH}` (BUILDOS is empty for most platforms)
5. `set_target_postfix(target)` macro renames output to `{target}{POSTFIX}` with no `lib` prefix

### Platform to BUILDOS Mapping

| Platform | BUILDOS |
|----------|---------|
| Windows, Linux, macOS, FreeBSD, etc. | `""` (empty) |
| Android | `"android"` |

### Architecture to BUILDARCH Mapping

| Architecture | BUILDARCH |
|-------------|-----------|
| x86 | `i386` |
| x86_64 | `amd64` |
| ARM64 | `arm64` |
| ARM32 hard-float | `armv7hf` |
| MIPS (little-endian) | `mipsel` |
| MIPS (big-endian) | `mips` |

### Output Examples

| Target | POSTFIX | Client Output |
|--------|---------|---------------|
| macOS ARM64 | `_arm64` | `client_arm64.dylib` |
| macOS Intel | `_amd64` | `client_amd64.dylib` |
| Linux amd64 | `_amd64` | `client_amd64.so` |
| Linux i386 | `_i386` | `client_i386.so` |
| Windows x86 | `""` (empty) | `client.dll` |
| PS Vita | `_psvita_armv7hf` | `client_psvita_armv7hf.so` |
| Android arm64 | `_android_arm64` | `libclient_android_arm64.so` |

> Android adds a `lib` prefix automatically — this is not controlled by `LibraryNaming.cmake`.

---

## Submodules

| Submodule | Path | Purpose |
|-----------|------|---------|
| YaPB | `3rdparty/yapb` | Yet Another PodBot — CS 1.6 bot with navigation graphs |
| ReGameDLL_CS | `3rdparty/ReGameDLL_CS` | Reverse-engineered game DLL (server-side game logic) |
| mainui_cpp | `3rdparty/mainui_cpp` | VGUI2 menu UI library (main menu, server browser, settings) |
| miniutl | `3rdparty/miniutl` | Mini utility library (containers, strings) |

`3rdparty/cs16client-extras/` is **not** a submodule — it is tracked directly in the repository. It contains bot databases, game assets, touch configurations, and user config defaults.

---

## Build System

### Root CMakeLists.txt (147 lines)

- Minimum CMake version: 3.10
- Python interpreter is **mandatory** at configure time (`find_package(Python COMPONENTS Interpreter REQUIRED)`)
- Three build options, all ON by default:
  - `BUILD_CLIENT` — client library (`cl_dll/`)
  - `BUILD_MAINUI` — menu UI library (`3rdparty/mainui_cpp/`)
  - `BUILD_SERVER` — server library (YaPB bot + ReGameDLL_CS). PS Vita sets this OFF.
- Install directories: `GAME_DIR="cstrike"`, client goes to `cl_dlls/`, server goes to `dlls/`
- CPack packaging: ZIP for Windows/macOS/PS Vita, TGZ for Linux

### Configure-Time Scripts

| Script | Lines | Purpose |
|--------|-------|---------|
| `scripts/yapb_graph_dl.py` | 67 | Downloads 25 bot navigation graphs from GitHub. Non-fatal on failure. |
| `scripts/pack_extras.py` | 19 | Creates `extras.pk3` using `ZIP_STORED` (no compression — the game engine reads PK3 files directly). |

### CMakePresets.json (164 lines, version 3)

- 14 presets across 5 platforms
- macOS presets use `gcc`/`g++` (GNU, not Apple Clang) with C++11
- CI presets use RelWithDebInfo
- Linux i386 uses `toolchains/i386-linux-gnu.cmake` for cross-compilation
- PS Vita sets `BUILD_SERVER=OFF`

### Android Build

Separate Gradle-based build system in `android/`. Built with `./gradlew assembleGitDebug`.

---

## CI Workflow

`.github/workflows/build.yml` defines a matrix build across all supported platforms:

- **Android**, **Windows** (x86/amd64), **Linux** (i386/amd64), **PS Vita**, **macOS** (arm64/x86_64)
- macOS runners: `macos-latest` (arm64) and `macos-15-intel` (x86_64)
- macOS dependencies: `brew install cmake gcc ninja fontconfig`
- Continuous releases are published from the `main` branch

---

## Code Style

`.clang-format` is present in the repository root with the following configuration:

- **Base style:** WebKit
- **Braces:** Allman (braces on their own line)
- **Indentation:** Tabs
- **Pointer alignment:** Right (`int *foo`)
- **Sort includes:** Disabled (`SortIncludes: false`)

### Code Defines

| Define | Purpose |
|--------|---------|
| `CLIENT_DLL` | Defined during client builds |
| `CLIENT_WEAPONS` | Enables client-side weapon prediction |
| `LINUX` / `_LINUX` | Defined on non-Windows platforms |
| `XASH_COMPAT=ON` | Forced for ReGameDLL_CS submodule integration |

### Non-MSVC Conventions

- `stricmp` is replaced with `strcasecmp`
- `-fms-extensions` is enabled for MSVC-compatible syntax

### Excluded Files

- `inputw32.cpp` — legacy Half-Life input code, excluded from all builds
- `ev_hldm.cpp` — legacy Half-Life event code, excluded from all builds

---

## Prerequisites

| Platform | Requirements |
|----------|-------------|
| All | CMake >= 3.10, Python interpreter |
| Linux i386 | `gcc-multilib`, `g++-multilib`, `libfontconfig-dev:i386` |
| macOS | `brew install cmake gcc ninja fontconfig` (uses GNU gcc/g++, not Apple Clang) |
| PS Vita | VitaSDK (`$VITASDK` environment variable) |
| Android | Android SDK + NDK, Gradle |
