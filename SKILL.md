---
name: android-adaptive-haptics
description: Architect, debug, tune, and port high-fidelity adaptive multi-texture Android haptics on Qualcomm QTI qcom-hv-haptics LRA hardware, Xiaomi/Oplus/ColorOS/AOSP AIDL IVibrator HALs, and RichTap HE 1.0/2.0 engines. Use when fixing monotonous single-frequency vibration, broken haptic intensity sliders, PMIC qcom-hv-haptics rc=-16 EBUSY faults, rapid-typing haptic drops or latency stalls, dropped AIDL binder extensions (getExtension failed) in vibrator proxies, or implementing multi-texture tactile UX mapping and RichTap HE 2.0 synthesis.
---

# Android Adaptive Multi-Texture Haptics Engineering

Build crisp, expressive, zero-fault tactile feedback on modern Android X-axis Linear Resonant Actuators (LRAs) — especially Qualcomm `qcom-hv-haptics` PMIC platforms (SM8550/SM8650/SM8750) running stock, custom ROM, or cross-OEM ported AIDL `IVibrator` stacks.

## Core Design Principles (Android Haptics UX Foundation)

1. **Tactile Vocabulary Over Monotone Buzzing:** Never collapse distinct UI semantics onto a single generic `EFFECT_CLICK` (`id=0`). Users distinguish micro-interactions by envelope sharpness, resonant pitch, and decay:
   - **Keyboard Typing:** Ultra-short (`10–20ms`), crisp, light transient with fast active braking (e.g., dedicated keystroke RTP waveforms, never heavy or muddy thuds). Zero artificial inter-key cooldown (`slide_effect_protect_time = 0`) so 15–25 keys/sec typing never drops or stalls.
   - **Primary UI Tap vs. Mechanical Toggle Switch:** Separate light button taps from crisp two-phase mechanical switch detents.
   - **Sliders, Pickers & Scroll Detents:** Fine-grained micro-ticks (`tick` / `picker` / `seekbar` detents) with restrained amplitude so rapid dragging feels textured rather than noisy.
   - **Overscroll & Boundary Collisions:** Elastic spring/bounce waveforms distinct from clicks.
   - **Rich Media & O-Haptics (RichTap HE 1.0 / HE 2.0):** Parse every transient (`0x1001`) and continuous (`0x1000`) event's `(relativeTime, duration, intensity, frequency)` and map across varied hardware material textures (crystalline, metallic clank, organic bubble/pop, elastic spring, deep thud, textured damping) with per-event amplitude scaling.
2. **True Continuous Intensity Scaling:** The system `Haptic intensity` slider MUST smoothly scale physical motor drive from whisper-soft to full Vmax across a wide linear/perceptual dynamic range (e.g., `~9%–100%` of signed 16-bit `ff_effect.u.periodic.magnitude` `[3071..32767]`, rather than a cramped floor like `25%–100%`).
3. **Zero Polling Loops & Zero PMIC Faults:** Every HAL proxy or multi-event scheduler must be 100% event-driven (`pthread_cond_wait` / `pthread_cond_timedwait` on `CLOCK_MONOTONIC`, 0.0% idle CPU) and produce **zero** `qcom-hv-haptics` kernel `rc=-16` (`-EBUSY`) errors.

---

## Architecture & Critical Failure Modes (Proven Root Causes)

### 1. Qualcomm `qcom-hv-haptics` PMIC `rc=-16` (`-EBUSY`) & Broken Intensity Sliders

**Root Cause:**
On Qualcomm `qcom-hv-haptics` Linux `input-ff` drivers (`EVIOCSFF` + `EV_FF`), calling `FF_GAIN` (`setGain` via `IVibrator::setAmplitude`) **while an effect is already playing** (or 1–4 ms *after* `IVibrator::perform()`) collides with the PMIC FIFO/auto-res playback state machine and returns **`rc=-16` (`-EBUSY`)**, causing dropped vibrations, kernel log spam, or stuttering.
Conversely, blindly NOP-ing `setGain` eliminates `rc=-16` **but breaks the system Haptic Intensity slider**, locking every effect to a fixed amplitude.

**Correct Hardware-Safe Solution (Pre-Upload Magnitude Scaling):**
1. **Order of Operations in Bridge/Client HAL:** Always call `IVibrator::setAmplitude(float amp)` **strictly BEFORE** `IVibrator::perform(effectId, strength)` — never after playback starts.
2. **Pre-Upload Storage in Vendor HAL:** Inside `IVibrator::setAmplitude(float amp)`:
   - Validate `0.0f < amp <= 1.0f`, clamp floor `amp = max(amp, 0.05f)`.
   - Store `amp` in the HAL's pre-upload amplitude slot (the same float read by `perform()` when populating `ff_effect.u.periodic.magnitude` **prior to** `ioctl(fd, EVIOCSFF, &effect)`), then return `AStatus_newOk()` **without** emitting a live `FF_GAIN` ioctl.
   - Reset the pre-upload float back to `-1.0f` after `perform()` consumes it so subsequent unscaled calls fall back cleanly to `EffectStrength` (`LIGHT`/`MEDIUM`/`STRONG`).
3. **Dynamic Range Calibration:** Map normalized amplitude `s0` `[0.05..1.0]` to `ff_effect.u.periodic.magnitude = (int16_t)(s0 * 29696.0f + 3071.0f)` (`[4556..32767]` = `13.9%..100%` Vmax) instead of narrow OEM defaults like `[8191..32767]`.

### 2. Rapid Keyboard Typing Latency, Dropped Keystrokes & Cross-OEM RTP Onset Cutoff (0-Haptics Bug!)

**Root Cause A (Cooldown Stall):**
Many OEM `qcom-hv-haptics` / `vibratorfeature` HALs read a `slide_effect_protect_time` cooldown (often `30–60ms`) and call `usleep()` on the binder thread when consecutive `perform()` calls arrive faster than the cooldown window. During fast typing (10–20 keys/sec = `50–100ms` inter-key intervals, bursts at `30–40ms`), this stalls the vibration thread and drops keystrokes.

**Root Cause B (Cross-OEM RTP Leading-Silence / Duration Cutoff -> Silent Keystrokes!):**
When porting one OEM's framework (e.g., ColorOS `/my_product/etc/vibrator/effect_waveform.xml`) onto another OEM's hardware RTP library (e.g., Xiaomi `/odm/firmware/*_P_RTP.bin` at `24 kHz` = `24 bytes/ms`), the framework's `VibrationThread` sleeps strictly for its XML `mDuration` (e.g., **`20ms`** for keyboard `waveformId=111`, **`35ms`** for button `waveformId=1`, **`45ms`** for switch `waveformId=2`) and then immediately calls `IVibrator::off()`.
If a mapped target RTP waveform has **leading silence or a slow attack longer than `mDuration`** (for example, Xiaomi `180_signalKeyHigh_P_RTP.bin` is `147.1ms` long with **25.4ms of leading silence** and peak at `32.6ms`, and `187_signalSwitch_P_RTP.bin` peaks at `73.5ms`), `VibrationThread` cancels playback at `20ms` **before the actuator even receives energy**, resulting in **0% physical vibration** despite clean HAL logs!

**Fix:**
1. Force `slide_effect_protect_time = 0` at HAL initialization so back-to-back short transient waveforms upload and trigger immediately with zero artificial sleep.
2. **Audit every mapped target RTP binary's 24 kHz PCM envelope (`bytes / 24.0 ms`):** verify `onset_ms <= 2.1 ms`, `peak_ms < mDuration`, and high initial RMS (`R15 >= 40`). Map `20ms` keyboard strokes (`110/111/112`) and short UI taps (`1/2/7/13/14/105`) strictly to instant-onset, short-envelope hardware RTPs (e.g., `181_signalKeyUnit` `13.0ms / 0.0ms onset`, `178_signalClockUnit` `12.2ms / 0.0ms onset`, `4_pop` `15.8ms / 0.6ms onset`, `166_feedbackNeutral` `14.3ms`, `88_launcherIconSelection` `15.2ms`, `201_BoundaryBounce` `13.3ms`). Apply the same instant-onset audit to RichTap HE 1.0/2.0 event targets so rapid beats never clip silent.

### 3. AIDL Vibrator Proxy Pitfalls: Dropped Extensions & Service Hijacking

When stacking a vibrator proxy (e.g., for ringtone envelopes or cross-OEM translation) in front of a vendor `IVibrator/default` HAL:

1. **Dropped Extension (`E ...: getExtension failed`):**
   - `AIBinder_new()` creates a fresh local binder **without** copying the wrapped service's AIDL extension (such as `IOplusVibrator` or `IVibratorExt`).
   - **Fix:** Immediately after creating the wrapper binder (both for `IVibrator/default` and every binder returned by `IVibratorManager::getVibrator()`), call:
     ```cpp
     AIBinder* real_ext = nullptr;
     if (AIBinder_getExtension(real_vib, &real_ext) == STATUS_OK && real_ext) {
         AIBinder* ext_wrap = wrap_extension(real_ext);
         AIBinder_markVintfStability(ext_wrap);
         AIBinder_setExtension(wrapper_vib, ext_wrap);
         AIBinder_decStrong(ext_wrap);
     }
     ```
2. **Duplicate `IVibrator/default` Registration Race:**
   - On cross-OEM ports, a secondary vendor HAL (e.g., Xiaomi `vibratorfeature.service`) often registers **both** `IVibrator/vibratorfeature` **and** `IVibrator/default`. If it starts 100ms after your bridge/proxy, `servicemanager` silently overwrites `IVibrator/default` with the secondary HAL — breaking `getExtension()` because clients receive the wrong extension interface descriptor (`IVibratorExt` instead of `IOplusVibrator`).
   - **Fix:** Ensure the secondary hardware HAL registers **only** its named instance (`IVibrator/vibratorfeature`) and never overwrites `IVibrator/default`.
3. **Boot-Time `waitForService` SELinux AVC:**
   - Calling `AServiceManager_waitForService()` from a `hal_vibrator_default` proxy registers a callback binder that `servicemanager` may lack SELinux `binder { call }` permission to invoke.
   - **Fix:** Use a bounded boot-only `AServiceManager_checkService()` retry loop (e.g., 150ms interval until the target HAL is up, then exit the init loop permanently) — zero SELinux AVCs and zero runtime polling.

### 4. Multi-Texture Waveform Taxonomy & RichTap HE 1.0 / HE 2.0 Synthesis

See [references/richtap-he-and-rtp-mapping.md](references/richtap-he-and-rtp-mapping.md) for:
- Complete ColorOS/AOSP tactile interaction -> hardware RTP waveform mapping strategy (avoiding collapse onto `effect=0`).
- Binary layout and C++ parser design for AAC RichTap **HE 1.0** (`ver=1`, 17-int stride) and **HE 2.0** (`ver=2`, 8-int header + multi-pattern `0x1000` continuous / `0x1001` transient blocks).
- Event-driven `CLOCK_MONOTONIC` `pthread_cond_timedwait` worker with instant generation-counter cancellation on `off()` / preemption and `IRichtapCallback` completion signaling.

---

## Diagnostic & Verification Checklist

Always verify all five gates on live hardware after modifying a vibrator stack:

1. **Service Ownership & Extension Integrity:**
   - Confirm `dmesg` / `servicemanager` shows no duplicate `IVibrator/default` overwrite after boot.
   - Confirm `AIBinder_getExtension(IVibrator/default)` returns the expected vendor extension and `logcat` shows `0` `getExtension failed` errors.
   - Confirm `0` SELinux AVCs (`dmesg | grep avc:.*vibrat`).
2. **Zero PMIC Faults:**
   - Run rapid typing (20+ consecutive triggers at `35ms` cadence) and multi-event HE playback, then verify `dmesg | grep -iE 'rc=-16|haptics.*(err|fail)'` returns **0 matches**.
3. **True Stepless Intensity Scaling:**
   - Sweep system haptic intensity Low / Mid / High and verify `logcat` confirms `setAmplitude(amp)` executes **before** `perform(effect)` and the HAL consumes the exact scaled float (`0.40 -> 0.72 -> 1.00`) prior to `EVIOCSFF`.
4. **Multi-Texture Distinctness:**
   - Verify keyboard, UI button, toggle switch, slider/picker detent, overscroll boundary, and notifications dispatch **distinct** hardware RTP effect IDs (never collapsing to `effect=0`).
5. **RichTap HE / O-Haptics Demo Playback:**
   - Play an HE 2.0 showcase (e.g., Settings O-Haptics demo) and confirm multiple distinct transient (`0x1001`) and continuous (`0x1000`) events schedule accurately across varied hardware RTP waveforms with per-event amplitude scaling and `0.0%` idle CPU when finished.
