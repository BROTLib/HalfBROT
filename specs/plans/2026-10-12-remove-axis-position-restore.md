# Remove the position restore from the azimuth and elevation axes (#7)

**Status: implemented on `develop` (see the PR linked from #7), not run on a PLC or a telescope.** Built by CI
(TcBuild) only. Facts below were read from `develop` @ `018ddae` and the consumers' `develop` on 2026-10-12.

Decided 2026-10-12 (Tim): remove the restore rather than make it work. Azimuth and elevation keep homing on their
cam at every start.

## Problem

`FB_AxisControl._RestoreLastPosition()` (moved there in #14, called by `FB_AzimuthControl` and
`FB_ElevationControl`) tries to restore the last known position after a restart:

- `fLastPosition` (`VAR PERSISTENT`, default `-1.0`) is written with `fActualPosition` while `bCalibrated`.
- At start-up `fbAxisCalibration : MC_SetPosition` runs once with `Execute := bGetLastPosition AND
  fLastPosition >= 0.0 AND NOT bCalibrated`; `Done` clears `bGetLastPosition`.

It most likely has no effect:

- BROTLib `FB_Axis` derives `Calibrated` from the NC flag `Axis_Status.Status.Homed`. The Tc2_MC2 docs describe
  `MC_SetPosition` only as shifting the actual position without motion; only `MC_Home` (`MC_Direct`,
  `MC_ForceCalibration`, cam homing) is documented to set the calibration state. So the axis stays not calibrated
  after the restore (not tested on hardware).
- MONETcommon `FB_MonetTelescopeControl._PowerOn` homes every axis that is `Ready AND NOT Calibrated`. The cam
  homing then overwrites the restored position.
- The `-1.0` sentinel with `>= 0.0` can never restore a negative position, though the azimuth ranges are
  -274..270 (MONETS) and -72..495 (MONETN).

A restore that did work would be worse: with the incremental encoders, a mount moved while switched off
(maintenance, manual move) would trust a wrong position. The focus is different: it restores on purpose with
`MC_Direct` (#18) and is not touched here.

## Change (one PR to `develop`)

`FB_AxisControl`:

- Remove the method `_RestoreLastPosition` and the variables `fbAxisCalibration : MC_SetPosition`,
  `bGetLastPosition`, and `VAR PERSISTENT fLastPosition` (the whole `VAR PERSISTENT` section, it holds nothing
  else).
- `_SoEReset`, step 0: drop `fLastPosition := -1.0` (nothing to invalidate any more; a drive reset still ends in
  `bReset := TRUE`, and the axis re-homes because it is not calibrated).

`FB_AzimuthControl`: remove `_RestoreLastPosition();` and `bGetLastPosition := FALSE;` (in
`IF bCalibrated OR bError`). `FB_ElevationControl`: remove `_RestoreLastPosition();` and its comment.

Not touched: `FB_DerotatorControl` (never restored, absolute encoder, #17), `FB_FocusControl` (own persistent
`fLastPosition`/`fLastPposition`, extends `FB_BaseAxis`, not `FB_AxisControl`).

Docs: README "Calibration restore" bullet replaced by one sentence (azimuth and elevation home on their cam at
every start, no restore; the focus restores with `MC_Direct`); the #14 plan gets a note that
`_RestoreLastPosition` was removed again; plans index.

Consumers: nothing in MONETcommon, MONETS or MONETN reads `fLastPosition`, `bGetLastPosition` or
`fbAxisCalibration` (POUs, GVLs, visualisations and `.tsproj` checked), so no change there.

Release: part of the next HalfBROT minor (already breaking: #63, #64).

## Risk: persistent data

Removing a `PERSISTENT` variable changes the persistent data of `MAIN.AzimuthControl`, `MAIN.ElevationControl`
and `MAIN.DerotatorControl`. If TwinCAT does not restore the remaining persistent data by name after the download,
the focus's `fLastPosition`/`fLastPposition` would also be lost: one cam homing of the focus instead of
`MC_Direct`, harmless. Check on the first download (below).

## Checks

- CI: TcBuild of HalfBROT.
- XAE (Tim): MONETS and MONETN build against the new HalfBROT release (no references to the removed variables
  expected).
- On the telescope, first start after the update (normal start-up, remote): azimuth and elevation home on their
  cam and become `Calibrated` as before; the focus restores its position without moving (#59; if it does a cam
  homing instead, the persistent data were reset, see Risk). No separate hardware-test issue: this is what every
  start does today; it goes into #59 as one more line.
