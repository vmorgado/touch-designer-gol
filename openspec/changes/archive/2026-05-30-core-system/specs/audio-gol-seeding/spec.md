## ADDED Requirements

### Requirement: Beat-triggered random cell seeding
On each beat pulse, approximately 15% of the 32×32 grid cells are randomly forced alive. The random pattern varies ~30 times per second to prevent fixed cells always seeding.

#### Scenario: Beat fires
- **WHEN** the `pulse` channel from `/audio/out_audio_vals` goes to 1.0
- **THEN** ~15% of cells are set alive in the seed texture for that frame

#### Scenario: No beat
- **WHEN** `pulse` is 0.0
- **THEN** the seed texture is all zero (no cells injected)

### Requirement: Seed texture delivered as Script TOP input
The seeding logic runs in a `seed_injector` GLSL TOP inside `/gol`. Its output is wired as input 1 to `gol_compute`. Audio data is bound via `array0chop` referencing `/audio/out_audio_vals`.

#### Scenario: Audio device inactive
- **WHEN** no audio device is connected and channels are absent
- **THEN** `array0chop` binding handles missing channels gracefully (no cook error)
