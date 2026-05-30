## ADDED Requirements

### Requirement: fmsd-driven camera X-axis shake
`cam1`'s `tx` parameter is driven by an expression that multiplies scaled `fmsd` energy by a fast noise signal for random directional jitter.

CHOP chain: `fmsd_sel (Select) → fmsd_math (abs × 0.2) → fmsd_limit (clamp 0–0.5)`

`cam1.par.tx` expression: `op('fmsd_limit').chan('chan1')[0] * op('shake_noise').chan('chan1')[0]`

`shake_noise` is a Noise CHOP at 20 Hz period.

#### Scenario: Loud transient (high fmsd)
- **WHEN** `fmsd` spikes above 0.1 on a sharp audio attack
- **THEN** `cam1.tx` jitters in a random direction with magnitude up to 0.5

#### Scenario: Silence (fmsd near zero)
- **WHEN** audio input is silent
- **THEN** `cam1.tx` remains near 0 (no shake)

### Requirement: Shake is magnitude-only on X axis
Only the X axis (`tx`) is driven by camera shake. `ty`, `tz`, and rotation parameters are not affected.

### Requirement: fmsd absolute value before scaling
`fmsd` can be negative (variance-derived signal). `fmsd_math` takes the absolute value before multiplying by 0.2 to ensure shake magnitude is always non-negative.
