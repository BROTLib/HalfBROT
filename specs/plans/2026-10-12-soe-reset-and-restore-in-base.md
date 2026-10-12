# SoE reset and position restore once, in FB_AxisControl (#14)

**Status: implemented on `develop` (see the PR linked from #14), not run on a PLC or a telescope.** Built by CI
(TcBuild) only. Hardware check: separate issue (linked from #14).

> **Update 2026-10-12:** `_RestoreLastPosition` was removed again (#7, see
> [2026-10-12-remove-axis-position-restore.md](2026-10-12-remove-axis-position-restore.md)); `_SoEReset` no longer
> sets `fLastPosition`. The rest of this plan stands.

## Problem

`FB_AzimuthControl`, `FB_ElevationControl` and `FB_DerotatorControl` each carry their own copy of the SoE
reset, azimuth and elevation also of the position restore (`MC_SetPosition` with the persistent
`fLastPosition`). The SoE reset copies differ:

- **Azimuth** calls `fbSoEReset(Execute := TRUE)` and, while `Busy`, immediately `Execute := FALSE`. The next
  cycle is a new rising edge, so the reset is probably restarted every other cycle until `Busy` happens to be
  FALSE right after a call.
- **Elevation, derotator** finish on `NOT Busy`. If `Busy` is not yet TRUE right after the rising-edge call,
  they finish at once and no reset ran.
- **All three** ignore `Error`: a failed reset is followed by `bReset := TRUE` as if it had worked, and nothing
  is logged.

`FB_SoEReset` (Tc2_MC2_Drive) has no `Done`. Its outputs are `Busy`, `Error`, `AdsErrId`, `SercosErrId`;
`Timeout` defaults to 10 s (read from the type info in `MONETSRuntime.tmc`). Success therefore means: `Busy`
was seen TRUE and dropped, and `Error` is FALSE.

Who triggers it: no consumer (MONETcommon, MONETS, MONETN) writes `bSoEReset`. The only writer is the
derotator's own diagnostic branch (`bSoeReset := bReset`), which never runs because of #2. So the SoE reset is
reachable today only through ADS or a visualisation writing `bSoEReset`.

## Change

`FB_AxisControl` gets two methods; the three axis blocks call them instead of their copies.

### `_SoEReset(sAxis)` (every cycle)

A step sequence with its state in `FB_AxisControl`'s `VAR` (method `VAR` is re-created per call, MONETcommon
#17/#59):

| Step | Action | Next |
|---|---|---|
| 0 idle | on `bSoEReset`: `fLastPosition := -1.0` (forces a re-home), clear the error flag, call `fbSoEReset` with `Execute := FALSE` (guarantees the edge) | 1 |
| 1 started | `Execute := TRUE` | `Error` → failed; `Busy` → 2 |
| 2 running | `Execute := TRUE` | `Error` → failed; `NOT Busy` → done |
| done | `Execute := FALSE`, `bSoEReset := FALSE`, `bReset := TRUE` (axis reset, as before) | 0 |
| failed | `Execute := FALSE`, `bSoEReset := FALSE`, **no** axis reset, error event with `AdsErrId`/`SercosErrId` | 0 |

A 15 s watchdog over steps 1 and 2 (longer than `DEFAULT_ADS_TIMEOUT` passed as `Timeout`) counts as failed,
so the sequence cannot hang if `Busy` never rises.

Behaviour changes:

- Azimuth: one reset per request instead of repeated restarts.
- Elevation/derotator: a reset that did not start is no longer reported as done.
- All: a failed reset logs `'<axis>: SoE reset failed (ADS 16#..., SERCOS 16#...)'` and does not reset the axis.
- Derotator: `fLastPosition := -1.0` now also happens there. It has no position restore, so this has no effect.

### `_RestoreLastPosition()` (every cycle)

The identical azimuth/elevation code moved as is: `MC_SetPosition` with `fLastPosition` when
`bGetLastPosition AND fLastPosition >= 0.0 AND NOT bCalibrated`, clear `bGetLastPosition` on `Done`, track
`fLastPosition := fActualPosition` while calibrated. Called at the same place in each block as before. The
open questions of #7 (does the restore count as calibrated, the `-1` sentinel) are unchanged.

## Not changed

- The diagnostic check (`inDiagnostic < 16#D012 AND inDiagnostic > 16#D014`, never TRUE) stays in the three
  blocks: that is #2, a separate fix.
- Torque monitoring: thresholds and timers differ per axis on purpose (50 %/3 s, 50 %/1 s, 20 %/3 s).
- The `bSoEReset` input stays (MONETcommon, ADS, visualisations); turning it into a method is step 1 of the
  #31 plan.
- IAG50cm has the same `NOT Busy` copy in `FB_DeclinationControl` and `FB_FocusControl`. `fbSoEReset` lives in
  BROTLib's `FB_BaseAxis`, so the sequence could move there later and serve both; filed as a follow-up.

## Checks

- CI: TcBuild of HalfBROT on the PR branch.
- XAE: MONETS and MONETN build against the next HalfBROT release (Tim).
- Hardware (MONETS or MONETN, per axis): write `bSoEReset := TRUE` over ADS with the drive in a diagnostic
  error or idle; the drive resets once (no repeated resets in the drive's diagnosis history), `bSoEReset` drops,
  the axis is reset and needs homing (`fLastPosition = -1`). With the drive unreachable or the timeout forced,
  the error event appears and the axis is not reset.
