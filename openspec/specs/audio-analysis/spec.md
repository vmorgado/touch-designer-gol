# audio-analysis Specification

## Purpose
TBD - created by archiving change core-system. Update Purpose after archive.
## Requirements
### Requirement: Audio input capture
**Previous**: Hand-built chain of 21 operators (AudioFilter, Analyze, Rename, Merge, Math, Limit, Beat CHOP).
**New**: `audioAnalysis` container COMP (Xavier Tremblay / Matthew Ragan / Greg Hermanovic, v6) connected as: `audio_in → audioAnalysis → out_audio_vals`.

Only three operators remain in `/audio`: `audio_in`, `audioAnalysis`, `out_audio_vals`.

### Requirement: Frequency band extraction
**Previous**: `bass`, `mid`, `treble`
**New**: `low`, `mid`, `high`

All GoL scripts reference the new names directly.

### Requirement: Beat and pulse detection
**Previous**: `pulse` (from beatCHOP, one-frame pulse)
**New**: `kick` (from audioAnalysis, threshold-based onset detection on the low band)

`kick` is converted to a rising-edge pulse by `kick_pulse` Logic CHOP in `/gol`.

### Requirement: Unified audio output CHOP
All channels (`bass`, `mid`, `treble`, `ramp`, `pulse`, `beat`) are merged into a single `out_audio_vals` Null CHOP inside `/audio`. This is the sole output consumed by `/gol/sel_audio`.

#### Scenario: Downstream read
- **WHEN** `/gol/sel_audio` reads from `/audio/out_audio_vals`
- **THEN** all 6 channels are available as single-sample values per frame

### Requirement: Additional onset and spectral channels
`audioAnalysis` exposes additional channels beyond the basic frequency bands:

| Channel | Description |
|---------|-------------|
| `kick` | Kick drum onset — low-frequency transient, 0/1 pulse |
| `snare` | Snare onset — mid-frequency transient, 0/1 pulse |
| `rythm` | Broader rhythmic onset, 0/1 pulse |
| `fmsd` | Fast Moving Standard Deviation — spikes on sudden transients |
| `smsd` | Slow Moving Standard Deviation — reflects broad dynamics |
| `spectralCentroid` | Centre of mass of frequency spectrum, 0–1 normalised |

All channels are exported via `out_audio_vals` and available to downstream operators.

### Requirement: Threshold controls inside audioAnalysis container
Per-band gain, smoothing, and threshold sliders live inside `audioAnalysis`. If `kick`, `snare`, or `rythm` fire too often or rarely, adjust the corresponding threshold inside the container — not at the `/audio` network level.

