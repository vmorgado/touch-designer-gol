# Audio Analysis Switch Plan

**Date**: 2026-05-26 — **Status: Executed**  
**Goal**: Remove the hand-built audio analysis chain inside `/project1/audio` and replace it entirely with the `audioAnalysis` container component already present in that network.

---

## 1. Current State

### `/project1/audio` — operators that exist today

| Operator | Type | Role |
|---|---|---|
| `audio_in` | AudioDeviceIn CHOP | Live audio source |
| `bass_filter` → `bass_analyze` → `bass_rename` | AudioFilter / Analyze / Rename | Bass band RMS |
| `mid_hp` → `mid_lp` → `mid_analyze` → `mid_rename` | AudioFilter × 2 / Analyze / Rename | Mid band RMS |
| `treble_filter` → `treble_analyze` → `treble_rename` | AudioFilter / Analyze / Rename | Treble band RMS |
| `bands_merge` → `bands_norm` → `bands_limit` → `sel_bands` | Merge / Math / Limit / Select | Normalise + collect bands |
| `beat_detect` | Beat CHOP | Produces `ramp`, `pulse`, `beat` channels |
| `out_beat` | Null CHOP | Exports beat channels |
| `out_bands` | Null CHOP | Exports band channels |
| `audio_analys` | Null CHOP | Bridge: receives `audioAnalysis/out1`, feeds merge |
| `audio_vals` | Merge CHOP | Combines `sel_bands` + `out_beat` + `audio_analys` |
| `out_audio_vals` | Null CHOP | **Final output consumed by `/gol`** |
| `audioAnalysis` | Container COMP | New component — feeds `audio_analys` today |

### What `/gol` reads

`/project1/gol/sel_audio` → `op('/project1/audio/out_audio_vals')`

The two `/gol` Python scripts (`seed_top_callbacks`, `pattern_ctrl_callbacks`) each read these four channels from `sel_audio`:

| Channel | Source in old network |
|---|---|
| `bass` | `bass_analyze` → `bass_rename` |
| `mid` | `mid_analyze` → `mid_rename` |
| `treble` | `treble_analyze` → `treble_rename` |
| `pulse` | `beat_detect` (Beat CHOP) — 0 → 1 rising-edge trigger |

### What `audioAnalysis/out1` produces today

`low`, `mid`, `high`, `kick`, `snare`, `rythm`, `smsd`, `fmsd`, `spectralCentroid`

---

## 2. Decisions

| Question | Answer |
|---|---|
| Q1 — Beat pulse replacement | **`kick`** replaces `pulse` |
| Q2 — Audio source | Keep `audio_in` (AudioDeviceIn CHOP), wired into `audioAnalysis` as its container input |
| Q3 — `out2` usage | **Ignore** — `out2` (`chan1` / Neutone signal) is not needed outside `audioAnalysis` |
| Q4 — Channel rename approach | **Edit the GoL scripts directly** (`bass`→`low`, `treble`→`high`, `pulse`→`kick`) |

---

## 3. Target State

### `/project1/audio` after the switch

```mermaid
flowchart LR
    AI[audio_in\nAudioDeviceIn CHOP] --> AA[audioAnalysis\nContainer COMP]
    AA -->|out1| OAV[out_audio_vals\nNull CHOP]
    OAV --> SEL[/gol/sel_audio\nSelect CHOP]
```

Only three operators remain: `audio_in`, `audioAnalysis`, `out_audio_vals`.

### Channel mapping

| Old channel name | New channel name | Location changed |
|---|---|---|
| `bass` | `low` | GoL scripts |
| `mid` | `mid` | No change |
| `treble` | `high` | GoL scripts |
| `pulse` | `kick` | GoL scripts |

---

## 4. Operators to Delete from `/project1/audio`

- `bass_filter`, `bass_analyze`, `bass_rename`
- `mid_hp`, `mid_lp`, `mid_analyze`, `mid_rename`
- `treble_filter`, `treble_analyze`, `treble_rename`
- `bands_merge`, `bands_norm`, `bands_limit`, `sel_bands`
- `beat_detect`
- `out_beat`
- `out_bands`
- `audio_analys`
- `audio_vals`
- `out_audio_vals` (will be recreated wired to `audioAnalysis`)

---

## 5. Script Changes in `/project1/gol`

### `seed_top_callbacks`

| Old | New |
|---|---|
| `sel_audio.chan('bass')` | `sel_audio.chan('low')` |
| `sel_audio.chan('treble')` | `sel_audio.chan('high')` |
| `sel_audio.chan('pulse')` | `sel_audio.chan('kick')` |
| `total_audio = bass + mid + treble` | `total_audio = low + mid + high` (variable names updated throughout) |

### `pattern_ctrl_callbacks`

Same channel substitutions as above. The `total_audio` dead-grid recovery check also uses `bass + mid + treble` — update all three variable names.

---

## 6. Execution Steps

1. **Delete** all operators listed in §4 from `/project1/audio`.
2. **Ensure** `audio_in` is wired into `audioAnalysis` (already connected — verify it remains).
3. **Create** a new Null CHOP named `out_audio_vals` in `/project1/audio`, wired from `audioAnalysis`.
4. **Edit** `/project1/gol/seed_top_callbacks` — replace `bass`/`treble`/`pulse` channel reads with `low`/`high`/`kick`.
5. **Edit** `/project1/gol/pattern_ctrl_callbacks` — same substitutions.
6. **Verify** `/project1/gol/sel_audio` still references `/project1/audio/out_audio_vals` (path unchanged, no edit needed).
7. **Test** — play audio and confirm cells seed and patterns inject on kick hits.
8. **Update** `documentation/requirements.md` §3 (audio analysis chain) and §10 (network component table) to reflect `audioAnalysis` as the source.
