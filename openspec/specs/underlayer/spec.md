# underlayer Specification

## Purpose
TBD - created by archiving change core-system. Update Purpose after archive.
## Requirements
### Requirement: Switchable underlayer source
The underlayer is a full-resolution TOP switchable between two modes at runtime via `ctrl_switch` Constant CHOP (channel `index`; set 0 or 1).

| Index | Mode | Operator |
|-------|------|----------|
| 0 | Video / Media | `video_src` Movie File In TOP |
| 1 | Generative | `gen_src` GLSL TOP (Perlin noise field) |

#### Scenario: Switch to generative
- **WHEN** `ctrl_switch.par.value0` is set to 1
- **THEN** `src_switch` Switch TOP re-cooks and outputs the generative shader

#### Scenario: Switch to video
- **WHEN** `ctrl_switch.par.value0` is set to 0
- **THEN** `src_switch` outputs the video source without interrupting the GoL simulation

### Requirement: Generative shader — audio-driven Perlin noise
`gen_src` (GLSL TOP) generates an animated Perlin noise colour field. Two audio channels modulate it via 1×1 `choptoTOP` textures:

| Channel | Uniform | Effect |
|---------|---------|--------|
| `spectralCentroid` | `uHueShift` (× 0.5) | Shifts base hue of the colour field |
| `smsd` | `uAnimSpeed` (abs × 0.15, clamped 0.02–0.3) | Controls auto-rotation speed of hue cycling |

#### Scenario: Bright treble-heavy audio
- **WHEN** `spectralCentroid` is high
- **THEN** `uHueShift` rotates the colour field toward the warm end of the spectrum

### Requirement: Underlayer output boundary
`out_underlayer` Null TOP is the sole output of `/underlayer`, consumed by `cube_mat`'s `sUnderlayer` sampler.

### Requirement: Hot-swappable sources
Both sources run simultaneously; switching between them does not interrupt the GoL simulation or audio analysis.

### Requirement: Video source file configurable at runtime
`video_src.par.file` can be set to any video clip path at runtime. The clip loops continuously.

### Requirement: Switch via CHOP (not custom COMP parameter)
Custom COMP parameters do not register as cook dependencies on downstream TOPs. The switch index must come from a CHOP to ensure `switchTOP` re-cooks when the value changes.

