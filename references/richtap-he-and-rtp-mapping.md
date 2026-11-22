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

## 2. Adaptive Frequency/Intensity -> Hardware RTP Texture Matrix (24 kHz Instant-Onset Verified)

When translating RichTap HE events onto a device with pre-baked hardware RTP waveforms (such as Xiaomi `/odm/firmware/*_P_RTP.bin` on `qcom-hv-haptics` at `24 kHz` = `24 bytes/ms`), **audit every target binary's PCM envelope** to verify **`onset_ms <= 2.1 ms`** so rapid HE beats or short framework `mDuration` windows never clip silent. Route each event by `(type, frequency, duration, intensity)` and apply `eventAmp = clamp((globalAmp / 255.0f) * (event.intensity / 100.0f), 0.08f, 1.0f)` **before** `perform()`:

| Event Type | Frequency / Duration Band | Intensity | Tactile Material Character | Verified Instant-Onset RTP Target (`Onset <= 2.1ms`) |
| :--- | :--- | :--- | :--- | :--- |
| Transient (`0x1001`) | `freq >= 78` | `< 45` | Ultra-light crisp micro-tick | `178_signalClockUnit` (`12.2ms`, `0.0ms` onset) |
| Transient (`0x1001`) | `freq >= 78` | `>= 45` | Crisp bright mechanical click | `181_signalKeyUnit` (`13.0ms`, `0.0ms` onset) |
| Transient (`0x1001`) | `60 <= freq <= 77` | `< 70` | Elastic crisp bounce | `201_BoundaryBounce` (`13.3ms`, `0.6ms` onset) |
| Transient (`0x1001`) | `60 <= freq <= 77` | `>= 70` | Punchy bright pop (`412Hz`) | `4_pop` (`15.8ms`, `0.6ms` onset) |
| Transient (`0x1001`) | `42 <= freq <= 59` | `< 65` | Clean neutral tap (`314Hz`) | `166_feedbackNeutral` (`14.3ms`, `2.0ms` onset) |
| Transient (`0x1001`) | `42 <= freq <= 59` | `>= 65` | Mid-pitch crisp tick (`226Hz`) | `2_tick` (`22.2ms`, `0.6ms` onset) |
| Transient (`0x1001`) | `25 <= freq <= 41` | `< 65` | Textured tab click | `188_signalTab` (`25.1ms`, `1.0ms` onset) |
| Transient (`0x1001`) | `25 <= freq <= 41` | `>= 65` | Directional release snap | `163_GestureBackRelease` (`23.3ms`, `2.1ms` onset) |
| Transient (`0x1001`) | `freq < 25` | `< 75` | Deep low-freq toggle (`98Hz`) | `88_launcherIconSelection` (`15.2ms`, `2.0ms` onset) |
| Transient (`0x1001`) | `freq < 25` | `>= 75` | Heavy boundary impact | `202_BoundaryBottom` (`22.5ms`, `1.0ms` onset) |
| Continuous (`0x1000`) | `dur >= 180ms`, `freq >= 60` | any | Rich resonant chime sweep | `171_todoAlldone` (`444ms`, `2.1ms` onset) |
| Continuous (`0x1000`) | `dur >= 180ms`, `35..59` | any | Firm sustained alert | `164_alert` (`170ms`, `2.1ms` onset) |
| Continuous (`0x1000`) | `dur >= 180ms`, `freq < 35` | any | Deep positive swell | `205_SuperPositive` (`436ms`, `0.4ms` onset) |
| Continuous (`0x1000`) | `70..179ms`, `freq >= 60` | any | Authoritative textured hold | `173_signalButtonHighlight` (`99ms`, `0.0ms` onset) |
| Continuous (`0x1000`) | `70..179ms`, `freq < 60` | any | Low textured drag | `207_Negative` (`98ms`, `0.7ms` onset) |
| Continuous (`0x1000`) | `dur < 70ms` | any | Short directional pull / unlock | `162_GestureBackPull` (`30.6ms`) / `210_FingerprintUnlock` (`26.4ms`) |

---

## 3. System UI Multi-Texture Mapping Guidelines (Cross-OEM Duration-Safe)

Never map a short framework `OplusPrebakedSegment` (e.g., `mDuration = 20ms` keyboard or `35–45ms` UI tap/switch) to a slow-onset RTP (e.g., Xiaomi `180_signalKeyHigh` has **`25.4ms` leading silence** before peak at `32.6ms`, causing `VibrationThread::off()` at `20ms` to produce **0% vibration**; `187_signalSwitch` peaks at `73.5ms` past ColorOS's `45ms` cutoff). Always choose PCM-verified instant-onset targets (`Onset <= 2.1ms`, `DurMs` fitting cleanly inside `mDuration`):

| System Interaction Category | Framework `mDuration` | Verified Instant-Onset RTP Mapping |
| :--- | :--- | :--- |
| Keyboard (weak / medium / strong) | `14–20 ms` | `178_signalClockUnit` (`12.2ms`), **`181_signalKeyUnit` (`13.0ms`, `0.0ms` onset)**, `4_pop` (`15.8ms`) |
| Standard UI Button Tap | `35 ms` | `166_feedbackNeutral` (`14.3ms`, `2.0ms` onset) |
| Toggle Switch / Checkbox | `45 ms` | `88_launcherIconSelection` (`15.2ms`, `98Hz` deep mechanical toggle) |
| Slider / Seekbar Detent | `15–30 ms` | `201_BoundaryBounce` (`13.3ms`), `84_uninstallDialog` (`7.6ms`) |
| Wheel / Overscroll Micro-Tick | `30 ms` | `178_signalClockUnit` (`12.2ms`), `188_signalTab` (`25.1ms`) |
| Boundary / Edge Collision | `30–45 ms` | `201_BoundaryBounce` (`13.3ms`), `202_BoundaryBottom` (`22.5ms`), `203_BoundaryTop` (`22.5ms`) |
| Long Press / 3D Touch | `80–350 ms` | `4_pop` (`15.8ms`), `410_longpress` (`21.3ms`), `173_signalButtonHighlight` (`99.2ms`) |
| Gesture Back Pull / Release | `30–110 ms` | `162_GestureBackPull` (`30.6ms`), `163_GestureBackRelease` (`23.3ms`) |
| Notifications | `400–6000 ms` | `67_NotificationXylophone`, `60_Fresh`, `205_SuperPositive`, `167_feedbackPositive`, `164_alert`, `208_Mute` |

---

## 4. Event-Driven Scheduler & Cancellation Hygiene

- Run a single detached worker thread waiting on `pthread_cond_wait(&cv, &mu)` initialized with `CLOCK_MONOTONIC` (`pthread_condattr_setclock`).
- Protect playback state with a monotonic `uint64_t generation` counter:
  - Starting a new pattern or receiving `IVibrator::off()` / `IOplusVibrator::stop()` increments `generation`, calls `pthread_cond_broadcast(&cv)`, and immediately stops the hardware actuator.
  - Between events, sleep via `pthread_cond_timedwait(&cv, &mu, &abs_target_ts)` and abort immediately if `generation != my_gen`.
- Debounce ultra-dense micro-events (`< 12ms` apart) so the LRA actuator cleanly completes each impulse without clipping.
- Fire `IRichtapCallback` (`code=1`, `FLAG_ONEWAY`, `int32(0)`) upon normal pattern completion.
