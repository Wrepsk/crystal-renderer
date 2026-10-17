# CrystalRenderer

CrystalRenderer is a standalone screen-rendering library for FFXIV/Dalamud plugins, consumed by [CrystalCast](https://github.com/Wrepsk/crystal-cast) and other hosts; the renderer has no dependency on CrystalCast, browser providers, playback, audio, configuration, or IPC.

Originally extracted from CrystalCast, the library contains the tested screen-rendering subset from the CrystalCast-specific Pictomancy fork at `ed205c725fe1d17124480823eac7affaccd669ee`. It reuses that implementation's shaders, D3D11 resource handling, scene-depth occlusion, UI masks, and native-overlay bridge. General shapes, scene decals, VFX, and the Pictomancy static service/draw-list API are excluded. The original submodule has been removed; source provenance and license text remain in this repository.

## Use in another plugin

Add the repository as a submodule, then reference the library project:

```powershell
git submodule add https://github.com/Wrepsk/crystal-renderer.git CrystalRenderer
dotnet add YourPlugin/YourPlugin.csproj reference CrystalRenderer/CrystalRenderer/CrystalRenderer.csproj
```

Use `git clone --recurse-submodules` when cloning a consuming repository, or run `git submodule update --init --recursive` in an existing checkout. Each host pins a renderer commit and can update it independently. Commit and push renderer changes here first, then commit the updated submodule pointer in each host.

## Build and test

Install the .NET SDK selected by `global.json` and Dalamud development assemblies at `%AppData%/XIVLauncher/addon/Hooks/dev`. From this repository's root:

```powershell
dotnet restore CrystalRenderer.sln --locked-mode -p:Platform=x64
dotnet build CrystalRenderer.sln -c Debug --no-restore -p:Platform=x64
dotnet test CrystalRenderer.sln -c Debug --no-build --no-restore -p:Platform=x64
```

CI runs the same checks in Debug and Release on Windows. The renderer has no runtime dependency on CrystalCast. The copied rendering-validation notes describe the CrystalCast host used for live investigations; its placement, browser, and fallback policies belong to that host. Existing transitive ImageSharp audit findings are recorded in [docs/DEPENDENCY_AUDIT.md](docs/DEPENDENCY_AUDIT.md).

## Host API

- `IScreenRenderBackend` owns render resources and output integration.
- `DalamudScreenRenderer` supplies FFXIV/Dalamud integration through instance-owned services.
- `IScreenRenderFrame` accepts immutable `ScreenSurface` submissions and completes or cancels a frame exactly once.
- `ScreenSurface` contains a D3D11 shader-resource-view pointer, world-space center, full-width/full-height axes, UV coordinates, and `ScreenAppearance`.
- `RenderOutput` selects scene compositing, native overlay, or ImGui output. Output selection and compatibility fallback policy belong to the host.

Create one backend per plugin, on the host's render/UI thread after character login:

```csharp
var renderer = new DalamudScreenRenderer(pluginInterface,
    new ScreenRendererOptions { MaxSurfaces = 256 });

// Inside the host's draw callback:
using var frame = renderer.BeginFrame(RenderOutput.NativeOverlay);
frame?.Submit(new ScreenSurface(shaderResourceViewPointer, center,
    rightAxis * width, downAxis * height, new ScreenAppearance()));
```

Keep the original texture valid until `Submit` returns. Submission retains its shader-resource view for deferred rendering; the host continues to own its original wrapper. The renderer cache releases unused texture views after 120 rendered frames, or when the backend is disposed. Call `Cancel()` on a frame whose submissions must not render. Dispose the backend before unloading the plugin.

The host must poll `ThrowIfRenderFailed()` before preparing its next frame so callback failures reach managed graphics recovery. Check pending/stalled scene-frame diagnostics before updating media textures; cancel pending compositing before switching output layers. CrystalCast's adapter implements this sequence and its Wine/native fallback policy.

CrystalCast offers an explicit experimental Wine scene-composite opt-in while keeping native overlay as the Wine default. The renderer reuses the game's D3D11 device and hook; no Linux-specific graphics backend is assumed to be validated. Status includes hook installation errors, backbuffer/scene-stage bind counts, relaxed callbacks, and scheduled/flushed frames. The host cancels queued submissions when falling back and keeps experimental callback failures disabled across resource recovery until an explicit retry. See the [CrystalCast Linux trial](docs/RENDERING_VALIDATION.md#experimental-wine-scene-composite-trial).

`D3D11/` owns the screen batch, render targets, depth views, compositing, and hook scheduling. `Integration/` contains FFXIV scene capture and the KamiToolKit native overlay. Per-frame camera, viewport, and depth pointers are captured together at the pass that consumes them. Scene-frame timeouts use a monotonic clock.

Screen geometry uses the render camera's `ViewMatrix * ProjectionMatrix`. Depth sampling uses that same view snapshot with `ProjectionMatrix2`, which contains the scene's temporal jitter on the validated game setup. This preserves the game's lens shifts and depth convention without estimating or smoothing projection offsets. Mixing the gameplay Control matrix with a newer camera view can leave transient object-shaped cutouts while moving. Depth conversion uses the depth projection's coefficients rather than the translated combined matrix. This does not smooth real camera motion or change the game's anti-aliasing settings.

The D3D11 context owns explicit point/clamp and linear/clamp samplers. Screen draws bind point sampling for scene depth at `s0` and linear sampling for the video at `s1`; the composite and UI-mask passes also bind their samplers explicitly. HLSL state declarations alone did not configure those states, leaving depth sampling on Direct3D's default linear filter and allowing false foreground depths between adjacent texels.

DLSS silhouette distortion remains a known limitation of the current compositing path. FSR with TSCMAA was clean in the reported live comparison. Spatial coverage filtering and a one-pixel foreground guard did not resolve the DLSS artifact; both experiments were removed. See [rendering validation](docs/RENDERING_VALIDATION.md) before pursuing changes to that path.

## Validation and next steps

`CrystalRenderer.Tests` has no CrystalCast project reference. Tests cover the host contract, frame lifecycle/capacity, geometry validation, deferred scheduling, COM identity, depth formats, shader compilation, and software D3D11 pixel output. Test injected scene capture also verifies compositing and missing-depth recovery without FFXIV.

Live validation of FFXIV render timing and Windows/Wine graphics drivers remains necessary; see [rendering validation](docs/RENDERING_VALIDATION.md). This migration preserves the current transparency model: opaque intersections use depth testing; intersecting translucent surfaces still use distance sorting. SharpDX and KamiToolKit remain at their existing versions. Possible later work includes explicit texture leases, a fuller game-device abstraction, more precise translucent ordering, and replacement of the native overlay bridge.

## Attribution

Ported files retain Pictomancy attribution. See [CrystalRenderer/LICENSE.Pictomancy](CrystalRenderer/LICENSE.Pictomancy) for the AGPL-3.0 license text and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for provenance. This is a focused derivative implementation, not an independent reimplementation from scratch.
