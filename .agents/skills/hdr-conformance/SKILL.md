---
name: hdr-conformance
description: >
  Test, review, and specify HDR behavior across Linux desktop stacks. Use for EDID HDR
  capability detection, Wayland color management protocols, DRM HDR metadata, SDR and HDR
  tone mapping, HLG, ICC, HDR capture, and cross-stack benchmarks against Hyprland, KWin,
  Windows, and Apple. Ships the normative rule set, test matrix, golden values, and report
  schema for the hdr-conformance project.
---

# HDR Conformance

Operational guide for testing, reviewing, and specifying HDR behavior on Linux desktops. The
target is a stack that detects the display, negotiates the right color description, maps content
correctly, emits matching metadata, and preserves HDR through capture.

## Scope

- Display capability: EDID HDR Static Metadata, DisplayID, BT.2020, PQ, HLG, and luminance.
- Protocol: `wp_color_manager_v1`, `frog_color_management_v1`, and `xx_color_management_v4`.
- Pipeline: SDR to HDR, HDR to HDR, HDR to SDR, gamut mapping, and ICC.
- Output: DRM `HDR_OUTPUT_METADATA`, bpc, pixel format, and content type.
- Capture: screenshot, screen capture, and writeback.
- Benchmarks: Hyprland, KWin, Windows, and Apple.

## Core model

Four descriptions must agree:

1. Display capability, read from EDID.
2. Content description, sent by the client.
3. Output description, chosen by the compositor.
4. Metadata sent to the display.

A conformance failure is a disagreement between two of them. Inspect all four before changing
code.

## Normative rules

RFC 2119 keywords apply. Every rule has an ID, a requirement, and a test in the suite.

| ID | Requirement | Tier |
|---|---|---|
| CAP-01 | A compositor MUST parse HDR Static Metadata from the EDID, including PQ, HLG, BT.2020, min and max luminance, maxCLL, and maxFALL. | core |
| CAP-02 | A compositor SHOULD use 10-bit or FP16 scanout for HDR, and MUST report the actual bit depth to clients and in the conformance report. | core |
| CAP-03 | A compositor MUST reject or clamp invalid HDR metadata, including min luminance greater than or equal to max luminance and white points outside the gamut. | core |
| CAP-04 | A compositor MUST distinguish SDR-only, PQ, and HLG-only displays. | core |
| PROTO-01 | A compositor MUST advertise only the color management features it implements. | core |
| PROTO-02 | A compositor MUST reject invalid image descriptions with the protocol error defined by the interface. | core |
| PROTO-03 | The preferred image description exposed to clients MUST match the output's current image description. | core |
| PROTO-04 | A compositor MUST keep SDR and HDR image descriptions distinct, with no HDR headroom on SDR descriptions. | core |
| PIPE-01 | SDR reference white in HDR mode SHOULD default to 203 nits, or to a user-configured reference, and MUST be applied in linear light. | pipeline |
| PIPE-02 | Tone mapping MUST use the content peak from maxCLL or mastering luminance, not the PQ encoding ceiling. | pipeline |
| PIPE-03 | Gamut mapping MUST use the source container primaries for the encoding and the display native primaries for the output. Tone mapping MUST use the mastering primaries to bound the content gamut. | pipeline |
| PIPE-04 | HLG conversion MUST apply the OOTF and system gamma, and MUST use the display peak. | pipeline |
| PIPE-05 | ICC profiles MUST compose with HDR output, preserving headroom above the reference white. | pipeline |
| PIPE-06 | Saturation and brightness adjustments MUST be applied before the output transfer function. | pipeline |
| META-01 | HDR metadata MUST use the display native primaries, not the container primaries. | core |
| META-02 | maxCLL and maxFALL MUST be independent and MUST come from the content when present. | core |
| META-03 | A compositor MUST NOT send HDR metadata when the output is in SDR mode. | core |
| STATE-01 | HDR metadata and the output description MUST be restored after DPMS, suspend, resume, and hotplug. | core |
| CAPTURE-01 | HDR capture MUST preserve the output image description or explicitly tone map to SDR with a declared transform. | capture |
| CAPTURE-02 | SDR screenshots in HDR mode MUST NOT be clipped or dimmed by the HDR pipeline. | capture |

## Conformance profiles

| Profile | Rules | Claim |
|---|---|---|
| `hdr-core` | CAP, PROTO, META, STATE | The stack detects HDR, negotiates it, and emits valid metadata. |
| `hdr-pipeline` | PIPE | The stack maps SDR and HDR content to the display correctly. |
| `hdr-capture` | CAPTURE | The stack preserves HDR through screenshot and screen capture. |
| `hdr-lab` | all plus measured output | The physical display matches the declared behavior. |

A stack can pass `hdr-core` and fail `hdr-pipeline`. Report the profile with the result.

## Procedure

### 1. Probe the stack

Collect the four descriptions before running any test.

```bash
# Display capability
edid-decode /sys/class/drm/card*-*/edid
di-edid-decode /sys/class/drm/card*-*/edid

# DRM output state
drm_info
modetest -c

# Compositor state
hyprctl monitors -j
kscreen-doctor -o
swaymsg -t get_outputs

# Vulkan HDR WSI
vulkaninfo | grep -E 'VK_EXT_hdr_metadata|VK_EXT_swapchain_colorspace'
```

`wayland-info` does not print `wp_color_manager_v1`. Use a dedicated client for protocol probing.

### 2. Select fixtures

Use synthetic EDID and content fixtures. CI must run without HDR hardware.

- EDID: SDR only, HDR400, HDR1000, OLED, HLG only, BT2020 SDR, invalid metadata, DisplayID HDR.
- Content: sRGB, PQ 1000, PQ 4000, HLG, scRGB, ICC sRGB, ICC PQ.
- Output: 8-bit SDR, 10-bit SDR, 10-bit HDR, 12-bit HDR.
- State: DPMS off and on, suspend and resume, hotplug, mode switch.

### 3. Test the protocol

Bind the color manager and inspect features, image descriptions, preferred description, feedback,
and errors. Cover the upstream protocol and the two protocols used by Hyprland and gamescope.

- `wp_color_manager_v1`: KWin, Mutter, Weston, wlroots.
- `frog_color_management_v1`: Hyprland, gamescope.
- `xx_color_management_v4`: Hyprland, gamescope.

### 4. Test the pipeline

Compare the compositor's output with an independent reference. Do not use the compositor's own
math as the oracle.

- Reference implementations: libplacebo `pl_tone_map`, KWin `ColorPipeline::evaluate`, Colour
  Science for offline vectors.
- Check the golden values below.
- Test both directions: SDR to HDR and HDR to SDR.

### 5. Test the output

Read the DRM metadata blob and compare every field with the expected values.

- `eotf`: 2 for PQ.
- `display_primaries`: native primaries from EDID.
- `white_point`: native white point.
- `max_display_mastering_luminance`: mastering peak.
- `min_display_mastering_luminance`: in 0.0001 nits.
- `max_cll` and `max_fall`: independent values from the content.

### 6. Test capture

Capture must preserve the image description or declare a tone map to SDR. A screenshot that
silently clips HDR is a failure.

### 7. Report

Emit JSON plus a human summary. Every failure must include evidence: EDID, metadata blob, captured
values, and logs.

## Golden values

Panel peak 600 nits, reference white 203 nits, unless the fixture says otherwise. Default tolerance
is plus or minus 5 percent for synthetic pipeline tests and plus or minus 10 percent for captured
output. A profile may define a tighter tolerance.

| Input | Expected output | Notes |
|---|---:|---|
| SDR white, 80 nits | 203 nits | BT.2408 reference white in HDR mode |
| PQ 1000 nits | 600 nits | Tone map to panel peak |
| PQ 4000 nits | 600 nits | Tone map to panel peak |
| HLG 75% | 203 nits | BT.2408 HLG reference |
| PQ code for 100 nits | 0.5081 | Encoding check |
| PQ code for 203 nits | 0.5807 | Encoding check |

If a stack maps 1000-nit content to 203 nits on a 600-nit panel, it is using the PQ encoding
ceiling as the content peak. That is a `PIPE-02` failure.

## Pitfalls

| Trap | Symptom | Rule |
|---|---|---|
| Container primaries in HDR metadata | Desaturation, clipped white point | META-01 |
| PQ encoding ceiling as content peak | 1000 nits rendered near 203 nits | PIPE-02 |
| Brightness or saturation applied after PQ encoding | SDR slider behaves nonlinearly, highlights blow out | PIPE-06 |
| HLG without OOTF or system gamma | HLG looks flat or too bright | PIPE-04 |
| ICC LUT clamped to [0,1] | HDR headroom disappears when ICC is active | PIPE-05 |
| EDID HDR block in DisplayID not parsed | HDR option missing on a capable panel | CAP-01 |
| SDR screenshot taken through the HDR pipeline | Screenshot is dim or clipped | CAPTURE-02 |
| HDR metadata duplicated from maxFALL into maxCLL | Wrong content peak and average | META-02 |
| Autologin or session selector hides the desktop | User cannot choose the HDR-capable session | not HDR, operational |
| Auto HDR only reacts to Wayland color management clients | mpv or a game stays SDR while the output is SDR | operational |

## Report schema

```json
{
  "stack": "hyprland",
  "version": "0.57.0",
  "gpu": "amd",
  "session": "wayland",
  "edid": {
    "hdr": true,
    "pq": true,
    "hlg": false,
    "bt2020": true,
    "max_luminance": 600,
    "min_luminance": 0.05
  },
  "profile": "hdr-pipeline",
  "checks": [
    {
      "rule": "PIPE-02",
      "id": "pq_1000_peak",
      "expected": 600,
      "actual": 203,
      "unit": "nits",
      "status": "fail",
      "evidence": "HDR_OUTPUT_METADATA max_cll=1000, tone map input=10000"
    }
  ]
}
```

## Benchmark targets

| Stack | HDR path | What to compare |
|---|---|---|
| Hyprland | `xx-color-management-v4`, `frog-color-management-v1`, `cm` presets, `cm_auto_hdr` | Fullscreen auto HDR, SDR in HDR, metadata, capture |
| KWin | `wp-color-management-v1`, DRM `HDR_OUTPUT_METADATA`, per-output HDR | Protocol coverage, ICC in HDR, tone mapping, gamescope interop |
| Windows | DWM, DXGI `DXGI_OUTPUT_DESC1`, `IDXGISwapChain4::SetHDRMetaData` | SDR white level, HDR metadata, auto HDR, tone mapping |
| Apple | macOS CoreVideo, ColorSync, Metal EDR, `NSScreen` EDR values | EDR headroom, reference modes, color sync, capture |

## Review checklist

- Does the change read the right description for each stage?
- Does it use content peak, not encoding ceiling, for tone mapping?
- Does it use native primaries for metadata?
- Does it scale luminance in linear light?
- Does it apply HLG OOTF?
- Does it keep ICC and HDR composable?
- Does it restore state after DPMS, suspend, and hotplug?
- Is there a test that fails before the change and passes after?

## References

- Wayland color management protocol: https://wayland.app/protocols/color-management-v1
- IGT `kms_hdr`: https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/blob/master/tests/kms_hdr.c
- Weston color tests: https://gitlab.freedesktop.org/wayland/weston/-/tree/main/tests
- KWin color tests: https://invent.kde.org/plasma/kwin/-/tree/v6.8.0/autotests
- Mutter color tests: https://gitlab.gnome.org/GNOME/mutter/-/tree/main/src/tests
- libplacebo: https://github.com/haasn/libplacebo/tree/master/src/tests
- BT.2408: https://www.itu.int/rec/R-REC-BT.2408
- CTA-861: https://www.cta.tech/Resources/Standards
