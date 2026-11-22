# Android Adaptive Haptics Skill

Agent skill and engineering reference for building, debugging, and porting high-fidelity adaptive multi-texture Android haptics on Qualcomm `qcom-hv-haptics` LRA hardware, AIDL `IVibrator` HALs, and RichTap HE 1.0/2.0 engines.

## What This Skill Covers
- **Eliminating PMIC `qcom-hv-haptics` `rc=-16` (`-EBUSY`) Faults** while preserving true continuous stepless `Haptic intensity` slider scaling via pre-upload `ff_effect.u.periodic.magnitude` calibration (`setAmplitude` before `perform`).
- **Zero-Latency Rapid Typing:** Removing `slide_effect_protect_time` stalls so fast 15–25 keys/sec typing never drops or stutters.
- **Multi-Texture Tactile Vocabulary:** Mapping keyboard, UI taps, toggle switches, slider/picker detents, overscroll boundaries, and notifications to distinct hardware RTP waveforms rather than collapsing onto a single monotone click.
- **RichTap HE 1.0 & HE 2.0 Adaptive Synthesis:** Parsing multi-pattern HE `int32[]` payloads and driving event-driven (`CLOCK_MONOTONIC` `pthread_cond_timedwait`, 0.0% idle CPU) multi-texture hardware playback.
- **AIDL Vibrator Proxy Hygiene:** Forwarding `AIBinder_getExtension` / `AIBinder_setExtension`, preventing duplicate `IVibrator/default` service hijacking, and avoiding boot-time `servicemanager` SELinux AVCs.

## Contents
- `SKILL.md` — Primary Antigravity agent skill definition & diagnostic checklist.
- `references/richtap-he-and-rtp-mapping.md` — RichTap HE 1.0/2.0 wire format, frequency/intensity texture matrix, and event-driven scheduler patterns.
