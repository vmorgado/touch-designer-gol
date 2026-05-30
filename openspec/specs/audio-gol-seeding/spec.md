# audio-gol-seeding Specification

## Purpose
TBD - created by archiving change core-system. Update Purpose after archive.
## Requirements
### Requirement: Beat-triggered random cell seeding
**Previous**: Frequency-band seeding ran every frame unconditionally.
**New**: All band seeding (low/mid/high random cells) is gated inside the `if beat_rising:` block in `seed_top_callbacks`. No seeding occurs on non-kick frames unless triggered by snare or rythm.

#### Scenario: Non-kick frame with loud bass
- **WHEN** `low` energy is 0.9 but no kick fires this frame
- **THEN** no band-seeded cells appear in the seed texture

### Requirement: Seed texture delivered as Script TOP input
**Previous**: Dead-grid recovery: if `low + mid + high < 0.15` for ≥ 2 seconds, a methuselah was injected.
**New**: Dead-grid recovery is removed. The grid is intentionally frozen between kicks — a frozen grid is not a dead grid. If the grid dies during a live session, the next kick will inject patterns via normal beat-triggered injection.

### Requirement: Kick-triggered category-rotating pattern injection
On each kick rising edge, `patterns_per_beat` (default 2) canonical GoL patterns are stamped at random valid positions in the grid. The pattern category rotates each kick: spaceship → oscillator → methuselah → spaceship …

Pattern library (8 patterns, encoded as cell-coordinate offsets in `pattern_library` Table DAT):

| Category | Patterns |
|----------|----------|
| Spaceship | Glider, LWSS |
| Oscillator | Blinker, Toad, Beacon, Pulsar |
| Methuselah | R-pentomino, Acorn |

#### Scenario: Kick fires, current category = spaceship
- **WHEN** kick fires and the category index is 0 (spaceship)
- **THEN** `patterns_per_beat` patterns from {Glider, LWSS} are stamped at random positions; category advances to oscillator

### Requirement: Snare-triggered oscillator injection
On each snare rising edge (independent of kick), one oscillator-category pattern is stamped at a random grid position. Fires whether or not the grid is frozen.

#### Scenario: Snare fires
- **WHEN** `snare` channel fires a rising edge
- **THEN** one pattern from {Blinker, Toad, Beacon, Pulsar} appears in the seed texture

### Requirement: Rythm-triggered quadrant seeding
On each rythm rising edge (independent of kick), a random 16×16 quadrant of the grid (one of four corners) is seeded at approximately 50% density.

#### Scenario: Rythm fires
- **WHEN** `rythm` channel fires a rising edge
- **THEN** ~50% of a randomly chosen 16×16 corner quadrant of cells are set alive in the seed texture

### Requirement: Dead-grid recovery
If total audio energy (`low + mid + high`) stays below 0.15 for ≥ 2 continuous seconds, one methuselah pattern (R-pentomino or Acorn) is injected at a random position. The timer resets on any kick event.

#### Scenario: Prolonged silence
- **WHEN** audio energy stays below threshold for 2 seconds
- **THEN** one methuselah is injected into the seed texture on that frame

### Requirement: Seed texture is always applied (snare/rythm persist in frozen grid)
The GoL pixel shader always applies the seed layer (`max(next, seed)`) regardless of the `uAdvance` flag. Seeds from snare and rythm persist in the frozen grid state and evolve when the next kick fires.

### Requirement: Seeding implemented in Script TOP
All seeding logic runs in `seed_top` Script TOP (`seed_top_callbacks` cook()). A `noise_trigger` Noise TOP wired as input 0 ensures `seed_top` cooks every frame.

