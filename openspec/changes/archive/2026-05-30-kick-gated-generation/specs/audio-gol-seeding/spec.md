## MODIFIED Requirements

### Requirement: Beat-triggered random cell seeding
**Previous**: Frequency-band seeding ran every frame unconditionally.
**New**: All band seeding (low/mid/high random cells) is gated inside the `if beat_rising:` block in `seed_top_callbacks`. No seeding occurs on non-kick frames unless triggered by snare or rythm.

#### Scenario: Non-kick frame with loud bass
- **WHEN** `low` energy is 0.9 but no kick fires this frame
- **THEN** no band-seeded cells appear in the seed texture

### Requirement: Seed texture delivered as Script TOP input
**Previous**: Dead-grid recovery: if `low + mid + high < 0.15` for ≥ 2 seconds, a methuselah was injected.
**New**: Dead-grid recovery is removed. The grid is intentionally frozen between kicks — a frozen grid is not a dead grid. If the grid dies during a live session, the next kick will inject patterns via normal beat-triggered injection.
