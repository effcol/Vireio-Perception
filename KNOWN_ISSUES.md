# Known issues: v5 modernisation branch

This branch makes Vireio Perception build on a current toolchain (MSVC v143/v145,
C++20, Windows 10 SDK) without the June 2010 DirectX SDK. It compiles and links.
It has **not** been run against a game, and a read-through of the code found the
problems below. They are listed roughly in the order they need fixing.

Paths are relative to the repository root. `DxProxy/` means
`Perception_v3/DxProxy/DxProxy/`.

## Perception_v3 (D3D9 proxy)

### Blocks any use

- [ ] **Proxy only activates while `Perception_Win32.exe` / `Perception_x64.exe` is running.**
  `DxProxy/Direct3D9.cpp:54-63` sets `m_perceptionRunning`; `CreateDevice` returns
  the unwrapped device otherwise (`:336`). The Perception project builds
  `Perception.exe`, so nothing satisfies the check. The comment in `Main.cpp`
  saying the DLL works when dropped next to a game is wrong.
- [ ] **Config path comes from the registry**, `HKCU\SOFTWARE\Vireio\Perception\BasePath`
  (`Shared/ProxyHelper.cpp:272-300`), not the game folder. Without it `LoadConfig`
  fails and the proxy passes through.
- [ ] **Hard import of `openxr_loader.dll`.** The DLL links `openxr_loader.lib`, so a
  game will not start unless the loader DLL is beside it. The OpenXR code that
  needs it does nothing yet (see below); delay-load it or drop the dependency.
- [ ] **`Direct3DCreate9Ex` is a pass-through.** `DxProxy/Direct3D9Ex.cpp:197-231` has
  proxy creation commented out; D3D9Ex games get no Vireio.
- [ ] **`D3DPERF_*` exports call null pointers** if used before `Direct3DCreate9`
  (`DxProxy/Main.cpp:214-247`); `LoadDll` does not null `g_hDll` after a failed
  `GetProcAddress` (`:144-149`).

### Head tracking

- [ ] **`OpenTrackUDPTracker::init()` is never called.** The constructor only chains to
  `MotionTracker()`, whose own constructor calls the base `init()`. No socket or
  thread is created. `FreeTrackTracker` and `SharedMemoryTracker` call `init()`
  from their own constructors. `OpenXRTracker` has the same bug.
- [ ] **No pose reaches the engine even with `init()` fixed.**
  `updateOrientationAndPosition()` is an empty override; the engine reads
  `primaryYaw/Pitch/Roll/X/Y/Z`, which only the base update or the tracker sets
  (compare `SharedMemoryTracker.cpp:87-96`).
- [ ] **Units and signs.** OpenTrack's UDP output is six doubles: x, y, z in
  centimetres, then yaw, pitch, roll in degrees. The tracker treats position as
  metres. `FreeTrackTracker.cpp:113-115` negates yaw and roll; this tracker does
  not. The code and comments call the protocol "FreeTrack 2.0", which it is not.
- [ ] **Socket handling.** Unbalanced `WSACleanup` in the destructor and after a bind
  failure; no `SO_REUSEADDR`; bind failure is silent; a datagram over 48 bytes
  ends the worker thread; one `recvfrom` per 2 ms sleep falls behind a 250 Hz
  sender.
- [ ] **v3 has no view-matrix head-pose injection.** `SetTransform(D3DTS_VIEW)` only
  applies the left/right eye transform (`DxProxy/D3DProxyDevice.cpp:1327-1328`).
  Yaw and pitch go through mouse emulation or VRBoost. The comment at
  `:2447-2451` describing SetTransform-based head tracking is wrong.

### Head-tracking-only build (`Release-Bridge|Win32`, `VIREIO_HT_ONLY`)

- [ ] **Null `stereoView` is dereferenced during `CreateDevice`.**
  `DxProxy/D3DProxyDevice.cpp:3274` (`stereoView->Init`), then `:3276`,
  `:3281-3282`. Further unguarded uses: `D3DProxyDevice.cpp:2681, 2685, 2833-2839,
  2850, 3771-3775, 3797`, and throughout `Hotkeys.cpp`, `InGameMenus.cpp` and
  `Popup.cpp`.
- [ ] Suspected: even when guarded, the swap chain and render targets are still
  wrapped in the stereo proxy classes with nothing compositing to the back buffer.

### Stereo rendering

- [ ] **`D3DXCreateEffectFromFile` is a stub**, so `viewEffect` is null and
  `StereoView::PrePresent` dereferences it on the first `Present`
  (`DxProxy/StereoView.cpp:402`). Needs a real effect path (D3DCompile plus
  hand-written state setup) for every stereo mode.
- [ ] **The constant-table reader never finds a table.** The version-token skip in
  `ID3DXConstantTable::Parse` (`PluginSection/Include/Vireio_D3DX_Compat.h`) is
  inverted, so `D3DXGetShaderConstantTable` returns success with zero constants
  and no shader rule in `ShaderModificationRepository.cpp` matches.
- [ ] **`D3DXLoadSurfaceFromSurface` fails for its real use** (copying a render target
  into a system-memory surface for `LockRect` emulation); the failure is ignored
  and the game locks an uninitialised surface.
- [ ] `D3DXCreateFont` / `D3DXCreateSprite` are stubs: no menu, popups or FPS overlay.
- [ ] `D3DXAssembleShaderFromFile`, `D3DXDisassembleShader`, `D3DXSaveSurfaceToFile`
  are stubs: no shader replacement, no shader dump, no screenshots.
  `DataGatherer.cpp:419-420, 651-652` dereference the null disassembly buffer when
  shader-code output is enabled.

### OpenXR

- [ ] `OpenXRTracker` creates an instance and queries the system, then reports "no
  HMD". No session, space or pose.
- [ ] `OpenXRDirectMode` makes no OpenXR calls. Stereo mode 111 has no effect entry
  and hits the null `viewEffect` crash.
- [ ] `HMDisplayInfo_OpenXR.h` returns hard-coded DK1 values.

### Build

- [ ] `DxProxy.vcxproj` sets `VcpkgEnabled=false`, but `Directory.Build.props`
  re-enables it; the project in fact depends on vcpkg manifest mode for
  `openxr_loader`.
- [ ] Include paths use `$(SolutionDir)`, so only `Perception_v3/VireioPerception.sln`
  builds; `DxProxy/DxProxy.sln` and the bare project do not.
- [ ] `Release-Bridge` `OutDir` and linker `OutputFile` disagree.
- [ ] `MatrixNoShowModification.cpp` and `SocketTracker.cpp` are on disk but not in
  the project.
- [ ] Projects are split between v143 (Perception_v3) and v145 (v4 tree).

## v4 tree (Aquilinus, Perception, plugins)

Retargeted and ported to the D3DX shim; reported to build, not re-verified for
this branch and not runtime tested.

- [ ] `OpenXR-DirectMode` plugin does not submit frames.
- [ ] FreeTrack and OpenXR plugins are not wired into Aquilinus's node categories.
- [ ] The constant-table defect above affects the v4 D3D9 matrix modifier too.

## What was checked and found correct

- The system `d3d9.dll` is loaded by full path from the system directory, so the
  proxy does not load itself.
- The shim's matrix and vector functions (multiply, inverse, transpose,
  perspective and look-at builders, yaw/pitch/roll, vector transforms) match D3DX
  row-vector semantics. One difference: `D3DXMatrixInverse` never returns NULL
  for a singular matrix.
