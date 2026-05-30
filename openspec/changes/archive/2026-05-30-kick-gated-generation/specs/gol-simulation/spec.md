## MODIFIED Requirements

### Requirement: Conway's Game of Life rules applied via GLSL pixel shader
**Previous**: One generation computed per frame (~60 fps).
**New**: GoL advances exactly **one generation per kick hit**. Between kicks the grid is completely frozen (current state passed through unchanged).

The `gol_compute` pixel shader reads `uAdvance` uniform:
- `uAdvance ≥ 0.5` → apply Conway rules + seed injection for one generation
- `uAdvance < 0.5` → output current state unchanged (frozen)

`uAdvance` is driven by `kick_top` (1×1 choptoTOP from `kick_gate` Null CHOP).

#### Scenario: Kick fires
- **WHEN** `kick_pulse` Logic CHOP outputs 1.0 on a kick rising edge
- **THEN** `uAdvance = 1.0` for that frame; GoL advances one generation

#### Scenario: No kick
- **WHEN** `kick_pulse` is 0.0
- **THEN** `uAdvance = 0.0`; `gol_compute` outputs current state unchanged

## ADDED Requirements

### Requirement: Kick rising-edge pulse generation
`kick_sel` (Select CHOP) reads the `kick` channel from `/audio/out_audio_vals`. `kick_pulse` (Logic CHOP, preop = rise) converts the sustained kick envelope into a guaranteed single-frame 1.0 pulse per hit. `kick_gate` (Null CHOP) is the clean endpoint referenced by the `uAdvance` uniform.
