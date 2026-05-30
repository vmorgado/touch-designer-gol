## Context

The existing `seed_injector` GLSL TOP handles beat-only seeding with a fixed 15% density uniform random scatter. It cannot express structured patterns (glider, pulsar, etc.) that require specific cell-coordinate offsets stamped at random grid positions.

## Goals / Non-Goals

**Goals:**
- Canonical GoL patterns injected on beat with category rotation for visual variety
- Frequency-band seeding re-enabled (all on kick frame, not per-frame)
- Independent snare and rythm triggers for additional seeding variety
- Dead-grid safety net via methuselah injection on silence

**Non-Goals:**
- Weighted pattern selection based on real-time grid activity
- Per-pattern hold time longer than 1 frame (configurable but defaulting to 1)
- `pattern_ctrl` Script CHOP is retained in the network but not connected to `seed_top`'s data path

## Decisions

### All seeding logic consolidated into seed_top cook()
The originally planned two-operator pipeline (`pattern_ctrl` → `pattern_state` → `seed_top`) was collapsed into `seed_top`'s own `cook()` script. The Script TOP reads `sel_audio` (CHOP), `pattern_library` (DAT), and `ctrl_seeds` (CHOP) directly in a single Python pass and writes a numpy array. `pattern_ctrl` and `pattern_state` remain in the network as debug/monitoring artifacts but are not used by `seed_top`.

### noise_trigger Noise TOP as cook-dependency trigger
Script TOPs only re-cook when their TOP inputs change. `noise_trigger` (Noise TOP) is wired as input 0 to `seed_top`. It is time-dependent so it cooks every frame, forcing `seed_top` to cook. Its pixel values are never read by the script.

### Three seeding passes in one numpy array
`seed_top.cook()` runs three passes per frame, all writing into a single 32×32 float32 numpy array:
1. **Freq-band seeding** (on kick frame): `int(low × 20) + int(mid × 12) + int(high × 8)` random positions
2. **Pattern injection** (on kick rising edge): `patterns_per_beat` patterns from the current category, placed at random valid offsets within the 32×32 grid
3. **Dead-grid recovery** (if `total_audio < 0.15` for ≥ 2 s): one random methuselah

Category rotation: spaceship → oscillator → methuselah → spaceship …

### Y-axis coordinate convention
`texelFetch` uses y=0 at texture bottom (OpenGL convention). `copyNumpyArray` row 0 also maps to texture bottom. Pattern coordinates `(x, y)` map directly to `arr[y, x]` — no flip needed.

### ctrl_seeds defaults (actual)
| Channel | Default | Used |
|---------|---------|------|
| `patterns_per_beat` | 2 | Yes |
| `hold_frames` | 1 | No (seed applies for 1 frame only) |
| `fallback_timeout` | 3.0 | No (hardcoded 2.0 s in script) |
| `activity_threshold` | 0.02 | No (dead-grid uses audio level, not grid activity) |

## Risks / Trade-offs

- `pattern_ctrl` Script CHOP and `pattern_state` Table DAT are present but disconnected from `seed_top`'s data path. They are debug artifacts from an earlier iteration and can be ignored or removed in a future cleanup pass.
- Dead-grid recovery uses `total_audio < 0.15` as the threshold (sum of `low + mid + high`), not grid cell count. A grid that is alive but silent will also trigger recovery — this is intentional for live performance resilience.
