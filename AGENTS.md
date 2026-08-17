# AGENTS.md

Ironwail is a high-performance Quake 1 engine: a fork of [QuakeSpasm](https://sourceforge.net/projects/quakespasm/) that moves work from CPU to GPU (culling, lightmap updates, instancing, compute shaders, bindless textures) and decouples the renderer from the game logic server (code from QSS via vkQuake). Written in C (gnu11) with a C++14 subset for OpenVR.

## Repository layout

- `Quake/` — all engine source (C). This is where 99% of changes land.
- `Quake/Makefile` — Unix make build (used by Linux/macOS CI).
- `CMakeLists.txt` — CMake build (Windows, cross-platform dev).
- `Windows/VisualStudio/ironwail.sln` — MSVC build (used by Windows CI).
- `Misc/pak/` — sources for `Quake/ironwail.pak` (rebuild with `mkpak.sh`).
- `Quake/vr.c`, `Quake/vr_menu.c` — OpenVR support, **compiled as C++14** (`extern "C"` on the Quake includes).
- `Quake/openvr.h` — vendored OpenVR header.
- `build/` — build output, gitignored. `Quake/*.o`, `*.d`, `ironwail` binary also gitignored — never commit them.
- `Quake/ironwail.pak` — checked in; CMake copies it to `id1/` at build time.

## Build & verify

There is **no test suite**. Verification is: clean compile + manual playtesting in a VR-capable HMD (Quest 3 via SteamVR) for VR changes.

Linux (matches CI):
```
sudo apt install libcurl4-openssl-dev libmpg123-dev libsdl2-dev libvorbis-dev
make --jobs=3 --keep-going --directory=Quake CC=clang   # or gcc
```
Run: `./Quake/ironwail` (needs a Quake/Steam id1 alongside, or the copied pak).

CMake (optional, used on Windows):
```
cmake -B build && cmake --build build --config Release
```
Requires SDL2 + OpenGL; codecs (mpg123/mad, opusfile, flac, vorbisfile, mikmod/modplug/xmp), libcurl, and openvr (pkg-config) are all optional — the build degrades gracefully and warns if openvr is missing.

Windows: MSBuild `Windows/VisualStudio/ironwail.sln` (Release, x64/Win32).

Always run the build matching the target platform before claiming a change works.

## Code style

- Tabs for indentation, 4-width (`.editorconfig`).
- QuakeSpasm conventions: `===============` comment header above each function, function names in the `MODULE_SubName` style, `qboolean`/`cfloat`/`vec3_t` types, `Cvar_RegisterVariable`/`DEFINE_CVAR` for cvars.
- C is gnu11; keep VR files compilable as C++14 — no C-only idioms (e.g. implicit int, variable-length arrays are fine in C, but prefer fixed arrays in VR code).
- Match surrounding style exactly; no drive-by reformatting or refactors.
- Cvars: user-facing tunables are `CVAR_ARCHIVE`; debug-only flags use `vr_debug_*` and gate `Sys_Printf` output.

## Architecture notes (VR)

VR was copied from quakespasm-openvr and is still being brought up (branch `fix-projection`; as of last commit: eye poses correct, HMD movement not yet working). Key pipeline and invariants:

- Enable path: `vr_enabled` cvar → `VR_Enabled_f` callback (`Quake/vr.c`) → `VR_Enable()`. `vid_restart` calls `VID_Restart` which re-runs `VR_Enable()` (`Quake/gl_vidsdl.c:740`); `Host_Shutdown` calls `VID_VR_Shutdown()`.
- Frame flow: `V_CalcRefdef` (`Quake/view.c`) builds per-eye refdefs → `R_SetFrustum` (`Quake/gl_rmain.c`) → `R_SetupGL` (early-returns when `vr_enabled` — VR owns FBOs/viewport) → per-eye: `VR_UpdateScreenContent` → `VR_SetMatrices` + scene render → `VR_Draw2D` for 2D overlay (sbar, crosshair). `SCR_UpdateScreen` routes to `VR_UpdateScreenContent`/`VR_Draw2D` when VR is on (`Quake/gl_screen.c`).
- **Projection invariant (do not "fix" back):** OpenVR's projection matrices are standard OpenGL format and are incompatible with ironwail's `ExtractFrustumPlane`. `R_SetFrustum` must always build `r_matproj` via `GL_FrustumMatrix` for culling; the actual VR projection is applied at render time by `VR_SetMatrices`. Culling uses a wider-than-VR frustum.
- **Coordinate conventions:** OpenVR is Y-up right-handed; Quake is Z-up. The head-origin mapping is `(x, y, z)_Quake = (v[2], v[0], v[1])_OpenVR`. Eye-to-head transforms from OpenVR are used un-negated, rotated by the head quaternion and by `vrYaw`. `eyes[i].orientation` stores the **raw OpenVR quaternion** — `QuatToYawPitchRoll` expects that format, so don't convert it to Quake quat first.
- Debugging: `vr_debug_pose` cvar prints pose data at 4 Hz; `Sys_Printf` debug in `V_CalcRefdef` is rate-limited by `eye_count`.

## Pitfalls

- Don't feed OpenVR matrices into frustum extraction (see invariant above) — it silently produces wrong culling (missing/extra geometry).
- VR code paths must also work when VR is off; guard with `vr_enabled.value`.
- `R_SetupGL` skipping framebuffer setup is intentional for VR — don't re-add glViewport/FBO binding there.
- The renderer is decoupled from the game server (netcode-driven); don't assume one simulation tick per rendered frame.
- macOS builds in CI but is not a supported runtime target (needs OpenGL 4.3).
- The working tree currently carries uncommitted VR debugging changes on `fix-projection`; don't commit or revert them without checking with the user.

## License

GPLv2 (`LICENSE.txt`). Keep any new code compatible.
