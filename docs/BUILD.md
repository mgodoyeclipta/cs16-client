# Building cs16-client

This guide covers building cs16-client, a reverse-engineered Counter-Strike 1.6 client built on the Xash3D FWGS engine. The project is CMake-based, uses 4 git submodules, and requires a Python interpreter at configure time.

## Prerequisites

### macOS Apple Silicon (arm64)

| Dependency | Install | Notes |
|---|---|---|
| CMake 3.21+ | `brew install cmake` | Required for presets |
| GNU GCC | `brew install gcc` | **NOT Apple Clang.** The project uses `-fms-extensions`, a GCC-only flag |
| Ninja | `brew install ninja` | Build generator |
| Fontconfig | `brew install fontconfig` | |
| Python 3 | macOS built-in or `brew install python3` | Required at configure time |
| Xcode CLI Tools | `xcode-select --install` | |

**Critical:** macOS ships `/usr/bin/gcc` which is actually Apple Clang in disguise. Homebrew installs GCC as versioned binaries (`gcc-15`, `g++-15`) because Apple Clang occupies the generic name. You must use the versioned names when configuring manually.

### macOS Intel (x86_64)

Same prerequisites as Apple Silicon. Use `x86_64` arch in presets.

### Linux amd64

```shell
sudo apt install cmake ninja-build gcc g++ python3 libfontconfig-dev
```

### Linux i386 (cross-compile)

Requires multilib support and 32-bit fontconfig:

```shell
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install cmake ninja-build gcc-multilib g++-multilib python3 libfontconfig-dev:i386
```

Uses toolchain file `toolchains/i386-linux-gnu.cmake`.

### Windows x86 / amd64

- Visual Studio with MSVC (`cl.exe`)
- CMake 3.21+, Ninja, Python 3
- Use the **Visual Studio Developer Command Prompt** for your target architecture

### PS Vita

- VitaSDK (set `$VITASDK` environment variable)
- CMake, Python 3
- `BUILD_SERVER=OFF` is forced, `MAINUI_USE_STB=ON` is set

## CMake Presets

| Preset | Platform | Build Type | Generator | Compiler |
|---|---|---|---|---|
| `macos-ci-arm64` | macOS arm64 | RelWithDebInfo | Ninja | gcc/g++ |
| `macos-ci-x86_64` | macOS x86_64 | RelWithDebInfo | Ninja | gcc/g++ |
| `linux-release-amd64` | Linux amd64 | Release | Ninja | default |
| `linux-release-i386` | Linux i386 | Release | Ninja | default |
| `linux-ci-amd64` | Linux amd64 | RelWithDebInfo | Ninja | default |
| `linux-ci-i386` | Linux i386 | RelWithDebInfo | Ninja | default |
| `win32-debug-amd64` | Windows amd64 | Debug | Ninja | MSVC cl |
| `win32-debug-x86` | Windows x86 | Debug | Ninja | MSVC cl |
| `win32-release-amd64` | Windows amd64 | Release | Ninja | MSVC cl |
| `win32-release-x86` | Windows x86 | Release | Ninja | MSVC cl |
| `win32-ci-amd64` | Windows amd64 | RelWithDebInfo | Ninja | MSVC cl |
| `win32-ci-x86` | Windows x86 | RelWithDebInfo | Ninja | MSVC cl |
| `psvita-debug` | PS Vita | Debug | Makefiles | VitaSDK |
| `psvita-release` | PS Vita | Release | Makefiles | VitaSDK |

## Build Options

| Option | Default | Description |
|---|---|---|
| `BUILD_CLIENT` | ON | Client library (`cl_dll/`) |
| `BUILD_MAINUI` | ON | Menu UI (`3rdparty/mainui_cpp/`) |
| `BUILD_SERVER` | ON | Server (YaPB bot + ReGameDLL_CS). Forced OFF on PS Vita |

Pass options during configure, e.g.: `cmake -B build -DBUILD_SERVER=OFF ...`

## Step-by-Step Build Instructions

### macOS Apple Silicon

The `macos-ci-arm64` preset hardcodes `CMAKE_C_COMPILER=gcc`, which resolves to Apple Clang on local dev machines. **Use manual configure instead:**

```shell
# 1. Clone with submodules
git clone --recursive https://github.com/Velaron/cs16-client
cd cs16-client

# 2. Install prerequisites
brew install cmake gcc ninja fontconfig

# 3. Configure (manual — avoids Apple Clang issue)
cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=gcc-15 \
  -DCMAKE_CXX_COMPILER=g++-15 \
  -DCMAKE_CXX_STANDARD=11 \
  -DCMAKE_CXX_STANDARD_REQUIRED=ON

# 4. Build
cmake --build build

# 5. Install to Xash3D
cmake --install build --prefix /path/to/xash3d-fwgs
```

Replace `gcc-15`/`g++-15` with your installed version. Check with `brew info gcc`.

### macOS Intel

Same as Apple Silicon, but the preset may work if `/opt/homebrew/bin/gcc` exists as a symlink:

```shell
cmake --preset macos-ci-x86_64
cmake --build build
```

If the preset fails with an Apple Clang detection error, fall back to manual configure with `gcc-15`/`g++-15`.

### Linux amd64

```shell
sudo apt install cmake ninja-build gcc g++ python3 libfontconfig-dev

cmake --preset linux-release-amd64
cmake --build build

sudo cmake --install build --prefix /usr/local/share/xash3d
```

### Linux i386 (cross-compile)

```shell
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install cmake ninja-build gcc-multilib g++-multilib python3 libfontconfig-dev:i386

cmake --preset linux-release-i386
cmake --build build
```

### Windows

Open the **Visual Studio Developer Command Prompt** for your target architecture, then:

```shell
cmake --preset win32-release-amd64
cmake --build build
```

### PS Vita

```shell
export VITASDK=/usr/local/vitasdk

cmake --preset psvita-release
cmake --build build
```

### Android

The Android build uses Gradle separately from CMake:

```shell
cd android
./gradlew assembleGitDebug
```

## Output Libraries

The build system (via `cmake/LibraryNaming.cmake`) produces platform-postfixed library names with no `lib` prefix (except Android):

| Platform | Client | Server | Menu UI | Bot |
|---|---|---|---|---|
| macOS arm64 | `client_arm64.dylib` | `cs_arm64.dylib` | `menu_arm64.dylib` | `yapb_arm64.dylib` |
| macOS amd64 | `client_amd64.dylib` | `cs_amd64.dylib` | `menu_amd64.dylib` | `yapb_amd64.dylib` |
| Linux amd64 | `client_amd64.so` | `cs_amd64.so` | `menu_amd64.so` | `yapb_amd64.so` |
| Linux i386 | `client_i386.so` | `cs_i386.so` | `menu_i386.so` | `yapb_i386.so` |
| Windows x86 | `client.dll` | `cs.dll` | `menu.dll` | `yapb.dll` |
| Windows amd64 | `client_amd64.dll` | `cs_amd64.dll` | `menu_amd64.dll` | `yapb_amd64.dll` |

### Install Layout

| Path | Contents |
|---|---|
| `<prefix>/cstrike/cl_dlls/` | Client + menu library |
| `<prefix>/cstrike/dlls/` | Server + bot library |
| `<prefix>/cstrike/extras.pk3` | Extras pack (bot configs, assets) |

## Configure-Time Scripts

Two Python scripts run during CMake configure:

| Script | Purpose |
|---|---|
| `scripts/yapb_graph_dl.py` | Downloads 25 bot navigation graph files for official CS 1.6 maps. Requires internet. Non-fatal on failure (bots will lack nav meshes). |
| `scripts/pack_extras.py` | Creates `extras.pk3` ZIP containing `cs16client-extras/`, `yapb/cfg/`, and bot graphs. |

## Packaging

```shell
cd build
cpack --config CPackConfig.cmake
```

| Platform | Format | Filename |
|---|---|---|
| Windows | ZIP | `CS16Client-Windows-{arch}.zip` |
| macOS | ZIP | `CS16Client-macOS-{arch}.zip` |
| Linux | TGZ | `CS16Client-Linux-{arch}.tar.gz` |
| PS Vita | ZIP | `CS16Client-PSVita.zip` |

## Troubleshooting

### "The C compiler identification is Apple Clang" / `-fms-extensions` error

The preset uses `gcc`/`g++` which resolves to Apple Clang on local macOS. Fix: use manual configure with the versioned compiler names (`gcc-15`/`g++-15`).

### Submodule checkout failed

Clone with `--recursive`. If already cloned without it:

```shell
git submodule update --init --recursive
```

### Python not found

CMake requires Python at configure time. Install it:

- macOS: `brew install python3`
- Linux: `sudo apt install python3`

### Fontconfig linking error (Linux)

Install the dev package:

```shell
sudo apt install libfontconfig-dev        # amd64
sudo apt install libfontconfig-dev:i386   # i386
```

### Warnings about `ALIGN`, `M_PI`, `RAD2DEG` redefinition

These are benign macro redefinitions from SDK headers conflicting with system headers. They are not errors and the build succeeds normally.

### "Library postfix: _arm64" not shown in configure output

Platform detection failed. Check that `public/build.h` is accessible and the compiler targets arm64.

### GCC version mismatch

Check your installed version with `brew info gcc`. If it is not version 15, update the `-DCMAKE_C_COMPILER=gcc-XX` and `-DCMAKE_CXX_COMPILER=g++-XX` flags accordingly.
