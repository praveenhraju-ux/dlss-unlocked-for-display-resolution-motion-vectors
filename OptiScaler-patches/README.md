# OptiScaler patch: DLSS-NR Denoise First with display-resolution motion vectors

`0001-feat-dlssnr-denoise-first-with-display-resolution-mo.patch` applies to
[ShyVortex/OptiScaler-DLSSNR-PreSR-Multipass](https://github.com/ShyVortex/OptiScaler-DLSSNR-PreSR-Multipass)
at release `v0.9.50` (`d5fa05cb92e9667b648d120e9d9b65e09bd2370a`). That is the OptiScaler backend of DLSS Unlocked
[`NR-v0.9.50-hotfix`](https://github.com/ShyVortex/dlss-unlocked/releases/tag/NR-v0.9.50-hotfix), which this fork is based on.

## What it changes

Before this patch, Denoise First (`[DlssNr] DenoiseFirst=true`, native DX12) stayed off for games whose DLSS
feature is created without `MVLowRes`, i.e. games that give DLSS **display-resolution** motion vectors
(Stellar Blade is one). The log showed:

```
DLSS-NR denoise first: inactive: this game supplies display-resolution motion vectors; the 1:1 pass needs render-resolution ones
```

With the patch, the vectors are resampled each frame into a private render-resolution texture, which is used by
the private 1:1 DLSS/RR pass, the private DLSS SR step (`DenoiseFirstStep=1`) and the NR model:

- Each render pixel takes the display texel under its centre. Vectors are never blended across object edges.
- Values are converted to render pixels (`MV scale × render ÷ output` on each axis), so direction and on-screen
  distance stay the same. Jitter offsets pass through unchanged.
- The game's motion texture goes back to its declared resource state. The private copy is freed through the same
  GPU-lifetime retirement as the rest of the Denoise First generation.
- Every DLSS mode works: DLAA, Quality, Balanced, Performance and Ultra Performance. The scale comes from each
  frame's actual render and output sizes, so dynamic resolution works too. The tests cover all five modes at
  1080p, 1440p and 4K.
- The game's own DLSS upscale keeps its original motion vectors and parameters. Games that already supply
  render-resolution vectors take exactly the path they took before.

Files: `DlssNr_Dx12_DenoiseFirst.cpp`, new `DlssNr_DisplayMotion.h`, new shader `dlssnr_motion_downscale.hlsl`
(DXIL compiled with dxc `cs_6_0`), plus `DlssNr_Dx12.{h,cpp}` and `DlssNr_Dx12_State.h`. Tests:
`tests/nr_denoise_first_display_motion_unit.cpp` (headless) and `tests/nr_denoise_first_display_motion_warp.cpp`
(runs the shipped shader on WARP).

## Setup installer and release

`build-installer.yml` builds the patched DLL through `build-optiscaler-patched.yml` and puts it in place of the
upstream `OptiScaler.dll`, in both `setup.exe` and the standalone zip. It refuses to mix versions: the upstream
OptiScaler release has to be the one the patch was built on (`v0.9.50`). To publish a release of this fork, run
**Build DLSS-Unlocked Installer** from the Actions tab with:

| input | value |
|---|---|
| build_action | `release` |
| optiscaler_version | `v0.9.50` |
| tag_name | `NR-v0.9.50-display-mv` |
| release_type | `stable` |
| force_build | `true` |

## Building

The **Build patched OptiScaler** workflow (`.github/workflows/build-optiscaler-patched.yml`) checks out the
pinned upstream commit with its submodules and applies this patch. It runs the regression tests, builds Release
x64 with MSBuild, and uploads the `OptiScaler-denoise-first-display-mv` artifact. You can start it from the
Actions tab ("Run workflow"); pushing a change under `OptiScaler-patches/` also starts it.

To build locally on Windows with Visual Studio 2022:

```powershell
git clone --recurse-submodules https://github.com/ShyVortex/OptiScaler-DLSSNR-PreSR-Multipass optiscaler
cd optiscaler
git checkout v0.9.50
git submodule update --init
git apply --whitespace=nowarn ..\OptiScaler-patches\0001-*.patch
.\tests\run_nr_denoise_first_display_motion.ps1      # from a "x64 Native Tools" prompt
msbuild OptiScaler.sln /m /p:Configuration=Release /p:Platform=x64 /p:PostBuildEventUseInBuild=false
# output: x64\Release\a\OptiScaler.dll
```

## Installing alongside DLSS Unlocked

Only the OptiScaler proxy DLL changes. Every other DLSS Unlocked file stays as it is: `OptiScaler.ini`,
`nvngx_dlssnr.dll`, `nvngx.dll_dlssnr.dll`, the `OptiScaler\` folder, Streamline, dlssg_sm86 and so on.

1. Install DLSS Unlocked into Stellar Blade as usual. That is the folder holding the shipping executable, e.g.
   `...\StellarBlade\SB\Binaries\Win64`.
2. Back up the proxy DLL that DLSS Unlocked installed there. Its name depends on the installer option you chose:
   `dxgi.dll` (default), `version.dll`, `winmm.dll`, or `plugins\dlss-unlocked.asi`.
3. Copy `OptiScaler.dll` from the artifact over that file, **using the same name** (e.g. rename it to `dxgi.dll`).
4. In `OptiScaler.ini`, under `[DlssNr]`, keep NR enabled and set:
   ```ini
   DenoiseFirst=true
   ; 0, 1 or 2 -- all three steps are supported; 2 is the default
   DenoiseFirstStep=auto
   ```
   Denoise First is not combined with `DeferredDLSS` or `FinishedPicture`. Turn those off, along with the debug
   view, Compare and Show skin mask.
5. Start the game and pick any DLSS mode (DLAA, Quality, Balanced, Performance or Ultra Performance).

To revert, put the backed-up proxy DLL back.

## Verifying in Stellar Blade

`OptiScaler.log` (next to the proxy DLL) should contain:

```
DLSS-NR denoise first: display-resolution motion 3840x2160 resampled to 2227x1253 for the 1:1 pass and NR (scale ... -> render pixels); the game's upscaler keeps its own vectors
DLSS-NR denoise first: creating 1:1 DLSS SR 2227x1253, ... low-res MV true (resampled from display resolution), ...
DLSS-NR denoise first: running: DLSS 1:1 -> NR -> ...
```

The sizes shown are for a 4K output at Balanced; other resolutions show their own. The OptiScaler overlay's
DLSS-NR section should show the same "running" status with live 1:1 timings. It should not show "inactive".

Things to check visually, in each of `DenoiseFirstStep` 0, 1 and 2:

- Pan the camera quickly, and run past foliage and thin geometry. There should be no ghosting trails, smearing,
  or "swimming" detail that lags behind the camera. Errors in motion scale or direction show up most clearly here.
- Watch an edge such as Eve's silhouette against the sky while she moves. Doubled or offset edges point to
  misaligned vectors.
- Hold the camera still. The image should settle and stay stable, with no per-frame shimmer.
- Set `DenoiseFirst=false`. The game's ordinary DLSS image must be unchanged by the patch.

If anything looks wrong, enable the NR pipeline capture. With display-resolution vectors it also saves the
resampled `motion_render` texture and records `display_motion 1 motion_scale <x> <y>` in its metadata, so the
vectors the private passes saw can be inspected directly.
