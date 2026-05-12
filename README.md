# LED Audio Spectrum Analyzer — LEA

An audio-reactive spectrum analyzer running on an ESP32 with a 16×16 WS2812B LED matrix. The firmware performs real-time FFT on microphone input, maps frequency bins to 16 display bands with automatic gain control, and switches between a color spectrum display and a radial idle glow animation based on adaptive silence detection.

---

## Hardware

| Component | Detail |
|-----------|--------|
| Microcontroller | ESP32 (any standard dev board) |
| LED matrix | 16×16 WS2812B (256 LEDs total) |
| Microphone | Analog electret mic module (e.g. MAX4466 or KY-038) |
| LED data pin | GPIO 5 |
| Mic analog pin | GPIO 34 |
| LED power | 5 V, up to ~5 A depending on brightness |
| MCU power | 3.3 V / USB |

## Wiring

```
ESP32 GPIO 5  ──────────────► WS2812B matrix DIN
ESP32 GPIO 34 ◄────────────── Mic module OUT
ESP32 GND     ──────────────── Shared ground (matrix, mic)
5V supply     ──────────────── Matrix VCC
3.3V          ──────────────── Mic module VCC
```

Add a 300–500 Ω resistor in series on the LED data line and a 100–1000 µF bulk capacitor across the LED power rails to suppress transients.

---

## Libraries Required

Install both via the Arduino Library Manager or PlatformIO:

- **FastLED** — LED driving and HSV color math
- **ArduinoFFT** (`arduinoFFT.h`) — FFT computation

---

## Configuration and Tuning Parameters

All tunable constants are grouped at the top of the source file.

### FFT and Sampling

| Constant | Value | Description |
|----------|-------|-------------|
| `SAMPLES` | 128 | FFT window size |
| `SAMPLING_FREQ` | 10 000 Hz | ADC sample rate; sets max frequency to 5 kHz |
| `FFT_NOISE_GATE` | 20.0 | Minimum FFT bin magnitude; bins below this are zeroed to suppress electrical noise |

Sampling is software-timed via `micros()` to achieve exactly 10 kHz without a timer ISR. A Hamming window is applied before the FFT.

### Frequency Band Mapping

Sixteen bands are defined by a custom bin-edge table that skips FFT bins 0 and 1 (DC and near-DC):

```
bandEdges[17] = { 2, 3, 4, 5, 6, 7, 9, 11, 14, 17, 20, 24, 28, 33, 38, 44, 52 }
```

Each band's magnitude is the average of the FFT bins within its range. The band spacing is roughly logarithmic, concentrating resolution in the lower frequencies where musical content is dominant.

### AGC (Automatic Gain Control)

| Constant | Value | Description |
|----------|-------|-------------|
| `SENSITIVITY_GAIN` | 4.5 | Multiplier applied after normalization |
| `levelMax` (initial) | 650.0 | Starting AGC ceiling |
| `AGC_RISE` | 0.25 | Attack rate — how fast the ceiling rises to a new peak |
| `AGC_FALL` | 0.008 | Decay rate — how slowly the ceiling falls when signal is quiet |
| `LEVELMAX_FLOOR` | 120.0 | Minimum AGC ceiling; prevents the display from going full-scale on noise |

The AGC ceiling rises quickly when a loud signal exceeds it (`AGC_RISE = 0.25`) and decays very slowly (`AGC_FALL = 0.008`), maintaining useful dynamic range across a wide range of playback volumes and source distances. The floor of 120 allows quiet or distant audio to still produce visible bars.

### Bar Smoothing and Peak Dots

| Constant | Value | Description |
|----------|-------|-------------|
| `SMOOTHING` | 0.70 | IIR smoothing coefficient (higher = smoother/slower) |
| `PEAK_HOLD_FRAMES` | 5 | Frames a peak dot stays at its maximum before starting to fall |
| `PEAK_FALL_RATE` | 3 | Pixels per frame the peak dot falls during active audio |
| `SILENT_PEAK_FALL_RATE` | 5 | Faster fall rate during silence |

### Silence Detection

See the [Silence Detection](#silence-detection) section below for the full description of thresholds and debounce parameters.

---

## Display Modes

### Spectrum Mode

Activated when audio amplitude exceeds the enter threshold. Each of the 16 columns displays a bar whose height corresponds to the energy in that frequency band, scaled by the AGC and smoothed by the IIR filter. Color is mapped by vertical position using HSV hue values 0–160 (red at the bottom through green at the top). A white dot marks the peak level for each band, holding for `PEAK_HOLD_FRAMES` frames then falling at `PEAK_FALL_RATE` pixels per frame.

### Idle Glow Mode

Active during silence. Displays a radial breathing animation across the full 16×16 matrix:

- **Breathing:** brightness oscillates between `IDLE_GLOW_MIN` (18) and `IDLE_GLOW_MAX` (90) on a sine wave with a period of 2600 ms.
- **Radial falloff:** LEDs closer to the matrix center (7, 7) are brighter than corner LEDs, eliminating the flat-stripe look of a uniform fill.
- **Hue drift:** the base hue increments by 1 every 40 ms, slowly cycling through the full color wheel. Each LED also adds a distance-based hue offset (`d² >> 1`), producing gentle concentric color rings.
- **Saturation:** fixed at 170 (slightly desaturated for a warmer glow appearance).

---

## Silence Detection

Mode switching uses a three-layer system to avoid false triggers and chattering:

**1. Amplitude smoothing.** The mean-absolute-value amplitude of each 128-sample frame is exponentially smoothed with `AMP_DECAY = 0.90`, giving a sluggish signal that does not react to individual transients.

**2. Adaptive noise baseline.** A separate baseline tracks the ambient noise floor using a slow exponential filter (`BASELINE_ALPHA_SLOW = 0.0015`). During the first 2500 ms after boot (warmup), a faster rate (`BASELINE_ALPHA_FAST = 0.02`) is used to learn the environment quickly. The baseline only updates when the smoothed amplitude is below the exit threshold, so music does not poison the noise floor estimate.

**3. Hysteresis thresholds.** Enter and exit thresholds are derived from the baseline each frame:

```
enterThreshold = noiseBaseline × 1.25 + 5.0    (minimum 25)
exitThreshold  = noiseBaseline × 1.50 + 5.0    (minimum 18)
```

The exit threshold is always higher than the enter threshold, creating a hysteresis band that prevents rapid mode flapping.

**4. Debounce.** The amplitude must remain above `enterThreshold` for 4 consecutive frames before switching to spectrum mode, and must remain below `exitThreshold` for 8 consecutive frames before returning to idle. Either counter resets immediately if the condition breaks.

---

## Boot Sequence

On power-up the firmware scrolls "LEA MARIE" across the matrix three times using a built-in 5×7 bitmap font before entering the main loop. Each letter is rendered in a hue derived from its position and loop index, producing a shifting rainbow effect across the message. Spacing between letters is 6 pixels (5-pixel glyph + 1-pixel gap). The `NULL` entry in the message array inserts a blank space between "LEA" and "MARIE".

---

## Flashing Instructions

1. Rename `Spectrum Analyzer Code (LEA).txt` to `SpectrumAnalyzer.ino` (or place it inside a folder of the same name).
2. Open in the Arduino IDE and select your ESP32 board under **Tools → Board → ESP32 Arduino**.
3. Install **FastLED** and **ArduinoFFT** via **Sketch → Include Library → Manage Libraries**.
4. Select the correct COM port and click **Upload**.

The firmware prints nothing to Serial after startup (Serial is initialized at 115200 but only used for debugging if you add print statements). The boot scroll begins immediately on power-up.
