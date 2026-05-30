## Context

The hand-built chain had several correctness issues (mid-band required two filters, beatCHOP needed explicit channel enables, mathCHOP had no clamp). The `audioAnalysis` container encapsulates all of this with tunable thresholds per band, eliminating 18+ operators and providing onset detection channels that the original `beatCHOP` alone couldn't supply.

## Goals / Non-Goals

**Goals:**
- Single source of truth for audio analysis in `/audio`
- Maintain the same `out_audio_vals` Null CHOP output path so no downstream operator references change
- Expose `kick`, `snare`, `rythm` onset channels for use by `/gol` scripts
- Expose `fmsd`, `smsd`, `spectralCentroid` for camera shake and underlayer modulation

**Non-Goals:**
- Using `audioAnalysis` `out2` (`chan1` / Neutone signal) — not needed
- Custom re-implementation of any `audioAnalysis`-internal analysis

## Decisions

### Keep audio_in as the physical device operator
`audio_in` (AudioDeviceIn CHOP) remains as the connection to the physical audio interface. It is wired into `audioAnalysis` as its container input. This keeps the device selection parameter accessible outside the container.

### Channel rename in scripts (not in network)
Rather than inserting a Rename CHOP in the network to bridge old names to new, the GoL scripts are edited directly:
- `bass` → `low`
- `treble` → `high`
- `pulse` → `kick`

This keeps the network minimal and makes the script channel names match `audioAnalysis` output names exactly.

### kick replaces pulse for beat detection
`audioAnalysis` exposes `kick` as its primary beat channel (threshold-based onset detection on the low band). The previous `beatCHOP.pulse` (one-frame pulse on beat) is functionally equivalent. The Logic CHOP `kick_pulse` in `/gol` converts the `kick` envelope to a rising-edge pulse.

## Risks / Trade-offs

- `audioAnalysis` threshold controls live inside the container (per-band sliders). If kick/snare/rythm fire too often or rarely, thresholds must be adjusted inside the container, not at the `/audio` network level.
- `par.val` vs `par.eval()` confusion: after changing a custom parameter inside `audioAnalysis`, verify effects with `par.eval()` not `par.val`, as the latter may cache a stale value until the operator re-cooks.
