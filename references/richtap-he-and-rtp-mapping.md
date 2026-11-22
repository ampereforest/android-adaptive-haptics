# RichTap HE 1.0/2.0 & Hardware RTP Multi-Texture Mapping Reference

## 1. AAC RichTap HE `int32[]` Wire Format

Used by ColorOS/Oplus `IOplusVibrator::performHe(int looper, int interval, int amplitude, int freq, int[] pattern, IBinder callback)`:

### HE 1.0 (`pattern[0] == 1`)
- Header: `pattern[0] = 1`
- Stride: `17` `int32` values per event starting at `pos = 1`:
  - `pattern[pos + 0]`: `type` (`0x1001` = transient, `0x1000` = continuous)
  - `pattern[pos + 1]`: `relativeTimeMs`
  - `pattern[pos + 2]`: `intensity` (`1..100`)
  - `pattern[pos + 3]`: `frequency` (`0..100`)
  - `pattern[pos + 4]`: `durationMs` (for continuous `0x1000`; transients use `~48ms`)

### HE 2.0 (`pattern[0] == 2`)
- Header (8 `int32`s):
  - `pattern[0] = 2` (version)
  - `pattern[6] = absTimeMs` (first pattern absolute offset)
  - `pattern[7] = numEvents` (event count in first pattern)
- Event blocks starting at `idx = 8`:
  - `pattern[idx + 0]`: `type` (`0x1000` continuous / `0x1001` transient)
  - `pattern[idx + 1]`: `blockLen` (payload ints following `[type, blockLen]`; next block at `idx + 2 + blockLen`)
  - `pattern[idx + 2]`: event index
  - `pattern[idx + 3]`: `relativeTimeMs` (offset within current pattern; absolute time = `absTimeMs + relativeTimeMs`)
  - `pattern[idx + 4]`: `intensity` (`1..100`)
  - `pattern[idx + 5]`: `frequency` (`0..100`)
  - `pattern[idx + 6]`: `durationMs` (for `0x1000` continuous)
- Multi-pattern continuation: after consuming `numEvents`, if `next_idx + 3 < pattern.size()`, read a 3-int pattern subheader `[patternId, absTimeMs, numEvents]` and continue at `next_idx + 3`.

---

## 2. Adaptive Frequency/Intensity -> Hardware RTP Texture Matrix

When translating RichTap HE events onto a device with rich pre-baked hardware RTP waveforms (such as Xiaomi `/odm/firmware/*_P_RTP.bin` on `qcom-hv-haptics`), route each event by `(type, frequency, duration, intensity)` and apply `eventAmp = clamp((globalAmp / 255.0f) * (event.intensity / 100.0f), 0.08f, 1.0f)` **before** `perform()`:

| Event Type | Frequency / Duration Band | Intensity | Tactile Material Character | Example Hardware RTP Target |
| :--- | :--- | :--- | :--- | :--- |
| Transient (`0x1001`) | `freq >= 78` | `< 45` | Ultra-light crisp micro-tick | `181_signalKeyUnit` |
| Transient (`0x1001`) | `freq >= 78` | `>= 45` | Bright crystalline tap | `56_Crystal` |
| Transient (`0x1001`) | `60 <= freq <= 77` | `< 70` | Crisp tactile click | `180_signalKeyHigh` |
| Transient (`0x1001`) | `60 <= freq <= 77` | `>= 70` | Metallic crisp clank | `55_Clank` |
| Transient (`0x1001`) | `42 <= freq <= 59` | `< 65` | Organic bubble pop | `51_Bubble` |
| Transient (`0x1001`) | `42 <= freq <= 59` | `>= 65` | Punchy pop | `4_pop` |
| Transient (`0x1001`) | `25 <= freq <= 41` | `< 65` | Mechanical switch detent | `187_signalSwitch` |
| Transient (`0x1001`) | `25 <= freq <= 41` | `>= 65` | Elastic boundary knock | `201_BoundaryBounce` |
| Transient (`0x1001`) | `freq < 25` | `< 75` | Deep low-frequency thud | `3_thud` |
| Transient (`0x1001`) | `freq < 25` | `>= 75` | Heavy bass impact | `5_heavyClick` |
| Continuous (`0x1000`) | `dur >= 180ms`, `freq >= 60` | any | Elastic resonant spring | `70_Spring` |
| Continuous (`0x1000`) | `dur >= 180ms`, `35..59` | any | Smooth gradual swell | `57_FadeIn` |
| Continuous (`0x1000`) | `dur >= 180ms`, `freq < 35` | any | Deep directional swoosh | `71_Swoosh` |
| Continuous (`0x1000`) | `70..179ms`, `freq >= 60` | any | Vibrant shimmer burst | `53_Burst` |
| Continuous (`0x1000`) | `70..179ms`, `freq < 60` | any | Textured sliding damping | `170_slidingDamping` |
| Continuous (`0x1000`) | `dur < 70ms` | any | Short detent / bottom-out | `186_signalSeekbar` / `202_BoundaryBottom` |

---

## 3. System UI Multi-Texture Mapping Guidelines

Avoid collapsing all UI touch waveforms onto `0_click`:

| System Interaction Category | Recommended Tactile Profile | Example RTP Mapping |
| :--- | :--- | :--- |
| Keyboard (weak / medium / strong) | Crisp, light, zero-ringing keystrokes | `181_signalKeyUnit`, `180_signalKeyHigh`, `173_signalButtonHighlight` |
| Standard UI Button Tap | Clean crisp button click | `175_signalButton` |
| Toggle Switch / Checkbox | Distinct mechanical snap | `187_signalSwitch` |
| Slider / Seekbar Detent | Fine rhythmic step | `186_signalSeekbar` |
| Wheel / Time Picker Tick | Micro-granular tick | `184_signalPicker`, `178_signalClockUnit` |
| Overscroll / Edge Bounce | Elastic rebound | `201_BoundaryBounce`, `202_BoundaryBottom`, `203_BoundaryTop`, `70_Spring` |
| Long Press / 3D Touch | Deep pop / firm press | `4_pop`, `410_longpress` |
| Gesture Back Pull / Release | Directional slide cues | `162_GestureBackPull`, `163_GestureBackRelease`, `170_slidingDamping` |
| Notifications | Diverse organic/musical textures | `51_Bubble`, `55_Clank`, `56_Crystal`, `59_Flute`, `63_Harp`, `64_IncomingMessage`, `67_NotificationXylophone` |

---

## 4. Event-Driven Scheduler & Cancellation Hygiene

- Run a single detached worker thread waiting on `pthread_cond_wait(&cv, &mu)` initialized with `CLOCK_MONOTONIC` (`pthread_condattr_setclock`).
- Protect playback state with a monotonic `uint64_t generation` counter:
  - Starting a new pattern or receiving `IVibrator::off()` / `IOplusVibrator::stop()` increments `generation`, calls `pthread_cond_broadcast(&cv)`, and immediately stops the hardware actuator.
  - Between events, sleep via `pthread_cond_timedwait(&cv, &mu, &abs_target_ts)` and abort immediately if `generation != my_gen`.
- Debounce ultra-dense micro-events (`< 12ms` apart) so the LRA actuator cleanly completes each impulse without clipping.
- Fire `IRichtapCallback` (`code=1`, `FLAG_ONEWAY`, `int32(0)`) upon normal pattern completion.
