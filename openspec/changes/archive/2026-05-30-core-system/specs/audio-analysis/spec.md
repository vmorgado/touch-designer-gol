## ADDED Requirements

### Requirement: Audio input capture
Capture live audio from a microphone or line-in device via `AudioDeviceIn CHOP` (`audio_in`) inside `/audio`. No loopback or virtual audio required.

#### Scenario: Audio device active
- **WHEN** a physical audio interface is connected
- **THEN** `audio_in` produces a live audio signal each frame

### Requirement: Frequency band extraction
Split the audio signal into three named bands and compute per-frame RMS energy for each.

| Channel | Range | Filter |
|---------|-------|--------|
| `bass` | 0–180 Hz | Lowpass @ 180 Hz |
| `mid` | ~180–3500 Hz | Highpass 300 Hz + Lowpass 4 kHz in series |
| `treble` | 3500 Hz+ | Highpass @ 3500 Hz |

#### Scenario: Band energy at silence
- **WHEN** audio input is silent
- **THEN** all three band channels read 0.0

#### Scenario: Band energy at loud bass hit
- **WHEN** a loud low-frequency sound is present
- **THEN** `bass` channel approaches 1.0, `mid` and `treble` remain low

### Requirement: Beat and pulse detection
Detect rhythmic events and expose them as CHOP channels.

| Channel | Description |
|---------|-------------|
| `ramp` | Sawtooth 0→1 per beat cycle |
| `pulse` | One-frame 0→1 pulse on each beat |
| `beat` | Sustained beat indicator |

#### Scenario: Beat detected
- **WHEN** a beat onset crosses the configured threshold
- **THEN** `pulse` fires a single-frame 1.0 value; `beat` goes to 1.0; `ramp` resets

### Requirement: Unified audio output CHOP
All channels (`bass`, `mid`, `treble`, `ramp`, `pulse`, `beat`) are merged into a single `out_audio_vals` Null CHOP inside `/audio`. This is the sole output consumed by `/gol/sel_audio`.

#### Scenario: Downstream read
- **WHEN** `/gol/sel_audio` reads from `/audio/out_audio_vals`
- **THEN** all 6 channels are available as single-sample values per frame
