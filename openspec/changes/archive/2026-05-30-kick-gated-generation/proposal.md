## Why

The GoL simulation advancing every frame (~60 generations/second) makes the grid evolve too fast for live VJ use — patterns are unreadable before they are overwritten. Synchronising generation advancement to kick hits makes the evolution tempo track the music and creates a direct visual-audio relationship.

## What Changes

- GoL simulation advances exactly **one generation per kick hit**; the grid is completely frozen between kicks
- A CHOP chain converts the sustained kick envelope into a single-frame rising-edge pulse (`uAdvance` uniform)
- Frequency-band seeding and pattern injection only fire on kick frames (not every frame)
- Dead-grid recovery removed — a frozen grid between kicks is not a dead grid; the next kick will inject patterns

## Capabilities

### New Capabilities

<!-- None — modifies existing capabilities -->

### Modified Capabilities

- `gol-simulation`: Generation advancement is now kick-gated (one generation per kick), not every-frame
- `audio-gol-seeding`: Band seeding and pattern injection only fire on kick rising-edge frames; dead-grid recovery removed

## Impact

- New CHOP chain in `/gol`: `kick_sel → kick_pulse (Logic CHOP, rise mode) → kick_gate (Null) → kick_top (choptoTOP)`
- `gol_compute` GLSL TOP: new `uAdvance` CHOP uniform; shader branches on this flag
- `seed_top_callbacks`: band seeding moved inside `if beat_rising:` block; dead-grid recovery block removed
- No change to feedback loop topology, pattern library, or rendering
