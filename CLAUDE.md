# win-capture-audio

OBS Studio plugin that captures audio from a specific application rather than
the whole system. Based on `Ayrton09/win-capture-audio` (git remote `ayrton`), a
modernized fork of the unmaintained `bozbez/win-capture-audio`. Ayrton's history
shares no commits with bozbez's, so sync from `ayrton`, not from upstream.
C++23, CMake, Windows only, MSVC.

## How it works

`ActivateAudioInterfaceAsync` with `AUDIOCLIENT_PROCESS_LOOPBACK_PARAMS` — a
Windows API that is only officially documented for Windows 11 but works on
sufficiently updated Windows 10 2004+. This is the constraint behind most of
the plugin's support burden: "no audio captured" almost always means the user's
Windows is too old, not that the plugin is broken.

Roughly one thread per concern, each with its own `GetMessage` loop,
communicating by `PostThreadMessage`:

- `SessionMonitor` (singleton) — watches WASAPI devices and audio sessions,
  publishes session add/expire events to registered clients
- `AudioCaptureHelper` — one per captured PID; owns the `IAudioClient` loopback
  capture and forwards packets to every registered `Mixer`
- `Mixer` — one per OBS source; timestamp-aligns packets and submits to OBS
- `AudioCapture` — the OBS source itself; owns config, hotkeys, and properties

Objects and `AddRef`'d interfaces are routinely passed as `WPARAM` with
ownership transferring to the receiving thread, which frees them. When editing
any of this, read `.claude/agents/win-audio-reviewer.md` — it enumerates the
failure modes this design is prone to.

## Building

**Prerequisites (not yet satisfied on this machine):**

- `git submodule update --init` — `deps/wil` (Microsoft WIL) is empty in a fresh
  checkout and the build includes from it.
- CMake 3.24+ — not on PATH on this machine.
- Visual Studio 2022+ with the MSVC C++ workload, and an installed OBS Studio.
- Inno Setup (`iscc`) only if building the installer.

No libobs SDK is needed: if `find_package(libobs)` finds nothing, configure runs
`cmake/bootstrap-obs-sdk.ps1`, which builds headers + import library into
`deps/libobs-sdk` from the installed OBS (it downloads the matching obs-studio
headers). Pin the headers with `-DOBS_VERSION=<tag>`.

```bash
cmake -S . -B build -A x64
cmake --build build --config RelWithDebInfo
cmake --install build --config RelWithDebInfo   # -> %ProgramData%\obs-studio\plugins\win-capture-audio
```

Not yet verified in this environment.

The install layout is `%ProgramData%\obs-studio\plugins\<name>\{bin\64bit,data}`.
The legacy layout inside `C:\Program Files\obs-studio\` still loads and would
shadow it; the installer removes it, a bare `cmake --install` does not.

No test suite. CI (`.github/workflows/build.yml`) only builds and packages.
Verification is manual: load the plugin into OBS and check the "Application
Audio Output Capture" source.

## Generated files — never edit

`plugin-macros.generated.hpp`, `version.rc` and `installer.generated.iss` are
`configure_file` outputs written under `build/`, regenerated on every configure.
Edit the templates instead: `src/plugin-macros.hpp.in`, `src/version.rc.in` and
`installer/installer.iss.in`.

CMake also stamps the short commit hash into `GIT_HASH` at configure time, so
configure *after* committing if the build's reported hash matters.

## Conventions

- Formatting is `.clang-format` (tabs, 100 columns, custom brace wrapping). A
  PostToolUse hook runs `clang-format -i` on edited `.cpp`/`.hpp` — but only if
  clang-format is installed, which it currently is not
  (`winget install LLVM.LLVM`). Until then, match surrounding style by hand.
- The `TEXT_*` / `SETTING_*` macro block at the top of `src/audio-capture.hpp`
  is hand-aligned inside a `/* clang-format off */` guard. Preserve both the
  alignment and the guard.
- Every user-facing string goes through `obs_module_text`. Adding one means
  adding the key to `data/locale/en-GB.ini` (the fallback reference) — see
  `.claude/skills/locale-sync/SKILL.md`.
- The version lives in exactly one place: `project(... VERSION x.y.z)` in
  `CMakeLists.txt`. See `.claude/skills/release-prep/SKILL.md`.
