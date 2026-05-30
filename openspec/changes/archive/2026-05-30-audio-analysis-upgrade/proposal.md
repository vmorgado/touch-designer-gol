## Why

The hand-built audio analysis chain in `/audio` (21 operators: custom filter chains, beatCHOP, merge/norm/limit) was replaced by the `audioAnalysis` container component (Xavier Tremblay / Matthew Ragan / Greg Hermanovic, v6), which provides higher-quality onset detection and additional spectral channels (`kick`, `snare`, `rythm`, `fmsd`, `smsd`, `spectralCentroid`) that enable richer audio-visual mappings unavailable in the original chain.

## What Changes

- Delete all 19 hand-built operators from `/audio`; keep only `audio_in`, `audioAnalysis`, and `out_audio_vals`
- Rewire: `audio_in → audioAnalysis → out_audio_vals`
- Rename channel references in GoL scripts: `bass→low`, `treble→high`, `pulse→kick`
- New channels available: `kick`, `snare`, `rythm`, `fmsd`, `smsd`, `spectralCentroid`

## Capabilities

### New Capabilities

<!-- None -->

### Modified Capabilities

- `audio-analysis`: Replaces hand-built filter/analyze/beat chain with `audioAnalysis` container COMP as the sole analysis operator

## Impact

- `/audio` shrinks from ~21 operators to 3
- GoL scripts updated: `bass` → `low`, `treble` → `high`, `pulse` → `kick`
- `out_audio_vals` path unchanged (`/project1/audio/out_audio_vals`) — no downstream reference updates needed
- Additional channels now available for future mappings: `snare`, `rythm`, `fmsd`, `smsd`, `spectralCentroid`
