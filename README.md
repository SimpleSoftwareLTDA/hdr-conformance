# HDR Conformance

Cross-platform HDR conformance and benchmarking for desktop compositors and display stacks.

## Why this exists

HDR support on Linux is fragmented across compositors, drivers, and protocols. There is no shared, portable test suite that answers a simple question: does this stack correctly detect an HDR display, negotiate HDR with applications, map SDR and HDR content, and emit the right metadata?

This project builds that suite and uses four stacks as benchmarks: Hyprland, KWin, Windows, and Apple.

## Purpose

Provide a reproducible, platform-neutral way to:

- verify HDR capability detection from EDID;
- verify protocol negotiation and error handling;
- verify color pipeline behavior for SDR, PQ, HLG, scRGB, and ICC;
- verify output metadata and pixel format;
- verify HDR capture and mode transitions;
- compare correctness and quality across stacks with a common report.

## What conformance means here

A stack passes an HDR profile when:

1. It detects the display HDR capabilities from EDID, including PQ, HLG, BT.2020, min and max luminance, maxCLL, and maxFALL.
2. It exposes the correct protocol features and rejects invalid image descriptions.
3. It maps SDR content to the configured reference white and HDR content to the display peak without clipping or crushing.
4. It emits output metadata that matches the panel native primaries and the content metadata.
5. It preserves HDR through capture and survives DPMS, suspend, resume, and hotplug.
6. It reports failures with actionable diagnostics.

Conformance is profile-based. A stack can be conformant for HDR10/PQ and not for HLG, or conformant for desktop HDR and not for capture.

## Benchmark targets

| Stack | HDR path | Adapter | Focus |
|---|---|---|---|
| Hyprland | Wayland `xx-color-management-v4` and `frog-color-management-v1`, `cm` presets, `cm_auto_hdr` | Planned | Fullscreen auto HDR, SDR in HDR, metadata, capture |
| KWin | Wayland `wp-color-management-v1`, DRM `HDR_OUTPUT_METADATA`, per-output HDR | Planned | Protocol coverage, ICC in HDR, tone mapping, gamescope interop |
| Windows | DWM, DXGI `DXGI_OUTPUT_DESC1`, `IDXGISwapChain4::SetHDRMetaData`, `Windows.Graphics.Display` | Planned | SDR white level, HDR metadata, auto HDR, tone mapping |
| Apple | macOS CoreVideo, ColorSync, Metal EDR, `NSScreen` EDR values | Planned | EDR headroom, reference modes, color sync, capture |

The first milestone targets Hyprland and KWin. Windows and Apple adapters follow the same scenario definitions and report schema.

## Test layers

1. `probe`: collect EDID, DRM properties, Wayland globals, and compositor output state.
2. `edid`: synthetic EDID fixtures for SDR, HDR400, HDR1000, OLED, HLG-only, BT2020 SDR, invalid metadata, and DisplayID HDR blocks.
3. `protocol`: client tests for `wp_color_manager_v1`, `frog_color_management_v1`, and `xx_color_management_v4`. Features, image descriptions, preferred description, feedback, and errors.
4. `pipeline`: golden vectors for SDR to HDR, HDR to HDR, HDR to SDR, PQ, HLG, scRGB, gamut mapping, and ICC.
5. `output`: DRM metadata blob, bpc, pixel format, content type, and mode transitions.
6. `capture`: HDR screenshots and screen capture, code value checks, and color space tags.
7. `report`: JSON results, human summary, and comparison matrix.

## Test matrix

| Dimension | Values |
|---|---|
| EDID | SDR only, HDR400, HDR1000, OLED, HLG only, BT2020 SDR, invalid metadata, DisplayID HDR |
| Content | sRGB, PQ 1000, PQ 4000, HLG, scRGB, ICC sRGB, ICC PQ |
| Output | 8-bit SDR, 10-bit SDR, 10-bit HDR, 12-bit HDR |
| State | DPMS off and on, suspend and resume, hotplug, mode switch |
| Stack | Hyprland, KWin, Windows, Apple |

## Expected assertions

Examples of checks:

- SDR white in HDR mode maps to 203 nits, or to the configured reference, within tolerance.
- PQ 1000-nit content on a 600-nit panel peaks at 600 nits, not at the PQ encoding ceiling.
- HDR metadata `display_primaries` matches the EDID native primaries, not the container primaries.
- `max_cll` and `max_fall` are independent and match content metadata.
- HLG uses the OOTF and system gamma, and 75% HLG maps to the expected reference.
- ICC profiles compose with HDR instead of disabling HDR.
- No HDR metadata is sent in SDR mode.
- HDR metadata is restored after DPMS, suspend, and hotplug.

## Report format

Planned JSON shape:

```json
{
  "stack": "hyprland",
  "version": "0.57.0",
  "gpu": "amd",
  "edid": {"hdr": true, "pq": true, "hlg": false, "max_luminance": 600},
  "profile": "hdr10-pq",
  "checks": [
    {"id": "sdr_white", "expected": 203, "actual": 203, "unit": "nits", "status": "pass"},
    {"id": "pq_peak", "expected": 600, "actual": 203, "unit": "nits", "status": "fail"}
  ]
}
```

## Design principles

- One scenario definition, many adapters. Platform-specific code stays behind an adapter.
- Synthetic fixtures first. CI runs without HDR hardware.
- Measured verification is a separate tier. Software conformance and physical panel behavior are different claims.
- Failures include evidence: EDID, metadata blob, captured values, and logs.
- No vendor-specific assumptions in the core.

## Status

Pre-alpha. The repository currently contains the project definition and test design. No adapters or test runner yet.

## Roadmap

- M0: scenario spec, EDID fixtures, report schema.
- M1: Linux probe, protocol client for the three Wayland color management protocols, KWin and Hyprland adapters.
- M2: pipeline golden vectors and CI with synthetic EDID.
- M3: Windows and Apple adapters.
- M4: HDR capture checks and mode transition tests.
- M5: public comparison dashboard.

## Repository layout (planned)

```text
cmd/hdr-conformance/     CLI entry point
internal/probe/          EDID, DRM, Wayland, compositor probes
internal/protocol/       wp, frog, xx protocol clients
internal/pipeline/       reference color math and golden vectors
internal/adapters/       hyprland, kwin, windows, apple
fixtures/edid/           synthetic EDID profiles
fixtures/content/        test patterns and metadata
docs/                    scenario spec and report schema
.agents/skills/          agent skill for HDR conformance work
.agents/agents/          reviewer, runner, test author, and standard editor subagents
```

## Agent resources

The repository ships one skill and four subagents so coding agents apply the same rules as human
contributors.

| Resource | File | Use |
|---|---|---|
| Skill | `.agents/skills/hdr-conformance/SKILL.md` | Normative rules, test matrix, golden values, pitfalls, and report schema |
| `hdr-spec-reviewer` | `.agents/agents/hdr-spec-reviewer.md` | Review HDR changes against the rule set |
| `hdr-benchmark-runner` | `.agents/agents/hdr-benchmark-runner.md` | Run live probes and produce a report |
| `hdr-test-author` | `.agents/agents/hdr-test-author.md` | Add tests, fixtures, and golden vectors |
| `hdr-standard-editor` | `.agents/agents/hdr-standard-editor.md` | Maintain rules, profiles, and versioning |

`AGENTS.md` at the repository root is the entry point.

## Plans

- [AW2726DM HDR plan](docs/plans/aw2726dm-hdr.md): configuration, Hyprland fixes, build, and verification for the Dell Alienware AW2726DM on DP-2.
- [AW2726DM EDID fixture](fixtures/edid/dell-aw2726dm.json): decoded capabilities used by the plan.

## Contributing

Issues and pull requests are welcome. Start with a scenario or a fixture. Keep platform-specific behavior in adapters.

## License

MIT. See [LICENSE](LICENSE).
