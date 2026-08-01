# win-capture-audio

OBS Studio plugin that captures audio from a specific application rather than
the whole system. Fork of `bozbez/win-capture-audio`. C++23, CMake, Windows
only, MSVC.

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

**Prerequisites (none of these are currently satisfied in this checkout):**

- `git submodule update --init` — `deps/wil` (Microsoft WIL) is empty right now,
  and `target_include_directories` points into it, so the build cannot compile
  until this is run.
- CMake 3.20+ — not on PATH on this machine.
- libobs development files discoverable by `find_package(libobs REQUIRED)`.
  Typically an OBS source tree built from scratch, or an installed OBS SDK;
  point CMake at it with `-DCMAKE_PREFIX_PATH=<path-to-libobs>`.
- Inno Setup (`iscc`) only if building the installer.

The configure/build invocation has **not** been verified in this environment —
treat it as the standard OBS-plugin shape and confirm before relying on it:

```bash
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DCMAKE_PREFIX_PATH=<libobs>
```

Build `RelWithDebInfo` (not `Release`) when producing anything a user will
install: the `POST_BUILD` step only stages `release/` for those two
configurations, and only `RelWithDebInfo` copies the `.pdb`.

There is no test suite and no CI. Verification is manual: load the built
plugin into OBS and check the "Application Audio Output Capture" source.

## Generated files — never edit

`src/plugin-macros.generated.hpp` and `installer/installer.generated.iss` are
`configure_file` outputs, regenerated on every configure and gitignored via
`*.generated.*`. Edit the templates instead: `src/plugin-macros.hpp.in` and
`installer/installer.iss.in`. A PreToolUse hook blocks writes to the generated
pair.

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
  `.claude/skills/locale-sync/SKILL.md`. Five translated locales are already
  behind; that backlog predates current work.
- The version lives in exactly one place: `project(... VERSION x.y.z)` in
  `CMakeLists.txt`. See `.claude/skills/release-prep/SKILL.md`.
