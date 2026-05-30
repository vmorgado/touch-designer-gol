## 1. CHOP Chain for uAdvance

- [x] 1.1 Create `kick_sel` Select CHOP in `/gol`; set CHOP = `/project1/audio/out_audio_vals`, Channel = `kick`
- [x] 1.2 Create `kick_pulse` Logic CHOP; set function = Pulse on Rising Edge (preop = rise); wire from `kick_sel`
- [x] 1.3 Create `kick_gate` Null CHOP; wire from `kick_pulse`
- [x] 1.4 Create `kick_top` choptoTOP (1×1); wire from `kick_gate`

## 2. gol_compute Shader Update

- [x] 2.1 Add CHOP uniform `uAdvance` to `gol_compute`; reference `kick_gate`, channel `chan1`
- [x] 2.2 Edit `gol_compute_pixel` shader: add `uniform float uAdvance;`
- [x] 2.3 Wrap Conway rules in `if (uAdvance >= 0.5) { ... } else { next = current; }`
- [x] 2.4 Ensure seed injection (`next = max(next, seed)`) is inside the `uAdvance >= 0.5` branch only — seed always applied in any case (separate from advance logic)

## 3. seed_top_callbacks Script Update

- [x] 3.1 Move band seeding (`n_low`, `n_mid`, `n_high` random cells) inside `if beat_rising:` block
- [x] 3.2 Remove dead-grid recovery block (the `total_audio < 0.15` / `dead_timer` logic)
- [x] 3.3 Remove `dead_timer` from `scriptOp.fetch` and `scriptOp.store` calls

## 4. Verification

- [x] 4.1 With no audio: grid is completely frozen (no state changes frame to frame)
- [x] 4.2 On kick hit: grid advances exactly one generation and freezes again
- [x] 4.3 Snare/rythm: cells appear in frozen grid and persist until next kick
