## 1. Delete Hand-Built Chain

- [x] 1.1 Delete from `/audio`: `bass_filter`, `bass_analyze`, `bass_rename`
- [x] 1.2 Delete: `mid_hp`, `mid_lp`, `mid_analyze`, `mid_rename`
- [x] 1.3 Delete: `treble_filter`, `treble_analyze`, `treble_rename`
- [x] 1.4 Delete: `bands_merge`, `bands_norm`, `bands_limit`, `sel_bands`
- [x] 1.5 Delete: `beat_detect`, `out_beat`, `out_bands`, `audio_analys`, `audio_vals`
- [x] 1.6 Delete old `out_audio_vals` (will be recreated)

## 2. Rewire Audio Chain

- [x] 2.1 Verify `audio_in` is wired into `audioAnalysis` container input (already connected — confirm it remains after deletes)
- [x] 2.2 Create new `out_audio_vals` Null CHOP in `/audio`; wire from `audioAnalysis` output 1

## 3. Update GoL Scripts

- [x] 3.1 Edit `seed_top_callbacks`: replace `sel_audio.chan('bass')` → `sel_audio.chan('low')`
- [x] 3.2 Edit `seed_top_callbacks`: replace `sel_audio.chan('treble')` → `sel_audio.chan('high')`
- [x] 3.3 Edit `seed_top_callbacks`: replace `sel_audio.chan('pulse')` → `sel_audio.chan('kick')`
- [x] 3.4 Edit `pattern_ctrl_callbacks` (same substitutions as above)

## 4. Verification

- [x] 4.1 Verify `/gol/sel_audio` still references `/project1/audio/out_audio_vals` (path unchanged)
- [x] 4.2 Play audio — confirm cells seed and patterns inject on kick hits
- [x] 4.3 Confirm `snare`, `rythm`, `fmsd`, `smsd`, `spectralCentroid` channels visible in `out_audio_vals`
