# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An unofficial Android (arm64) port of the original Xbox game *Crash Twinsanity* (title 56550026), built from the Android host of [Black-Android](https://github.com/RanggaCasper/Black-Android) and made with the [XboxXBE-Recomp](https://github.com/TGP482/XboxXBE-Recomp) SDK. The game's x86 code in `default.xbe` is statically recompiled to C++ on a Windows PC. That C++ is then built for Android together with the SDK's portable runtime and this repo's host code. Drawing goes through Vulkan 1.1, and windowing, input and audio through SDL3. `assets/default.xbe` may be committed for the CI build; other game files in `iso/` and recompiled output in `generated/` are gitignored.

## Build

Everything runs on Windows. Requirements: Visual Studio 2022 C++ tools, plus Android SDK Platform 34, NDK `28.2.13676358` and CMake `3.31.6`. Java 17–21 is needed for Gradle 8.7 (`.vscode/settings.json` points to JDK 17). Python is used only to pull the xbe out of an ISO.

- `build.bat` runs the full pipeline:
  1. Checks the tools.
  2. Gets `assets/default.xbe`. If it is missing, `python -I tools/xdvdfs.py <iso> default.xbe assets/default.xbe` extracts it from `assets/*.iso` or `iso/*.iso`.
  3. Builds the recompiler (`tools/CMakeLists.txt` → `out/build/android-tools/xbe_recomp.exe` and `xbe_symbols.exe`) with clang-cl.
  4. Writes the `[symbols]` table into `config/game.toml` with `xbe_symbols`, only if the file doesn't have one yet.
  5. Runs `xbe_recomp config/game.toml --xdk <sdk>/xdk` → `generated/game/`.
  6. Runs `gradlew.bat assembleRelease`.
- To rebuild only the APK after editing host code (no re-recompile): `gradlew.bat assembleRelease`. Output: `app/build/outputs/apk/release/app-release.apk`, signed with the debug key.
- After editing `config/game.toml` (hooks, symbols, ABI, partitions), the recompile step has to run again. Run `build.bat`, or run the two `out/build/android-tools` commands by hand.
- `gradlew.bat assembleDebug` still builds the native code as Release: unoptimized recompiled code is too slow to run.
- Install: `adb install -r app/build/outputs/apk/release/app-release.apk`. On the device, settings (`twinsanity.toml`, `android.toml`, `layout.toml`), saves (`hdd`), logs and the shader cache are in `Android/data/com.port.crashtwinsanity/files`.
- The first native build compiles 128 partitions of recompiled code and takes 30–90 minutes.
- `-PxbeGenerated=<dir> -PxbeImage=<file>` makes Gradle build recompiled code from somewhere other than `generated/game`.

There are no tests or linters in this repo.

## CI/CD

`.github/workflows/build.yml` runs on `windows-latest` for each `v*` tag and by hand (`workflow_dispatch`). It uses `assets/default.xbe` when it is committed, otherwise extracts `default.xbe` from the first `assets/*.iso`, runs `./build.bat`, and uploads `config/game.toml` (with the generated symbol table) and `app/build/outputs/apk/release/*.apk` as artifacts. For a tag it also publishes a release with `ci/release-notes.md` and the commits since the previous tag. `app/build.gradle` signs with a release key from the `BLACK_KEYSTORE*` environment variables when they are set (the workflow sets them from the `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS` and `KEY_PASSWORD` repository secrets), else the debug key. `UpdateChecker.REPO` names the repository whose latest release the app checks; it is empty in this port until a release repository exists.

## Where the SDK comes from

The SDK is vendored in `sdk/` (upstream commit in `sdk/UPSTREAM_COMMIT`), changed in one place: `ExQueryNonVolatileSetting` takes XC_LANGUAGE from a weak `app::ConsoleLanguage()`, which `src/game/game.cpp` defines. Both `CMakeLists.txt` files look for the SDK in this order: `sdk/` in the repo, then `../XboxXBE-Recomp` beside the repo. If neither exists, they FetchContent the SDK's `main` branch. A sibling checkout lets you read or change the runtime (`<sdk>/runtime/{app,core,kernel,xdk,gpu,audio}`), whose headers (`app/app.h`, `kernel/kernel.h`, `xdk/hle.h`, `core/input.h`, ...) the host code includes. Gradle's `fetchSDL` task shallow-clones SDL `release-3.4.16` into `third_party/SDL`. Its Java sources go into the APK and its C side becomes `libSDL3.so`. glslang, toml++ and imgui are fetched by CMake.

## Architecture

The root `CMakeLists.txt` builds one shared library, `libmain.so`, which SDL's activity loads. It is made from:

- **`generated/game/`**: recompiler output (`x86_recomp.N.cpp` for N < `partitions`, `x86_func_mapping.cpp`, `image_symbols.h`). Compiled with `-O2 -w`. Never edit it by hand. Change `config/game.toml` or the host code and recompile instead.
- **The SDK runtime** (`<sdk>/runtime/*/*.cpp`), without the Windows-only parts and the parts this repo replaces. The regex filter in `CMakeLists.txt` drops `app/`, `core/input.cpp`, `gpu/d3d12`, `gpu/render_thread`, `audio/xaudio2`, `xdk/dsound` and `xdk/d3d8`.
- **`src/`**, the game-specific host, organized as `src/<area>/*.cpp` and globbed by CMake:
  - `src/game/`: game hooks and patches. `game.cpp` has the language glue (`app::ConsoleLanguage()`) and the small host-facing helpers; no hook is installed for Crash Twinsanity. Shaolin Monks' `FrameSkip` (0x0CB800) and `WidenAspect` (0x12432E) were tied to that image's addresses and were removed. `cheats.cpp` still holds the Shaolin Monks addresses but `ApplyCheats()` returns before using them (`kAddressesKnown`), and the cheat rows are hidden from the settings panel. Add new hooks or cheats only after finding the matching addresses in the Crash Twinsanity image.
  - `src/d3d/`: the title's Direct3D 8 (XDK 5849) entry points. `Direct3D_CreateDevice` also sets the device fields the library's init would (FVF shader, vertical blank event, display flags) and starts the vertical blank thread, which Black did from a hook. `render_thread.cpp` records D3D calls on the game's render thread and replays them on a dedicated Vulkan thread. Vertex, texture and surface memory is read at draw time, the way the console GPU reads it, so XDK fences, busy checks and locks wait on that replay.
  - `src/sound/`: DirectSound reimplemented (buffers, streams, ADPCM decode, mixer, reverb).
  - `src/common/`: `game_info.cpp` defines `app::kGame`, the struct that describes the game to the runtime (title, settings file, option bindings, load/save/init, pad and rumble filters, UI callback). Also here: the ImGui settings panel (`menu.cpp`, opened by the touch gear or F10), `options.h` (`game::Options`, saved to `twinsanity.toml`; most fields are Black's and unused) and translations.
- **`native/`**, the Android host. `main.cpp` runs SDL's window and event loop, the app lifecycle and foreground/surface handling. `vulkan.cpp` is the Vulkan backend, and `vk_shader.cpp` compiles the SDK's HLSL shader translations to SPIR-V with glslang. The rest: `touch.cpp` (touch controls and layout editor, plus keyboard and mouse), `gyro.cpp`, `input.cpp` (controllers, and the settings panel's CONTROLLER page bindings, saved as `[pad]` in `twinsanity.toml`, laid over SDL's mapping), `audio.cpp`, `ui.cpp` (ImGui), and `game_files.cpp`, which finds the game files or ISO and checks the xbe's SHA1 against `kImageSha1`; an empty `kImageSha1` accepts the image without a mismatch. `host.h` declares the interfaces these parts share. `native/compat/crt.h` is force-included into all host sources (not the generated ones).
- **`app/`**: Gradle module. `CrashTwinsanityActivity.java` (package `com.port.crashtwinsanity`) extends `SDLActivity`. It asks for all-files access (the storage permission on Android 10) and creates `/storage/emulated/0/CrashTwinsanity` and `files/assets`, which are where the game files are looked for, along with a `CrashTwinsanity` folder at the top of each SD card. `UpdateChecker.java` is off while its `REPO` is empty.

### How recompiled code reaches the host

`config/game.toml` is the recompiler's input:
- `[xbe]`: path, entry point, XDK version, output dir, `partitions` (CMake reads this number to list the generated sources), and code sections.
- `[symbols]` / `[data]`: addresses of library functions and globals in the image, 0 where the image has none. Host code refers to them through `SYM(Name)` from `image_symbols.h`.
- `[abi]`: register-argument conventions for link-time-code-generated XDK functions.
- `[[hook]]`: entries `at = <x86 address>`, `call = "<Name>"`. The recompiler emits a call at that address to a host function `void Name(X86Context& ctx, uint8_t* base)`, defined in `src/game/`. To add a hook, add the `[[hook]]` entry, define the function, and recompile.

Host code reads and writes guest memory through the kernel helpers (`HostPtr(va)`, `R32`, ...) and guest registers through `X86Context`.

### Supported game versions

The build targets the original Xbox release of Crash Twinsanity (title `56550026`). It recompiles whichever `assets/default.xbe` it is run with and CMake records that image's SHA-1; a different image is offered as "another release", and an empty `kImageSha1` disables that check. Images that differ beyond the executable may need `config/game.toml` adjustments.

### Notes on this image

- No `[[hook]]` entries and no active cheats: Shaolin Monks' `FrameSkip` (0x0CB800), `WidenAspect` (0x12432E) and cheat addresses were specific to that image. Add new hooks or cheats only after locating the matching addresses in Crash Twinsanity.
- `[symbols]`, `[data]`, `[abi]` and `[functions]` are written by `xbe_symbols` on the first build; the committed `config/game.toml` intentionally has no symbol table so `build.bat` regenerates it.
- The touch controls (`native/touch.cpp`) are a whole Xbox pad: no game hook tells menus from play, so `input::Play()` is always true, and a drag on the right is the right stick.

## Conventions

- Native code is C++20 and is compiled with `-fsigned-char -fno-strict-aliasing`, matching x86 semantics.
- Only arm64-v8a is built, minSdk 29 (`native/compat/crt.h` supplies `memfd_create`, which bionic has from API 30). Libraries are linked with 16 KB page alignment.
- Comment style is terse, lowercase, and explains *why*, as in the existing files.
