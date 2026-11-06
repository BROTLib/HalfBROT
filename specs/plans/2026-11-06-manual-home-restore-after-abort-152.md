# Manual focus re-home after an aborted one restores the first focus (#152)

**Status: implemented on `fix/manual-home-restore-after-abort-152` (from `develop` @ `78a0492`, 0.12.0), not compiled, not run on a PLC.**
Tests: MONETcommon branch `test/manual-home-restore-after-abort-152` (`FB_FocusManualHome_Tests`), not run; they need the HalfBROT
release that carries this change pinned in MONETcommonTests.

## Problem
MS-135 (IAG results file, reconstructed from the issue): focus calibrated at 35.98 mm, `bManualHome`, `stop=1` 60 s into the sweep,
`bManualHome` again to the end. The second re-home ended on the cam (91.5 mm) and did not go back to 35.98 mm; T17 step 1 expects it to.

## Cause (read on `develop`, 2026-11-06)
Not a junk `fTarget` (the issue read v0.7.2). `FB_FocusManualHome.Request` sets `bRestoreTarget := bCalibrated` and
`fRestoreTarget := SEL(bCalibrated, 0.0, fTarget)` (#138/#115: an uncalibrated axis has no target). The stopped sweep leaves the focus
uncalibrated, so the second request overwrote the 35.98 saved by the first with "nothing to restore". By design for a focus that was
uncalibrated to begin with; wrong for one made uncalibrated by the first re-home.

## Change (`FB_FocusManualHome`)
- `Request` on an uncalibrated axis keeps `bRestoreTarget`/`fRestoreTarget` when they are pending, else clears the target (as before).
  On a calibrated axis it takes the fresh target (as before).
- Pending ends: in the cycle after `bDone` (`FB_FocusControl` reads it in the cycle of `bDone`); at an abort that leaves the axis
  calibrated; at `ClearBlock` (the automatic homing, which restores nothing).
- Unchanged: a stop leaves the focus where it is, locked, uncalibrated, automatic homing blocked. `FB_FocusControl` is not touched.
- Decision (Tim, 2026-11-06): an interrupted re-home is not a cancelled one; the focus the observer had before it is still the target.

## Tests (MONETcommonTests `FB_FocusManualHome_Tests`, 31 of 32 tests)
`Request_Clears_Block` flipped (kept 36.0 instead of none); new `Aborted_Rehome_Then_Rehome_Restores_The_First_Target` (the MS-135
sequence, used up after `bDone`), `ClearBlock_Drops_The_Kept_Target`, `Calibrated_Abort_Drops_The_Pending_Target`.

## On site
T17 step 1 again on a build with this change: re-home from a calibrated focus, `stop=1` in the sweep, `bManualHome` again to the end:
`fbManualHome.fRestoreTarget` is the old focus after the second request and the focus ends there. Also a re-home on a focus that was
uncalibrated from the start (cold start) still ends on the cam.

## Unverified
Build; the tests; whether 0.11.1 (the MS-135 build) has the same `Request` as `develop`; the log lines of MS-135.
