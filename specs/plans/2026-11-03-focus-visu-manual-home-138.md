# Focus homing from the Visu: request `bManualHome`, accept an uncalibrated focus (#138)

**Status: implemented in HalfBROT 0.11.0 (`feat/focus-visu-manual-home-138`), not compiled or run on a PLC.** Facts read on 2026-11-03
from HalfBROT `develop` (0.10.2 plus #136), MONETcommon `5ffc0aa`, MONETS `origin/develop` `d170915`.

## Problem
The Visu focus Start writes `HomeAxis := TRUE`. On a calibrated focus with a stored position that is a no-op (`MC_Direct` sets the
position, no motion). `bManualHome` works (cam homing), but it rejected an uncalibrated focus (`NotCalibrated`) and the Visu had no
way to request it or to show a refusal.

## Decisions (defaults of the plan, taken 2026-11-03)
1. **No restore for an uncalibrated request.** `fPosition` is junk there (0.5 after a PLC start, #115; 91.5 after an aborted homing),
   and HalfBROT has no park position. The focus stays where the cam homing ends, like the automatic power-on homing; the caller sets
   the target afterwards. New output `FB_FocusManualHome.bRestoreTarget` (TRUE if the request came on a calibrated focus).
2. **Always a cam homing** (`fLastPosition := -1` on accept), also with a valid stored position of an uncalibrated focus.
3. **`eManualHome` is a `VAR_OUTPUT`** of `FB_FocusControl` so the Visu can show the refusal reason.
4. A request on a blocked (aborted, uncalibrated) focus drops the old target (`bRestoreTarget := bCalibrated`).
5. `E_FocusManualHome.NotCalibrated` stays as an unused value (existing references, `qualified_only`).
6. HalfBROT 0.11.0 (minor: behaviour of an input changed, new output). MONETN gets the same one-line `MAIN` change in its own PR.

## Changes
- `FB_FocusManualHome.Request`: the `NotCalibrated` rejection is gone; order is `NotAutomatic`, `ManualUnlock`, `AxisError`, `Homing`,
  `Moving`, `Accepted`. Manual mode stays refused (`FB_FocusDriveEnable` only enables in automatic).
- `FB_FocusControl`: restore at `bDone` only if `bRestoreTarget`; `eManualHome` is an output.
- README "Manual focus re-home", version 0.11.0.

## Interplay with the automatic homing / `bBlockAutoHome`
- Automatic homing fires first in the same cycle (drive enabled): the request gets `Homing`.
- Drive not enabled: the request is the only homing; `HomeAxis := TRUE` sets `HomePending`, `bHomeWanted` enables the drive.
- Aborted uncalibrated request: sets the block as before; the next request is accepted and lifts it.

## Tests / on site
Unit tests in MONETcommonTests (uncalibrated accepted, manual mode, error, unlock, moving and doubled start refused, done without
restore, abort cases). On site: calibrated Start, uncalibrated Start with drive off (ends at about 91.5 and locks after 30 s), manual
mode refused and shown, Start twice, Stop during an uncalibrated homing then Start.
MONETS: `MAIN` sets `FocusControl.bManualHome := TRUE` for `bVisuHomeFocus`, Visu text for `eManualHome`, pin 0.11.0.
