# AW2726DM HDR Plan

Draft, based on Hyprland 0.57.0 and the EDID read on 2026-11-03.

## Target

Dell Alienware AW2726DM on `DP-2`, 2560x1440 at 240 Hz, 10-bit, QD-OLED.

| Item | Value |
|---|---|
| Connector | `DP-2` |
| GPU | Intel Iris Xe, Raptor Lake-P, `i915`, Mesa 26.2.5 |
| Current compositor | Hyprland 0.57.0, local build from `~/src/hyprland-src` |
| Current format | `XRGB2101010` |
| Current color preset | `srgb` |
| Current auto HDR | `render.cm_auto_hdr = 0` |
| VRR | enabled, `vrr = 3` |
| HDR peak from EDID | 408.759 nits |
| HDR max frame average from EDID | 204.379 nits |
| HDR min from EDID | 0.000 nits |
| EOTF | traditional SDR, traditional HDR, SMPTE ST2084 |
| HLG | not advertised |
| BT.2020 | RGB and YCC advertised |
| Native primaries | red 0.6855/0.3037, green 0.2402/0.7119, blue 0.1435/0.0576, white 0.3134/0.3291 |

The panel is a True Black 400 class OLED. HDR means deep black and highlights up to about 409 nits,
not 1000-nit highlights. Tone mapping must target 409 nits peak and 204 nits frame average.

## Diagnosis

HDR is disabled by configuration, not by hardware:

- `cm = "srgb"` on `DP-2`.
- `render.cm_auto_hdr = 0`.

Two Hyprland code defects remain after enabling HDR:

1. Tone mapping uses the PQ encoding ceiling, 10000 nits, as the content peak. A 1000-nit sample
   on a 409-nit panel maps to roughly 186 nits instead of 409.
2. HDR metadata uses the container primaries, BT.2020, instead of the panel native primaries.

The `cm_auto_hdr = 2` preset (`hdredid`) works around the second defect because it sets the
container primaries to the EDID primaries. It does not fix the first.

## Phase 0: Diagnostics

Install the Vulkan probe and keep the EDID tools already present:

```bash
pkexec pacman -S --needed vulkan-tools
```

`drm_info` is optional and not in the official repositories. Use `modetest -c` for connector
properties and blob IDs when `drm_info` is unavailable.

Collect the baseline:

```bash
edid-decode /sys/class/drm/card1-DP-2/edid
di-edid-decode /sys/class/drm/card1-DP-2/edid
modetest -c
vulkaninfo | grep -E 'VK_EXT_hdr_metadata|VK_EXT_swapchain_colorspace'
hyprctl monitors -j
hyprctl getoption render:cm_auto_hdr
hyprctl getoption render:direct_scanout
```

Record the `HDR_OUTPUT_METADATA` blob ID from `modetest -c`. `modetest` and `drm_info` expose the
blob ID, not the decoded fields. The conformance output probe will decode it with libdrm. Until
that probe exists, use a small libdrm helper or the compositor trace logs.

## Phase 1: Configuration baseline

Edit `~/.config/hypr/monitors.lua`. Replace the `DP-2` block with:

```lua
hl.monitor({
  output = "DP-2",
  mode = "2560x1440@240",
  position = "-320x-1440",
  scale = 1,
  bitdepth = 10,
  vrr = 3,
  cm = "hdr",
  max_luminance = 409,
  max_avg_luminance = 204,
  min_luminance = 0.0005,
  sdr_max_luminance = 204,
  sdr_min_luminance = 0.0005,
  sdrbrightness = 1.0,
  sdrsaturation = 1.0,
})
```

Edit `~/.config/hypr/hyprland.lua`:

```lua
hl.config({
  render = {
    cm_auto_hdr = 2,
    direct_scanout = 2,
  },
})
```

`cm_auto_hdr = 2` switches a fullscreen HDR surface to `hdredid` when the client sets a Wayland
color management image description. Clients that use Vulkan WSI, including mpv 0.41, did not
trigger the switch on Hyprland 0.57.0. For those clients, set `cm = "hdr"` on the output. The
desktop stays in HDR mode, and SDR content is mapped with the configured reference white.

If `hdredid` produces artifacts for Wayland color management clients, test `cm_auto_hdr = 1`.

On the monitor OSD, confirm HDR is enabled and the picture mode is not a demo or SDR mode.

Reload Hyprland or log out and back in. Verify:

```bash
hyprctl monitors -j | python3 -c 'import json,sys; d=json.load(sys.stdin); [print(m["name"], m["colorManagementPreset"], m["currentFormat"], m["sdrMaxLuminance"]) for m in d]'
```

Expected with `cm = "hdr"`: `DP-2 hdr XRGB2101010 204`.
Expected with `cm = "srgb"` and a Wayland color management client: `DP-2 hdredid XRGB2101010 204`
during fullscreen HDR content.

Test with an HDR10 sample:

```bash
mpv --fullscreen --target-colorspace-hint=yes --target-colorspace-hint-mode=source "HDR10-sample.mkv"
```

Observed on 2026-11-03 with `cm = "hdr"`: `colorManagementPreset` became `hdr`, and the
`HDR_OUTPUT_METADATA` blob reported `eotf=2`, native primaries, `max_mastering=409`, and
`max_fall=204`. `cm_auto_hdr` did not trigger for mpv 0.41.

Without the code fixes, expect the monitor to enter HDR and the image to look dim or washed out.
That is the `PIPE-02` defect.

## Phase 2: Code fixes

Source: `~/src/hyprland-src`, base branch `lto-test`, fix branch `hdr-fixes`, tag v0.57.0. The HDR
files match v0.57.0.

### P0.1 Use the content peak for tone mapping

File: `src/render/Renderer.cpp`, `getCMSettings`, around line 2174.

Current code uses `getTFMaxLuminance(-1)`, which returns 10000 for PQ
(`src/helpers/cm/ColorManagement.hpp:307`).

Replace the source luminance selection with:

```cpp
const auto& srcDesc = imageDescription->value();
const float contentMax = srcDesc.maxCLL > 0
    ? sc<float>(srcDesc.maxCLL)
    : srcDesc.masteringLuminances.max > 0
        ? sc<float>(srcDesc.masteringLuminances.max)
        : 1000.0f;

const float maxLuminance = needsHDRmod
    ? contentMax
    : (srcDesc.luminances.max > 0 ? srcDesc.luminances.max : srcDesc.luminances.reference);
```

Keep `getTFMaxLuminance()` for the encoding ranges. It defines the PQ code space, not the content
peak.

### P0.2 Use the native primaries in HDR metadata

File: `src/render/Renderer.cpp`, `createHDRMetadata`, around line 2535.

Current code uses `settings.getPrimaries()`, which returns the container primaries. Replace with:

```cpp
const auto colorimetry = settings.masteringPrimaries == NColorManagement::SPCPrimaries{}
    ? settings.getPrimaries()
    : settings.masteringPrimaries;
```

The monitor native primaries are already in `masteringPrimaries`, read from EDID by
`CMonitor::getMasteringPrimaries()`.

### P0.3 Apply SDR brightness and saturation in linear light

File: `src/render/shaders/glsl/cm_helpers.glsl`, around lines 257 to 260.

Current order encodes to PQ first, then applies saturation and brightness. Move both before the
output transfer function:

```glsl
#if USE_SDR_MOD
    pixColor.rgb *= sdrBrightnessMultiplier;
    pixColor = saturate(pixColor, dstxyz, sdrSaturation);
#endif
    pixColor = fromLinearNit(pixColor, dstTF, dstTFRange);
```

Fix the luma weights in `saturate()`. The current code uses column 1 of the `toXYZ` matrix. Use the
Y row:

```glsl
vec3 brightness = vec3(primaries[0][1], primaries[1][1], primaries[2][1]);
```

At `sdrbrightness = 1.0` and `sdrsaturation = 1.0`, this change is inert. It matters as soon as
the sliders move.

### P1 HLG

The monitor does not advertise HLG. This fix is not required for the AW2726DM. Add it for the
general case by porting the OOTF and system gamma from KWin, and by sending `eotf = 3` when the
EDID advertises HLG.

## Phase 3: Build and install

Back up the current binary:

```bash
cp ~/.local/lib/hyprdeps/bin/Hyprland ~/.local/lib/hyprdeps/bin/Hyprland.pre-hdr
```

Build with the existing local flow:

```bash
cd ~/src/hyprland-src
cmake --build build -j"$(nproc)"
```

Install the new binary where the wrapper expects it:

```bash
install -m 755 build/Hyprland ~/.local/lib/hyprdeps/bin/Hyprland
```

Log out and back in. Confirm:

```bash
hyprctl version | head -3
```

## Phase 4: Verification

### Metadata

Read the `HDR_OUTPUT_METADATA` blob and check:

- `eotf = 2` for PQ.
- `display_primaries` equal the EDID primaries, not BT.2020.
- `white_point` equals 0.3134/0.3291.
- `max_display_mastering_luminance = 409`.
- `min_display_mastering_luminance` near 0.
- `max_cll` and `max_fall` match the content, not each other.

### Pipeline

Use synthetic or known HDR10 patterns:

| Input | Expected on this panel |
|---|---:|
| SDR white | 204 nits |
| PQ 1000 nits | about 409 nits |
| PQ 4000 nits | about 409 nits |
| PQ 100 nits in a 1000-nit master | about 77 nits |
| PQ 100 nits in a 409-nit master | about 100 nits |
| Black | 0.0005 nits or the panel minimum |

A 1000-nit pattern that peaks near 186 nits is still using the 10000-nit encoding ceiling.

### Desktop

- SDR white is not dim.
- Black is not lifted.
- No tint on neutral gray.
- No clipping on gradients.
- The OSD shows HDR only when HDR content is active.

### State

- DPMS off and on.
- Suspend and resume.
- Unplug and replug `DP-2`.
- Switch from HDR content to SDR desktop.

Metadata and the output description must be restored every time.

### Capture

Take a screenshot in HDR mode. It must not be dim or clipped. If the screenshot is SDR, the
compositor must declare the tone map.

## Phase 5: Upstream and conformance

Open three Hyprland changes, each with a reproduction:

1. `PIPE-02`: content peak for tone mapping.
2. `META-01`: native primaries in HDR metadata.
3. `PIPE-06`: linear-light SDR brightness and saturation.

Add the AW2726DM profile to `fixtures/edid/` and run the conformance suite against the patched
Hyprland and KWin. Publish the report with the raw EDID, metadata blob, and test values.

## Rollback

```bash
cp ~/.local/lib/hyprdeps/bin/Hyprland.pre-hdr ~/.local/lib/hyprdeps/bin/Hyprland
```

Restore `~/.config/hypr/monitors.lua` and `~/.config/hypr/hyprland.lua` from backup, or set:

```lua
cm = "srgb"
```

```lua
render = { cm_auto_hdr = 0 }
```

## Risks

- Intel HDR on `i915` is experimental and less tested than AMD. Check `vulkaninfo` for the HDR WSI
  extensions before debugging the compositor.
- `hdredid` is experimental. Test `hdr` if it glitches.
- VRR and HDR together can flicker on some OLED panels. Test `vrr = 2` and `vrr = 0` if needed.
- Direct scanout can bypass the compositor tone mapper. Test with `direct_scanout = 0` to compare
  the composited path.
- The panel peak is about 409 nits. Do not judge HDR by 1000-nit highlight expectations.

## Local status, 2026-11-03

- Config applied: `cm = "hdr"`, `max_luminance = 409`, `max_avg_luminance = 204`,
  `min_luminance = 0.0005`, `sdr_max_luminance = 204`, `sdr_min_luminance = 0.0005`.
- `render.cm_auto_hdr = 2`.
- Hyprland fixes committed on branch `hdr-fixes` at `73861b5`.
- Patched binary built and installed at `~/.local/lib/hyprdeps/bin/Hyprland`.
- Config parsed with `Hyprland --verify-config`. The process exits with a pre-existing destructor
  crash in this local branch. The old binary crashes in the same place, so the HDR patch did not
  introduce it.
- Modified shader compiled with `glslangValidator`.
- Metadata verified in the running session: `eotf=2`, native primaries, `max_mastering=409`,
  `max_fall=204`.
- Runtime verification of the patched tone mapping is pending a logout and login, because Hyprland
  cannot hot-swap the running binary.
