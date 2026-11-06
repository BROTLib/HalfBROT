# A failed homing must not leave the axis at the virtual -1e10 position (#147)

**Status: proposed (open questions answered in section 7), implemented in the PR, not compiled or run on a PLC.** Facts read on 2026-11-06 from HalfBROT `origin/develop` @
`a1287d7` (0.12.0, contains #136 and #148), BROTLib `origin/develop` @ `84c53c0` (0.18.0), MONETcommon `origin/develop` @ `f9c7d5d`
(0.16.0), MONETS `origin/develop` @ `e7247a4`. Line numbers are at those commits. Suggested branch `fix/failed-homing-position-147`
from `origin/develop`.

Issue: HalfBROT#147, split out of #136 (plan `2026-11-03-home-calibrated-axis-136.md`, section 7 item 8). MONET/S, 2026-11-03, MS-95
step (8) in the IAG test plan results: after the failed Visu re-home of the elevation (`0x4B0A`, emergency park) `fActualPosition`
was about -9 999 999 947, `bCalibrated` FALSE, NC software limits active; a manual jog down was refused with `0x4460` ("target
position -9999999947.6 is smaller than the active software limit switch MIN (0.0)"). Only XAE **Set Actual Position** (to an
estimated 90) made the axis jog again. The telescope had been left near zenith. Open question of the issue: restore a sane position
after a failed or aborted homing, or keep the axis visibly uncalibrated in a way that does not block jogging?
Related: #148 (latched jog, fixed in 0.12.0), #114 (stop during a focus homing, `HomeStopAbort`), #7 (the old `MC_SetPosition`
restore at start-up, removed), IAG report `docs/monetn-2026-10-27-power-on-and-drives.md` section 3 (NC homing trace).

## 1. Facts and hypotheses

### 1.1 Established

| # | Fact | Source |
|---|---|---|
| F1 | The NC, not our code, sets the virtual position: at the start of a cam homing it sets the actual position to **-1e10** and searches in the positive direction; at the cam the position flips to **+1e10** and the axis syncs in the negative direction; at the reference mark it jumps to the real value. | MONET/N recording 2026-10-27 (IAG `docs/monetn-2026-10-27-power-on-and-drives.md:69`, elevation and azimuth); focus traces MS-57, MS-85, MS-100 (`ActPos -9999999999.698` 2 s into a homing) |
| F2 | The MS-95 value is the search phase: -9 999 999 947 = -1e10 + 53, i.e. 53 degrees of positive travel since the homing started. It matches the reported 53 degrees upward in 54 s (`RefSearch` 1 deg/s, MONETS `.tsproj:2323`). The cam flip never happened. | issue #136 comments, MS-95 step (7)/(8) |
| F3 | Nothing in BROTLib or HalfBROT writes the position: no `MC_SetPosition` in either repo (the last one, `_RestoreLastPosition`, was removed in #7). `FB_Axis` passes the NC value through: `ActualPosition := AxisRef.NcToPlc.ActPos` (`FB_Axis.TcPOU:320-324`). `Home(fReference)` only stores `HomeReference` for `MC_Home.Position` (`:438`, `:277`), which the NC applies at the reference mark. | code |
| F4 | What `FB_Axis` does when `MC_Home` ends badly: on `Axis_Home.Error` the error goes to `Error`/`ErrorID` (`:348-350`, `:376`) and `HomeCommand.Execute` **stays TRUE** until the next request or `Reset` (`FB_AxisCommand.TcPOU` header: "On an error Execute stays TRUE"), so `HomeActive` stays TRUE until `Reset` (`:281`). A `Stop()` during the homing sets `HomeStopped`; the resulting `0x4B0A` is turned into an abort (`HomeStopAbort`, `:285`), not an error (#114). `CancelHome()` only drops a request that has not started (`:483-485`). None of these paths touch the position. | code |
| F5 | HalfBROT already detects the failure: `FB_HomeRequest` (`_HomeRequest`, `FB_AxisControl:90-127`) releases `bHomeAxis` and pulses `bFailed` on an error, an abort, an uncalibrated end or a start timeout; `homeFailedEvent` logs a WARN. With the release the homing overrides (`MC_SetAcceptBlockedDriveSignal`, `Enable_Positive OR bHomeAxis`) switch off. | code |
| F6 | `MC_Reset` does not restore the position. After the first MS-95 failure, `reset=1` + `power=1` recovered without XAE because the power-on homes every `Ready AND NOT Calibrated` axis (MONETcommon `FB_MonetTelescopeControl:1439-1445`) and the new homing starts from -1e10 anyway (24 s). **Only a manual jog is blocked**, not the automatic recovery. | MS-95 steps (7), (8) |
| F7 | NC software limits (MONETS `develop`): elevation MIN enabled with the default 0, MAX enabled 91 (`.tsproj:2328-2329`; the issue text says MAX 90, the online value was not read); azimuth -274 / 270 enabled (`:2181-2182`); derotator -90 / 360 (`:2474-2475`, homes with `MC_ForceCalibration`, no motion, no virtual position); focus MAX 100 **not enabled**, MIN not set (`:2618`), so the focus is not blocked by this issue. | `.tsproj` |
| F8 | The elevation cam is the positive limit input: `bCalibrationCam := NOT inDigitalInputs.0` and `Enable_Positive := inDigitalInputs.0 OR bHomeAxis` (`FB_ElevationControl:112`, `:116`); same pattern on the azimuth with `inDigitalInputs.1` (`FB_AzimuthControl:62`, `:66`). During a homing the positive limit is overridden, so the search is stopped only by the cam (or the NC's `0x4B0A`). MONETS `fCalibPosition` of the elevation is 60.57 deg (`MAIN.TcPOU:197`). | code |
| F9 | Since #148 (0.12.0) a jog request is cleared while `bError` (the `0x4460` jog error included), so the latched `bMoveNeg` of MS-95 no longer remains; the jog itself is still refused by the NC. | `FB_AxisControl._JogRequest:73-86` |
| F10 | XAE "Set Actual Position" worked on a reset, disabled elevation (09:54:43, `bEnable` at 09:54:50) and did not make it calibrated (the following power-on homed it, 56 s). | MS-95 step (8) |
| F11 | The Visu (MONETS `Elevation.TcVIS`) shows `fActualPosition` (`:2098`) and `bCalibrated` (`:482`), with no Set Position element. Telemetry publishes `fElevationCurrent` and `bElevationCalibrated` (MONETcommon `:2286`, `:1692`). | Visu, code |

### 1.2 Hypotheses (to verify, see section 5)

| # | Hypothesis | Why it matters | How to check |
|---|---|---|---|
| H1 | The NC **sets** (not adds) exactly -1e10 at the start of the search and +1e10 at the cam, so the travel since the homing start is the sum of the per-cycle position changes with the two jumps left out. | The restore estimate (option A) relies on it. | Hardware step 1: compare with the raw encoder count `In.nDataIn1` (scale 360/1048576), which no NC offset changes. |
| H2 | The NC never restores the pre-homing position on an `MC_Home` error or abort (Tc2_MC2 says nothing; seen in MS-95, MS-85, MS-100). | If some NC option restored it, no PLC code would be needed. | Beckhoff docs / support; hardware step 1 without the fix. |
| H3 | `MC_SetPosition` (absolute mode) works on a reset, disabled, stationary axis and is refused while the axis has an error or a command running; it does not set `Homed`. | Decides when the restore can run and that `Calibrated` stays FALSE. | F10 supports it for XAE; hardware steps 1, 3 for the PLC block. |
| H4 | The NC refuses a jog whose start position is outside the window in **either** direction (`0x4460` MIN; presumably the MAX counterpart above MAX), even toward the window. | An exact estimate of 93 deg (above MAX 91) would be refused like -1e10. Hence the clamp in option A. | Hardware step 4 (only if cheap and safe; the clamp makes it irrelevant). |
| H5 | Why the MS-95 search ran 53 deg past the 60.57 deg reference without seeing the cam is not known (#136 explained the `0x4B0A` by the missing overrides, not the missed cam). | If the cam can be missed, an upward search that starts high runs to the end of travel with the positive limit overridden: the risk named in the issue. | Out of scope; Q5. Evidence for the normal case: after the first failure the power-on homing from near zenith took 24 s, so a start inside the cam region worked. |

## 2. Options

| Option | What | Verdict |
|---|---|---|
| **A. Restore an estimated position after a failed homing** (PLC, automatic) | While a homing runs, integrate the position change per cycle (skip jumps larger than one cycle can move, i.e. the +/-1e10 flips and the reference jump). On `fbHomeRequest.bFailed` with the position still virtual (`ABS(ActPos) > 1e9`), latch a restore; once the axis has no error, no homing and no motion, `MC_SetPosition` to `fStart + fTravel`, clamped into the NC soft-limit window with a margin. The axis stays uncalibrated; WARN with the estimate and whether it was clamped. | **Recommended.** Makes the axis jog by hand with a position close to the truth (the encoder is continuous, only the NC offset changes), no operator step, no XAE. The estimate is exact up to the encoder (H1) when the homing started calibrated (MS-95 case). |
| B. Keep uncalibrated, offer a Visu "Set Position" step | New HalfBROT method `SetPosition(fPosition)` (only when not calibrated, no homing, no motion) wrapping `MC_SetPosition`; MONETS/MONETN Visu input field + button on the axis pages. | **Recommended as a follow-up** on top of A (same `MC_SetPosition` plumbing): lets the operator correct a bad estimate (failed power-on homing after a PLC start, see Q3). Alone it needs someone at the Visu and a guess like XAE today. |
| C. Relax the NC software limits while uncalibrated | `MC_WriteBoolParameter` `AxisEnMinSoftPosLimit`/`AxisEnMaxSoftPosLimit` FALSE while `NOT Calibrated`, TRUE again at calibration. | Not recommended: changes NC parameters at runtime, a missed re-enable leaves a calibrated axis without soft limits (tracking, slews), and the jog still runs on a meaningless -1e10 display. Hardware limit switches would be the only protection. Keep as a fallback if `MC_SetPosition` turns out to be refused (H3). |
| D. Guard the next homing start near the end of travel | Before a cam homing that starts with a restored estimate above the cam (`fStart >= fCalibPosition`) or within a few degrees of the positive end, refuse it with a WARN (or jog down first). | Not now: the power-on homing from near zenith worked (F6, 24 s), and the new homing does not use the restored value. Worth doing only if Q5 shows the cam can be missed; then it belongs in HalfBROT next to the covers guard (`FB_ElevationControl:61-76`). Cheap part now: log the start estimate in the homing-start WARN of A. |
| E. Restore in BROTLib `FB_Axis` | Same as A inside `FB_Axis`. | Not for #147: BROTLib release chain, IAG50cm has its own homing (`FB_LatchHome` + `MC_SetPosition`) and must be re-tested. A is pure logic and can move to BROTLib later (Q2). |
| F. Do nothing, document the recovery | Recovery is `reset=1`, `power=1` (F6); manual jogs need XAE. | Rejected: the recovery homes upward from wherever the failure left the telescope, and a manual move away (the safe reaction near zenith) is impossible without XAE. |

## 3. Recommendation and design (option A, B prepared)

### 3.1 New pure-logic `FB_HomeFailPosition` (HalfBROT `POUs/`)

No NC access, testable like `FB_HomeRequest`. Sketch:

```
// Position after a failed homing (#147). The NC sets the actual position to -1e10 when a cam homing starts and to +1e10 at
// the cam; if the homing fails, the axis stays there and the NC software limits refuse every jog (0x4460). This block keeps
// the last real position before the homing and the travel since then, and asks for a restore to fStart + fTravel (clamped
// into the soft-limit window) once the failed homing has ended and the axis can take MC_SetPosition.
// Call once per cycle after fbAxis(...) and _HomeRequest; on bRestore the caller runs MC_SetPosition(fRestorePosition).
FUNCTION_BLOCK FB_HomeFailPosition
VAR_INPUT
	fActualPosition	: LREAL;	// fbAxis.ActualPosition
	bHomeActive		: BOOL;		// fbAxis.HomeActive
	bHomePending	: BOOL;		// fbAxis.HomePending
	bCalibrated		: BOOL;		// fbAxis.Calibrated
	bFailed			: BOOL;		// fbHomeRequest.bFailed (one cycle)
	bCanSet			: BOOL;		// NOT fbAxis.Error AND NOT fbAxis.Busy AND NOT fbAxis.InMotion
	fMin, fMax		: LREAL;	// NC soft-limit window
	fMargin			: LREAL;	// keep the restored value this far inside the window (e.g. 0.5)
	fMaxStep		: LREAL;	// largest real change in one cycle (e.g. 1.0 deg); larger changes are NC jumps
	bSetDone		: BOOL;		// MC_SetPosition.Done
	bSetError		: BOOL;		// MC_SetPosition.Error
END_VAR
VAR_OUTPUT
	bRestore			: BOOL;		// level: run MC_SetPosition now (rising edge = Execute)
	fRestorePosition	: LREAL;
	fEstimate			: LREAL;	// unclamped fStart + fTravel, for the log
	bClamped			: BOOL;
	bStartCalibrated	: BOOL;		// the homing started on a calibrated axis: the estimate is good to the encoder
	bRestored, bRestoreFailed	: BOOL;	// one cycle each
END_VAR
VAR
	fStart, fTravel, fLast	: LREAL;
	bTracking, bPending		: BOOL;
END_VAR
```

Logic (in words; exact code in the PR):
1. While idle (not homing, `ABS(fActualPosition) < 1e9`): `fStart := fActualPosition`, `bStartCalibrated := bCalibrated`.
2. Rising `bHomeActive OR bHomePending`: `fTravel := 0`, `fLast := fActualPosition`, tracking on. Each cycle while tracking:
   `d := fActualPosition - fLast; IF ABS(d) <= fMaxStep THEN fTravel := fTravel + d; END_IF; fLast := fActualPosition`.
3. `bFailed AND NOT bCalibrated AND bStartCalibrated AND ABS(fActualPosition) > 1e9` (Q3: no restore when the homing started uncalibrated) -> `bPending := TRUE`, `fEstimate := fStart + fTravel`,
   `fRestorePosition := LIMIT(fMin + fMargin, fEstimate, fMax - fMargin)`, `bClamped := fRestorePosition <> fEstimate`.
   A failure without a virtual position (start timeout, covers refusal, error before motion) needs no restore.
4. Tracking continues until the restore (an error keeps `HomeActive` until `Reset`, F4, and the axis may still coast).
5. `bRestore := bPending AND bCanSet AND NOT bHomeActive AND NOT bHomePending`; `bSetDone` -> `bRestored`, clear; `bSetError` ->
   `bRestoreFailed`, clear (no retry loop; the operator path is B or XAE). A new homing request clears `bPending`.

`fMaxStep` 1.0 deg: the fastest real change is the azimuth at 10 deg/s (`Maximum`, `.tsproj:2176`), 0.01 deg per 1 ms cycle; the
smallest NC jump is the reference jump from about +1e10. LREAL resolution at 1e10 is about 2e-6 deg, so the sum is exact enough.

### 3.2 Wiring

`FB_AxisControl`: `fbHomeFailPosition : FB_HomeFailPosition; fbSetPosition : MC_SetPosition; restoreEvent : FB_EventLog;` and a
helper `_HomeFailPosition(sAxis)` called right after `_HomeRequest(...)` in `FB_AzimuthControl` (`:79`) and `FB_ElevationControl`
(`:126`). Soft limits: read once per PLC start with `MC_ReadParameter` (`AxisMinSoftPosLimit`, `AxisMaxSoftPosLimit`,
`AxisEnMinSoftPosLimit`, `AxisEnMaxSoftPosLimit`), so the NC configuration is the only source (F7 shows the config and the issue text
already disagree on 90/91); a disabled limit means no clamp on that side. The existing `fbReadParameter` (`FB_ElevationControl:146`)
is cyclic for `AxisRemainingTimeToGo` and stays as it is; use separate instances.

`fbSetPosition(Axis := axisRef, Execute := fbHomeFailPosition.bRestore, Position := fbHomeFailPosition.fRestorePosition,
Mode := FALSE)` (absolute). WARN on `bRestored`: "<axis>: homing failed, position restored to <x> (estimate <e>[, clamped]
[, start not calibrated]); axis not calibrated, jog only"; ERROR on `bRestoreFailed` with the `MC_SetPosition` error id.

Not wired: derotator (`MC_ForceCalibration`, no virtual position), focus (`FB_BaseAxis`, no soft limits on MONETS, F7; MONETN to
check, Q6).

### 3.3 Option B prepared

Public method on `FB_AxisControl`: `SetPosition(fPosition : LREAL) : BOOL`, accepted only while `NOT bCalibrated AND NOT bHomeAxis
AND NOT fbAxis.HomeActive AND NOT fbAxis.Busy`, through the same `fbSetPosition` instance (one owner of `MC_SetPosition`). The Visu
part (input + button, MAIN helper bit like `bVisuHomeElevation`, `MAIN.TcPOU:121-123`) is a separate MONETS/MONETN change after the
release.

### 3.4 Behaviour after the change

| Situation | Before | After |
|---|---|---|
| MS-95: calibrated elevation at 40, re-home fails after 53 deg up | -9 999 999 947, jog refused (0x4460) until XAE | after `reset=1`: about 93 clamped to 90.5 (MAX 91 - 0.5), WARN; uncalibrated; jog down works |
| Stop during an elevation search | -1e10 + travel | restored, uncalibrated, no error (HomeStopAbort) |
| Failure after the cam flip (sync phase) | +1e10 - travel | restored (the flip is skipped in the sum) |
| Failed power-on homing after a PLC start | -1e10 + travel | unchanged, no restore (start value meaningless, Q3); WARN names it; recover with `reset`/`power` |
| Homing request that never started (timeout, covers) | position unchanged | unchanged, no restore |
| Successful homing | calibrated | unchanged, no restore |
| `reset=1`, `power=1` after a failure | homes (uncalibrated) | same; the restored value is overwritten by the new homing |
| Automatic moves while restored | refused (uncalibrated) | still refused: `MoveTo` and tracking need `Calibrated` (`FB_Axis.TcPOU:186`, `:236`) |

## 4. Affected files

HalfBROT (one PR to `develop`):
1. New `HalfBROT/HalfBROT/POUs/FB_HomeFailPosition.TcPOU` (+ `HalfBROT.plcproj` `<Compile Include>`).
2. `FB_AxisControl.TcPOU`: declarations, method `_HomeFailPosition(sAxis)`, soft-limit read, method `SetPosition` (option B
   plumbing), LineIds.
3. `FB_AzimuthControl.TcPOU` (after `:79`) and `FB_ElevationControl.TcPOU` (after `:126`): `_HomeFailPosition(...)`.
4. README: one sentence (a failed homing restores an estimated, uncalibrated position; jog only).
5. `specs/plans/index.md`: this plan.

No change in BROTLib. MONETcommon: tests and pins only. MONETS/MONETN: pins; Visu Set Position (B) separately.

## 5. Tests and hardware verification

**TcUnit (MONETcommonTests, `FB_HomeFailPosition_Tests`, style of `FB_HomeRequest_Tests`):** search-phase failure (start 40, +53,
fail -> 93 estimate, clamp 90.5); sync-phase failure (flip to +1e10 skipped); reference jump ignored on success, no restore; failure
without virtual position -> no restore; `bCanSet` FALSE (error, `HomeActive` held until reset) -> waits; `bSetError` -> one
`bRestoreFailed`, no retry; new request clears a pending restore; uncalibrated start -> `bStartCalibrated` FALSE; disabled limit
side -> no clamp. Register the suite in `MAIN`, check `MAXNUMBEROFTESTSUITES`. Building/running (XAE, `tctool`): this machine is
the CI runner, ask first.

**Hardware (MONET/S, supervised; needs the user's go-ahead).** Pinned MONETS build; recorder on `fActualPosition`, raw encoder
`In.nDataIn1` of the elevation and azimuth, `fbHomeFailPosition.*`, `fbSetPosition.*`, `bHomeAxis`, `HomeActive`, `bCalibrated`,
`bError`/`nErrorID`. Manual mode, person at the E-stop, covers open, elevation started well below the cam (about 40 deg).

1. Elevation calibrated at 40: Visu Start, `stop=1` after about 10 s (search phase, about 50 deg). Expect: no error, uncalibrated,
   restore WARN with about 50; check against the raw encoder delta (H1) to 0.01 deg.
2. Same, but stop after the cam flip (watch for +1e10): restore matches the raw encoder.
3. After 1: jog down 5 deg and up 5 deg: no `0x4460`; `MoveTo`/`goto` refused (uncalibrated); `Calibrated` stays FALSE (H3).
4. Optional (H4): only if it can be done well away from the end of travel. Skip if the clamp is accepted.
5. `reset=1`, `power=1`: power-on homes normally, calibrated, no restore WARN.
6. Error variant (if a safe way exists, e.g. a drive disable during the search): restore only after `reset=1`, not while the error
   stands.
7. Azimuth: variant 1 once, short stop (5 s), away from the cable-wrap ends.
8. Regression: full power-on from cold, Visu Start to the end on elevation and azimuth (MS-96 path), no restore.

## 6. Releases and pins

1. HalfBROT minor **0.13.0** (new block, new public method) via `release.yml`, unless `develop` moved.
2. MONETcommon: pin HalfBROT 0.13.0 (`MONETcommon.plcproj`, `MONETcommonTests.plcproj`), new suite; patch release.
3. MONETS, MONETN: pin both. Visu Set Position (B) in the same or a later PR (Q1).

## 7. Decisions on the open questions (2026-11-06)

Answers proposed after the first draft; Q6 was checked, Q7 could not be.

1. **Q1 A or B:** A now (automatic restore), B as a follow-up for corrections. The axis stays uncalibrated, so the risk is bounded.
2. **Q2 Layer:** HalfBROT now. Move `FB_HomeFailPosition` to BROTLib only after it has proven itself on hardware.
3. **Q3 Failed homing that started uncalibrated:** **no restore** (changed from the first draft). There is no trustworthy start value,
   so `fStart + fTravel` would be a guess shown as a position. The block restores only when `bStartCalibrated` is TRUE (the MS-95
   case). In the other case the virtual value stays; recovery is `reset=1`, `power=1` (F6), XAE, or later B. Step 3 of the logic and the
   table in 3.4 are read with this rule.
4. **Q4 Clamp:** 0.5 deg inside the window, limits read from the NC at runtime (the 90/91 mismatch then does not matter). MONET/S and
   MONET/N elevation are both MIN 0 (enabled) / MAX 91 (enabled) in the `.tsproj`; read the online value during hardware step 1.
   An estimate above MAX shown as MAX - 0.5 while the telescope is near 93 is acceptable because the axis is uncalibrated and
   the WARN names the estimate.
5. **Q5 Missed cam in MS-95 (H5):** guard D is **not** part of this PR. Open a separate follow-up issue with the MS-95 data (not
   created yet, needs the user's go-ahead). The start estimate is logged in the homing WARN so the data is there.
6. **Q6 MONETN:** checked in `MONETN/MONETN/MONETN.tsproj`: elevation MIN 0 / MAX 91 enabled (`:2931-2932`), azimuth -170 / 300 enabled
   (`:3080-3081`), focus limits not enabled (Range only, `:2622-2623`; not affected). The clamp therefore needs the NC read, not constants.
7. **Q7 Telemetry consumers:** not verifiable here. pyobs and the GUI are not in this tree; in the PLC repos only
   `FB_BaseTelescopeControl` / `FB_RaDecCoordinates` use `fElevationCurrent`. **Open, check before the MONETS/MONETN pin.** The restored
   value is published with `bElevationCalibrated` FALSE.

## 8. Deviations in the implementation

- The position window is not read from the NC (`MC_ReadParameter` names for the soft limits could not be verified without XAE). `_HomeFailPosition` takes
  `fMin`/`fMax` from the block: azimuth `fMinPosition`/`fMaxPosition`, elevation `MAX(0, fMinPosition)`/`MIN(90, fMaxPosition)`. The elevation window is
  therefore 0..90 while the NC allows 91; the clamp is conservative. Set `fMinPosition`/`fMaxPosition` no wider than the NC limits.
- Option B (`SetPosition` method, Visu) and the TcUnit suite (in MONETcommonTests) are not part of this PR.
- Not compiled: no XAE/tctool run (this machine is the CI runner). The CI build of the PR is the first compile.
