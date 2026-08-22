# AGENTS.md

Ironwail is a high-performance Quake 1 engine: a fork of [QuakeSpasm](https://sourceforge.net/projects/quakespasm/) that moves work from CPU to GPU (culling, lightmap updates, instancing, compute shaders, bindless textures) and decouples the renderer from the game logic server (code from QSS via vkQuake). Written in C (gnu11) with a C++14 subset for OpenVR.

## Repository layout

- `Quake/` — all engine source (C). This is where 99% of changes land.
- `Quake/Makefile` — Unix make build (Linux/macOS CI). **Not VR-aware** — see Build & verify.
- `CMakeLists.txt` — CMake build; the only build that compiles the VR code.
- `Windows/VisualStudio/ironwail.sln` — MSVC build (Windows CI). **Not VR-aware** — see Build & verify.
- `Misc/pak/` — sources for `Quake/ironwail.pak` (rebuild with `mkpak.sh`).
- `Quake/vr.c`, `Quake/vr_menu.c` — OpenVR support, **compiled as C++14** (`extern "C"` on the Quake includes).
- `Quake/openvr.h` — vendored OpenVR header (runtime symbols come from `libopenvr_api`).
- `build/` — build output, gitignored. `Quake/*.o`, `*.d`, `ironwail` binary also gitignored — never commit them.
- `Quake/ironwail.pak` — checked in; CMake copies it to `id1/` at build time.

## Build & verify

There is **no test suite**. Verification is: clean compile + manual playtesting in a VR-capable HMD (Quest 3 via SteamVR) for VR changes.

On the VR branch only the **CMake build includes the VR sources**. `Quake/Makefile` and `Windows/VisualStudio/ironwail.vcxproj` were never updated (no `vr.o`/`vr_menu.o`, no openvr link) and **fail to link on this branch**, since VR symbols are referenced unconditionally from core files (gl_screen.c, gl_rmain.c, host.c, gl_vidsdl.c, view.c, cl_main.c, sv_phys.c).

CMake (the build to use on this branch):
```
cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build --parallel
```
Requires SDL2 + OpenGL. Codecs (mpg123/mad, opusfile, flac, vorbisfile, mikmod/modplug/xmp) and libcurl are optional. OpenVR is detected via pkg-config with a `find_library(openvr_api)` fallback: the vendored `Quake/openvr.h` is enough to *compile*, but *linking* needs the OpenVR runtime library (the SteamVR package) — if it is missing CMake only warns at configure time and the link fails. `vr.c`/`vr_menu.c` are compiled with `-std=c++14`; the C++ runtime is supplied automatically (CMake uses the C++ linker driver because the target has C++ sources) — do not add an explicit `stdc++` link.

Run: `./build/ironwail` (needs a Quake/Steam id1 alongside, or the copied pak).

The non-VR builds still used by CI, for reference (broken on this branch until the Makefile/vcxproj are updated):
```
sudo apt install libcurl4-openssl-dev libmpg123-dev libsdl2-dev libvorbis-dev
make --jobs=3 --keep-going --directory=Quake CC=clang   # or gcc
```
Windows: MSBuild `Windows/VisualStudio/ironwail.sln` (Release, x64/Win32).

Always run the build matching the target platform before claiming a change works.

## Code style

- Tabs for indentation, 4-width (`.editorconfig`).
- QuakeSpasm conventions: `===============` comment header above each function, function names in the `MODULE_SubName` style, `qboolean`/`cfloat`/`vec3_t` types, `Cvar_RegisterVariable`/`DEFINE_CVAR` for cvars.
- C is gnu11; keep VR files compilable as C++14 — no C-only idioms (e.g. implicit int, variable-length arrays are fine in C, but prefer fixed arrays in VR code).
- Match surrounding style exactly; no drive-by reformatting or refactors.
- Cvars: user-facing tunables are `CVAR_ARCHIVE`; debug-only flags use `vr_debug_*` and gate `Sys_Printf` output.

## Architecture notes (VR)

VR was ported from quakespasm-openvr (branch `fix/vr-projection`). `Quake/vr_menu.c/h` and `Quake/openvr.h` are byte-identical to the reference; `Quake/vr.c` is largely the reference code with shims where the ironwail API differs: a local `SCR_UpdateScreenContent()` wrapper (calls `V_UpdateBlend` + `V_RenderView` — ironwail has no separate content/2D split) and `GLSLGamma_GammaCorrect()` → `GL_PostProcess()`.

Current state: per-eye asymmetric projection, controller aiming, weapons anchored at the controller, room-scale movement (local), and the 2D HUD/menu as a 3D billboard all work. See Known open issues below for what is still broken.

Key pipeline and invariants:

- Enable path: `vr_enabled` cvar → `VR_Enabled_f` callback → `VR_Enable()` (`Quake/vr.c`). `Host_Shutdown` → `VID_VR_Shutdown()`. `vid_restart` → `VID_Restart` → `VID_VR_Disable()` + `VR_Enable()` only when `vr_enabled.value` (`Quake/gl_vidsdl.c:738-741`). Note: `VID_VR_Init` (called from `R_Init`) force-sets `vr_enabled 1` at startup — the `-vr` command-line check is commented out (debug leftover, see open issues).
- Frame flow: `SCR_UpdateScreen` (`Quake/gl_screen.c`) routes — when `vr_enabled && !con_forcedup`: `VR_UpdateScreenContent()` (per-eye render + submit), whose final step blits the left eye to the window as the mirror (no separate 2D pass for the mirror); otherwise the flat desktop path (`V_UpdateBlend` + `V_RenderView` + 2D). With the console forced up in VR the desktop path runs and the HMD shows the last submitted frame.
- `VR_UpdateScreenContent` (vr.c:1147): `WaitGetPoses` → head/eye/controller state (the aim-mode switch writes `cl.viewangles`/`cl.aimangles`; controller state goes to `cl.handpos[]`/`cl.handrot[]`) → per eye: compute `vr_viewOffset` (per-eye head offset in Quake units, rotated by `viewangles[YAW] − eye yaw`) and `RenderScreenForCurrentEye_OVR()`.
- `RenderScreenForCurrentEye_OVR` (vr.c:1024): bind the eye's (MSAA) FBO + viewport, set `r_refdef.fov_x/y` to the per-eye values, render the scene via the local `SCR_UpdateScreenContent()` wrapper (so `V_CalcRefdef` → `R_SetFrustum` → `R_SetupGL` run per eye; `R_SetupGL` early-returns when `vr_enabled` — VR owns FBOs/viewport), MSAA resolve, optional per-eye underwater warp into a shared non-MSAA composite FBO (`R_VRWarpScaleView`, gl_rmain.c), `VR_Draw2D()`, gamma via `GLSLGamma_GammaCorrect()` → `GL_PostProcess()` (`VR_HandleGammaCorrect()` rebinds the target FBO; when postprocess is needed the pass runs into a scratch FBO and samples `vr_postprocess_tex` — the eye image — instead of the desktop composite), `VRCompositor->Submit()`. After both eyes: blit the left eye to the window (mirror).
- **Projection (supersedes the old "wider cull frustum" invariant):** OpenVR's projection matrices are standard OpenGL format and are incompatible with ironwail's `ExtractFrustumPlane`. `VR_BuildProjectionMatrix()` (vr.c:1551) converts the driver's `GetProjectionMatrix` into ironwail's matrix convention (column-major; forward=+X, left=+Y, up=+Z; clip.x = −w·y, clip.w = x) including the per-eye asymmetric frustum terms; for a symmetric frustum it reduces exactly to `GL_FrustumMatrix`. When `vr_enabled`, `R_SetFrustum` (gl_rmain.c:844) builds `r_matproj` with it, so culling (`ExtractFrustumPlane`), GPU light clustering (`transposed_proj`), and rendering (`r_matviewproj` UBO) all use the same per-eye matrix — cull always matches render. Do not feed raw OpenVR matrices into frustum extraction or the UBO, and do not replace `r_matproj` with the symmetric `GL_FrustumMatrix` in VR (that crops/stretches the asymmetric FOV at the edges). Extents come from `GetProjectionMatrix`, not `GetProjectionRaw` — the raw tangents are unreliable (some drivers, e.g. Quest via Steam Link, return up/down swapped).
- **Coordinate conventions:** OpenVR is Y-up right-handed; Quake is Z-up. The head-origin mapping is `(x, y, z)_Quake = (v[2], v[0], v[1])_OpenVR`. Eye-to-head transforms from OpenVR are used un-negated, rotated by the head quaternion and by `vrYaw`. `eyes[i].orientation` stores the **raw OpenVR quaternion** — `QuatToYawPitchRoll` expects that format, so don't convert it to Quake quat first.
- **Head is pinned to the player origin:** HMD translation does not move the eye directly; room-scale movement is applied to the player entity in `SV_Physics_Client` (`Quake/sv_phys.c`, `vr_room_scale_move` velocity hack). This is local-server only (it does not move the player on a remote MP server) and the magnitude is derived from `host_frametime` (render frame rate, not tick rate) — approximate by design (same as the reference).
- **Eye offset:** `vr_viewOffset` is added to `r_refdef.vieworg` in `V_CalcRefdef` (view.c), and the view matrix built in `R_SetFrustum` translates by `vieworg` — the per-eye offset is already baked in, so there is no separate eye-view matrix.
- **2D overlay:** the reference places HUD/menus through the fixed-function modelview + eye GL_PROJECTION; the ironwail GUI is shader-driven, so `VR_Draw2D` (vr.c:1768) builds `vr_gui_matrix = per-eye ViewProj × billboard model` (billboard 48 units in front of the eye, aspect = `guiheight/guiwidth`, size scaled by `vr_menu_scale`) and `Draw_Flush` (gl_draw.c:613) uploads it as the GUI shader's `Matrix` uniform, so the HUD converges per eye. The desktop path uses identity (`GL_Set2D` resets `vr_gui_matrix_valid`). `VR_DrawSbar` simply draws `Sbar_Draw()` through the same canvas/billboard (the sbar intentionally renders on the menu billboard, not controller-anchored).
- **Alpha/OIT:** `R_GetEffectiveAlphaMode` returns `ALPHAMODE_SORTED` in VR (no OIT FBO; translucent water/entities/particles are sorted-alpha blended straight into the eye FBO). The flat underwater warp `R_WarpScaleView` early-returns in VR (per-eye warp instead).
- **View model:** in controller aim mode the weapon is placed at `cl.handpos[1]` (view.c) and is never frustum-culled in VR (r_alias.c); the `ZRANGE_VIEWMODEL` depth hack is skipped. `Mod_Weapon` (vr.c) applies per-weapon scale/offset from the `vr_wofs_*` cvars; `original_scale`/`original_scale_origin` are saved at alias/MD5 load time (gl_model.c). `CL_SendMove` sends `cl.aimangles` (controller direction) instead of `cl.viewangles` when `vr_enabled` (cl_input.c).
- Debugging: `vr_debug_pose` cvar (default 0, `CVAR_NONE`, console-only — deliberately not in the VR menu) gates all pose/quat/view dumps and the per-file "DEBUG" prints.

## Known open issues (VR)

- **`VID_VR_Init` force-enables VR at startup** — the `COM_CheckParm("-vr")` gate is commented out and `Cvar_SetQuick(&vr_enabled, "1")` always runs (vr.c:902). Restore the gate (inherited from the reference, which has the same commented-out check).
- **Build systems:** `Quake/Makefile` and `ironwail.vcxproj` lack the VR sources and openvr link (link fails on this branch).
- **Parity/cosmetic gaps vs quakespasm-openvr:** the mirror window shows the pre-warp/pre-gamma left eye (postprocess runs into a scratch FBO, so the mirror never sees the gamma'd image); room-scale movement is local-server only.

## Pitfalls

- Don't feed raw OpenVR matrices into frustum extraction or the `r_matviewproj` UBO (see projection note above) — it silently produces wrong culling (missing/extra geometry). Culling must use the same per-eye matrix as rendering.
- VR code paths must also work when VR is off; guard with `vr_enabled.value`.
- `R_SetupGL` skipping framebuffer setup is intentional for VR — don't re-add glViewport/FBO binding there.
- The renderer is decoupled from the game server (netcode-driven); don't assume one simulation tick per rendered frame.
- macOS builds in CI but is not a supported runtime target (needs OpenGL 4.3).

## License

GPLv2 (`LICENSE.txt`). Keep any new code compatible.
