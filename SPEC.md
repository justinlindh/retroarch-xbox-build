# Fast shader loads for RetroArch on Xbox Series X (Dev Mode)

## Problem

ES-DE launches RetroArch for every game, and the Xbox terminates and cold-starts
apps of type Game, so every launch is a full RetroArch start. With the Mega
Bezel CRT preset (MBZ__3__STD__GDV-NTSC, 35 passes) the shader chain is compiled
from source on every one of those starts. On the Xbox this takes long enough that
the launch watchdog killed RetroArch until `video_shader_delay` moved the
compile after launch. Loads are still slow.

## Facts from the code

- The Xbox build has no Vulkan driver (no `HAVE_VULKAN` in the UWP project, now
  or in its history). The bundled Mesa libraries serve OpenGL. Setting
  `video_driver = "vulkan"` on the Xbox falls back silently. The target driver is
  D3D11.
- Installed build (XboxEmulationHub, 2026-08-19): slang -> SPIR-V with glslang,
  SPIR-V -> HLSL with SPIRV-Cross, HLSL -> DXBC with `D3DCompile`, then the
  driver turns DXBC into GPU code. It has a SPIR-V cache that only works when
  `cache_directory` is set; on the Xbox it is empty.
- Current upstream: the UWP project builds with `HAVE_SLANG;HAVE_RSLC` and
  neither glslang nor SPIRV-Cross. rslc (libretro-common, C89) parses slang and
  prints HLSL for D3D11; `D3DCompile` still compiles it every time.
- `d3d_compile()` in `gfx/common/d3dcompiler_common.c` is the single call site
  for runtime HLSL compiles in all D3D drivers (9/10/11/12). Nothing caches its
  output.
- rslc also has `RSLC_DST_DXBC` (under `RSLC_EXTENDED`): DXBC straight from the
  parsed stage, no `D3DCompile`. No driver uses it yet.
- The XboxEmulationHub fork has no commits of its own; upstream carries the
  whole Xbox package: `pkg/msvc/RetroArch-msvc2026-UWP.sln`, Mesa/ANGLE
  libraries, manifest, temporary signing key. Upstream CI builds it on
  `windows-2025` and passes.
- Package identity in upstream and in the installed build is the same:
  `Name=1e4cf179-f3c2-404f-b9f3-cb2070a5aad8`, `Publisher=CN=libretro`,
  `Version=1.22.2.0`.

## Plan

### Phase 0: enable the existing cache (config only)

Set `cache_directory = "E:\RetroArch\cache"` in the live Xbox config
(`E:\RetroArch\config\retroarch-crt.cfg`, edited only while RetroArch is closed)
and set `video_driver` back to `d3d11`. Time the first and second launch of the
same game. This measures how much the glslang stage costs on the installed
build.

### Phase 1: build current upstream for Xbox

Repo `github.com/justinlindh/RetroArch`, branch `xbox-shader-cache` on
`upstream/master`. New manually-dispatched workflow
`.github/workflows/Xbox-Package.yml` on `windows-2025`:

1. Check out the branch.
2. Download `RetroArch-SeriesConsoles-AllCores.appx` from the XboxEmulationHub
   release named by an input, extract `cores/*.dll` into
   `pkg/msvc/msvc-uwp/cores/x64/cores/` (the project packages whatever is there).
3. Set `Version` in `Package.appxmanifest` to `1.22.2.<run_number>` so each build
   upgrades the installed app in place, keeping its LocalState and settings.
4. Build `RetroArch-msvc2026-UWP.sln`, `Release|x64`, appx packaging on, signed
   with the repo's temporary key.
5. Upload the package and its dependency packages as an artifact.

Install via Xbox Device Portal like the existing packages.

Acceptance: starts on the Xbox, cores load, Mega Bezel renders on D3D11, and the
timing log (Phase 2) or a stopwatch shows the rslc build against the installed
one.

### Phase 2: D3D compiled-shader cache (our patch)

In `d3d_compile()`:

- Key: 64-bit hash over the source bytes, entry point, target profile, compile
  flags and a compiler identity, in the same letters-and-digits form the SPIR-V
  cache uses.
- New `VIDEO_SHADER_CACHE_D3D_BLOB` kind, files at
  `<cache_directory>/d3dblob/<key>.dxbc`, read through `video_shader_cache_map`
  and written through `video_shader_cache_write`.
- Hit: wrap the bytes with `D3DCreateBlob` and return without compiling.
- Miss: compile as today, then write the blob.
- Empty `cache_directory`: unchanged behaviour.
- `D3DCreateBlob` resolved the same way as `D3DCompile` (dynamic on desktop,
  linked on UWP).

Instrumentation in the same change: one log line per preset load with total
milliseconds, passes compiled, cache hits and time spent in `D3DCompile`.

Upstreamable as a libretro PR once proven.

### Phase 3 (stretch): rslc DXBC for first loads

Feed `RSLC_DST_DXBC` output to `CreateVertexShader`/`CreatePixelShader` on D3D11,
so even a first load skips `D3DCompile`. Experimental in rslc; only worth it if
first loads still matter after Phase 2, and only with a correctness check of
Mega Bezel against the `D3DCompile` path.

## Testing

| What | Where |
|---|---|
| D3D blob cache, timing log | mingw-w64 Windows build of the branch under Wine with DXVK (D3D11 on Vulkan), Mega Bezel preset |
| Package build | GitHub Actions artifact |
| Real timings | Artifact installed on the Xbox via Device Portal |

Each test: empty cache directory, launch, then launch again; compare the timing
lines and confirm `d3dblob/` was written then read.

## Out of scope here

ES-DE's cold start (Cathode theme load, game list parse). The Xbox terminates
ES-DE whenever RetroArch runs, so its startup time is separate work of equal
weight.
