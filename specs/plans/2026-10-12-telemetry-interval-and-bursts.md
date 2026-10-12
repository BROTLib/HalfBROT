# Telemetry: hydraulics bursts and lost updates, interval input (#19)

**Status: implemented on `develop` (see the PR linked from #19), not run on a PLC or a telescope.** Built by CI
(TcBuild) only. No hardware test issue: topics and payloads are unchanged, only timing.

## Problem

MONETS/MONETN pass `FB_Comm_MQTT_Influx` (BROTLib). Its `Publish` sends directly with `bQueue := FALSE`, on
purpose (BROTLib #5): periodic telemetry is not retried; if the client is busy the message is dropped and
`deliveryFailed` counts it.

- **Hydraulics** publishes its 14 values in one cycle, **only when `statusWord` changes**:
  - a dropped message is never repeated, so a lost `BrakeOpen`/`OilPressureOK` stays stale downstream until the
    next change;
  - bit 15 is set for one cycle when the oil level moved by more than 5 and cleared in the next, so every level
    change gives two bursts (28 messages) in consecutive cycles; contact bounce gives a burst per edge.
- **Covers, focus:** `tonComm : TON := (PT := fTelemetryInterval)` is evaluated once at initialisation, so
  setting `fTelemetryInterval` in the call has no effect. Nobody sets it today (default 5 s everywhere).
  MONETcommon's cover copy already fixed the same thing (M3).
- Comments say "every second"; the interval is 5 s.
- Focus `POWER_STATE` is always `'1.0'`: intended (the drive is switched off at rest to engage the brake, which
  is not "off" for pyobs), but the comment does not say why.

## Change

### Hydraulics

New input `fTelemetryInterval : TIME := T#5S` (same name and default as covers/focus).

- A publish is **requested** when `statusWord` changed, when the oil level moved by more than 5 since the last
  publish, or when `fTelemetryInterval` has elapsed since the last publish (periodic, so a dropped message
  recovers within 5 s).
- A requested publish is **sent** at most once per second (`tonPublishHoldoff`, 1 s since the last publish);
  changes in between are collected and sent with the current values.
- Bit 15 of `statusWord` (the one-cycle level-change pulse) is gone; the oil level is compared directly. The
  published `StatusWord` therefore never has bit 15 set any more.

Load: at most 14 messages per second, normally 14 per 5 s.

### Covers, focus

`tonComm(IN := TRUE, PT := fTelemetryInterval)`, so the input works. Comments corrected; focus `POWER_STATE`
comment says why it is constant.

## Checks

- CI: TcBuild of HalfBROT.
- Optional, remote over ADS on MONETS/MONETN after the release: `MAIN.fbComm.deliveryFailed` before and after,
  and the hydraulics topics in Influx arriving every 5 s.
