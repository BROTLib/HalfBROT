# Homing a calibrated axis: keep `bHomeAxis` until the homing has run (#136)

**Status: decisions taken (see "Decisions"), implementation on `fix/home-calibrated-axis-136`; nothing compiled or run on a PLC.** Facts read on 2026-11-03 from HalfBROT `develop` @ `91a782f`
(0.10.2), BROTLib tag `v0.17.0` (the local BROTLib checkout is a detached bisect at 0.15.1; the homing lines quoted below are
identical in both), MONETcommon `develop` @ `5ffc0aa` (0.14.0, pins BROTLib 0.17.0 / HalfBROT 0.10.2), MONETS `develop` @ `3757306`.
Line numbers are at those commits. Suggested branch `fix/home-calibrated-axis-136` from `origin/develop`.

Issue: HalfBROT#136 (MONET/S, 2026-11-03, MONETS#77 Visu Start buttons, reproduced twice on the elevation). Related: MONETS#72/#77,
BROTLib#62/#63/#92 (`Home(fReference)`, command handshake, `HomeActive`), HalfBROT focus manual re-home (`F_FocusHomeRelease`,
`FB_FocusManualHome`), BROTLib plan `2026-10-06-fb-axis-rehome-check-and-0.5.5-release.md` (open question: does `MC_Home` clear
`Homed`; #136 answers it, see 1.2).

## Decisions (2026-11-03, after review)

| | Decision |
|---|---|
| Q1 | `tHomeStartTimeout` default `T#30S`. |
| Q2 | The MONETS Visu Start buttons work only from a stopped telescope (not tracking, no move running, no automatic target). Separate small MONETS change after the HalfBROT release; not in the HalfBROT PR. |
| Q3 | Guard: the elevation never homes (and so never moves) with closed covers. In `FB_ElevationControl`, before `fbAxis(...)`: a homing request (`bHomeAxis`) that has not started (`NOT fbHomeRequest.bStarted`) while `NOT (bConnected AND fbCovers.Opened)` is dropped through `THIS^.HomeAxis := FALSE` (`CancelHome` runs before `fbAxis` executes it) with a WARN. Covers are never opened by a re-home: with the guard the covers are open when the sweep starts, so the `NOT bCalibrated` branch (`:41-45`) only calls `Open()` on open covers. The power-on is unaffected (it homes only after `Ready`, which needs the brake, which needs the covers). Add a `FB_ElevationControl`-level test only if the interface mocks allow; otherwise review plus hardware step 7. |
| D2 | Helper method `_HomeRequest(sAxis : STRING)` in `FB_AxisControl` (like `_CheckDiagnostic`), holding `fbHomeRequest`, `tHomeStartTimeout` and the events; each block calls it once, right after its `fbAxis(...)` call. |
| D3 | Focus stays on `F_FocusHomeRelease`; migrating it is a separate issue after the hardware check. |
| D4 | Azimuth and derotator drop `bMoveAxis`/`bTracking` while homing in this PR (as the elevation already does). |

## 1. Root cause

### 1.1 What the library does

| Where (BROTLib 0.17.0) | What |
|---|---|
| `FB_BaseAxis.HomeAxis.Set` (`FB_BaseAxis.TcPOU:156-161`) | FALSE -> TRUE: `_StartHoming()`; FALSE: `fbAxis.CancelHome()`; then `bHomeAxis := HomeAxis`. TRUE while already TRUE does nothing. |
| `_StartHoming` (az `FB_AzimuthControl:131`, el, derot) | `fbAxis.Home(fReference := fCalibPosition)` |
| `FB_Axis.Home()` (`FB_Axis.TcPOU:425-432`) | ignored while `HomeActive`; else `HomeCommand.Request()` and **`HomePending := TRUE` at once** (synchronously, in the caller's scan) |
| `FB_Axis` body (`:270-279`) | `HomeCommand(Allowed := Ready, ...)` turns Pending into a rising `Execute` (one cycle low first if Execute is still TRUE); `HomePending := HomeCommand.Pending`; `HomeActive := HomeCommand.Execute OR Axis_Home.Busy` |
| `FB_Axis` body (`:281-289`) | `Calibrated` and `HomeDone` both copy the NC level `Status.Homed` (HomeDone is **not** a pulse) |
| `CancelHome()` (`:469-471`) | drops a pending request only; a running homing continues |

So in the scan in which `HomeAxis := TRUE` is written, before the axis body runs: `HomePending` TRUE, `HomeActive` FALSE, `Calibrated`
TRUE (stale, the NC has not started). `HomeActive` only becomes TRUE inside `fbAxis(...)`.

### 1.2 What the axis blocks do with it (HalfBROT 0.10.2)

| Block | Before `fbAxis(...)` | After `fbAxis(...)` |
|---|---|---|
| `FB_AzimuthControl` | `:33-35` `IF (bCalibrated AND NOT fbAxis.HomeActive) OR bError THEN bHomeAxis := FALSE` | `:78-80` `IF fbAxis.HomeDone AND NOT fbAxis.HomeActive` |
| `FB_ElevationControl` | `:60-62` same as azimuth | `:107-109` same as azimuth |
| `FB_DerotatorControl` | `:34-40` `IF fbAxis.Calibrated THEN ... IF NOT fbAxis.HomeActive THEN bHomeAxis := FALSE` | `:82-84` `(HomeDone AND NOT HomeActive) OR bReset OR bError`; `:52-58` range stop clears it too |
| `FB_FocusControl` (reference) | none | `:281-283` `F_FocusHomeRelease(bCalibrated, HomeActive, HomePending, bError)` = `bError OR (bCalibrated AND NOT HomeActive AND NOT HomePending)` |

Confirmed failure sequence for a calibrated azimuth/elevation (scan N = the scan with the Start press, MONETS `MAIN.TcPOU:107-122`
writes `HomeAxis` before the axis bodies run):

1. Scan N, setter: `Home()` -> `HomePending` TRUE, `bHomeAxis` TRUE.
2. Scan N, body: the pre-`fbAxis` clear sees `Calibrated AND NOT HomeActive` -> `bHomeAxis := FALSE`. It ignores `HomePending`.
3. Scan N: `MC_SetAcceptBlockedDriveSignal(Enable := bHomeAxis)` and `Enable_Positive := limit OR bHomeAxis` are evaluated FALSE;
   `fbAxis` starts `MC_Home` anyway (the request is in `HomeCommand`, a direct `bHomeAxis := FALSE` does not call `CancelHome`).
4. The sweep runs without the limit/blocked-drive overrides and fails after about 54 s with `0x4B0A` "Error during referencing"
   (`NCError_TO_STRING:28`), emergency park, `0x4260`.

The second clear is also wrong on its own: `HomeDone` is the stale `Homed` level, so in any scan where the request is pending but not
yet active (axis not `Ready`, or the one-cycle low of `FB_AxisCommand`) `HomeDone AND NOT HomeActive` releases it too.

The recording in the issue also settles BROTLib's open question: `HomeActive` TRUE 90 ms after the press, `bCalibrated` FALSE 0.2 s
later, i.e. **the NC clears `Homed` when `MC_Home` starts** (and sets it again before `MC_Home` is done, the reason for the existing
"not while HomeActive" rule).

Why the power-on works: `FB_MonetTelescopeControl._PowerOn` (`:1413-1423`) only requests homing when `NOT Calibrated`, so the clear
condition is FALSE. Derotator: homes with `MC_ForceCalibration` (no motion, no override on `bHomeAxis`), so the early clear is
harmless there, but it is the same wrong rule and the same block family; fix it for consistency.

## 2. Design

### 2.1 New pure-logic `FB_HomeRequest` (HalfBROT `POUs/`)

One block, no NC access (testable like `FB_FocusManualHome`, `FB_BrakeSettleEnable`), called **once per cycle right after
`fbAxis(...)`**, replacing every direct `bHomeAxis := FALSE` of the homing end in azimuth, elevation and derotator. Sketch:

```
// End of a homing request (bHomeAxis) of an axis block (#136). The request ends when the homing has run and finished
// (HomeActive seen for this request, now neither active nor pending), when it was dropped before it started (CancelHome,
// Stop, Reset), on an axis error, or when it has waited tStartTimeout without starting. Calibrated alone never ends it:
// it is the stale NC Homed level when a calibrated axis is re-homed, and is set again before MC_Home is done.
FUNCTION_BLOCK FB_HomeRequest
VAR_INPUT
	bRequest		: BOOL;		// bHomeAxis
	bHomeActive		: BOOL;		// fbAxis.HomeActive
	bHomePending	: BOOL;		// fbAxis.HomePending
	bCalibrated		: BOOL;		// fbAxis.Calibrated
	bError			: BOOL;		// bError of the axis block
	tStartTimeout	: TIME;		// pending without start longer than this ends the request; T#0S = wait for ever
END_VAR
VAR_OUTPUT
	bRelease		: BOOL;		// end the request in this cycle (caller: THIS^.HomeAxis := FALSE)
	bStarted		: BOOL;		// HomeActive was seen for the current request
	bDone			: BOOL;		// one cycle: ended after it ran, calibrated, no error
	bFailed			: BOOL;		// one cycle: ended otherwise (error, dropped, uncalibrated, timeout)
	bTimedOut		: BOOL;		// one cycle: the start timeout ended it
END_VAR
VAR
	rtRequest		: R_TRIG;
	tonStart		: TON;
END_VAR

rtRequest(CLK := bRequest);
IF rtRequest.Q THEN bStarted := FALSE; END_IF
bStarted := bRequest AND (bStarted OR bHomeActive);
tonStart(IN := bRequest AND NOT bStarted AND bHomePending AND tStartTimeout > T#0S, PT := tStartTimeout);
bRelease := FALSE; bDone := FALSE; bFailed := FALSE; bTimedOut := FALSE;
IF bRequest THEN
	IF bError THEN
		bRelease := TRUE; bFailed := TRUE;
	ELSIF NOT bHomeActive AND NOT bHomePending THEN		// finished, or dropped before it started
		bRelease := TRUE; bDone := bStarted AND bCalibrated; bFailed := NOT bDone;
	ELSIF tonStart.Q THEN
		bRelease := TRUE; bTimedOut := TRUE; bFailed := TRUE;
	END_IF
END_IF
```

Key property: it never looks at `Calibrated` to decide *whether* to end, only `HomeActive`/`HomePending` (the FB_Axis handshake).
That covers both scan orders (setter before or after the body), the one-cycle low, a not-yet-Ready axis, and the `Homed` drop and
early rise during the sweep. It is a generalisation of `F_FocusHomeRelease` (which differs only in keeping an uncalibrated idle
request alive) plus the "seen active" latch of `FB_FocusManualHome`.

### 2.2 Wiring (azimuth, elevation, derotator)

- New `VAR fbHomeRequest : FB_HomeRequest;` and `VAR_INPUT tHomeStartTimeout : TIME := T#30S;` in each block (or once in
  `FB_AxisControl`, D2).
- Delete the pre-`fbAxis` clears (az `:33-35`, el `:60-62`, derot `:36-39` keeping `bCalibrated := fbAxis.Calibrated`) and the
  post-`fbAxis` clears (az `:78-80`, el `:107-109`, derot `:82-84`). After `fbAxis(...)`:
  ```
  fbHomeRequest(bRequest := bHomeAxis, bHomeActive := fbAxis.HomeActive, bHomePending := fbAxis.HomePending,
  	bCalibrated := fbAxis.Calibrated, bError := bError, tStartTimeout := tHomeStartTimeout);
  IF fbHomeRequest.bRelease THEN
  	THIS^.HomeAxis := FALSE;	// through the property: CancelHome() drops a request that has not started (timeout, error)
  END_IF
  ```
  plus `ADSLOGSTR` WARN on `bTimedOut` ("<axis>: homing did not start within <t>, request dropped") and on `bFailed AND NOT bError`
  ("<axis>: homing ended uncalibrated" / "dropped"), HINT on `bDone` only if wanted (the `CalibrationEvent` already logs).
- Derotator: keep `bReset`/range-stop clears but route them through `THIS^.HomeAxis := FALSE`; `bReset` also releases via the
  handshake (FB_AxisCommand Reset drops Pending), so the explicit `OR bReset` can go.
- Overrides stay where they are (before `fbAxis`, on `bHomeAxis`); with the fix they are TRUE from scan N on, i.e. in the scan
  in which `MC_Home` gets its rising `Execute`, and FALSE from the scan after `HomeActive` ends.
- Focus: unchanged in this PR (`F_FocusHomeRelease` + `FB_FocusManualHome` work and are tested). Migrating it to
  `FB_HomeRequest` is a follow-up (D3).

### 2.3 Behaviour

| Case | Result |
|---|---|
| Re-home a calibrated az/el (Visu Start) | `bHomeAxis` TRUE from the press until `HomeActive` falls; overrides on for the whole sweep; `Calibrated` FALSE during the sweep (NC), TRUE at the end; `bDone` |
| Power-on homing (uncalibrated) | unchanged except the end is one scan later at most (post-`fbAxis` only) |
| Request while axis not `Ready` | waits (`HomePending`) up to `tHomeStartTimeout`, then `CancelHome`, WARN, `bFailed`. Today: pending for ever, overrides on a disabled axis |
| `HomeAxis := FALSE` by the caller (power-on abort `:1379-1384`, derotator `:1423`) | as today: property -> `CancelHome`; a running sweep is not stopped (BROTLib semantics) and `bHomeAxis` FALSE switches the overrides off; unchanged and out of scope |
| `Stop()`/`bReset` during the sweep | `MC_Home` aborted -> `HomeActive` FALSE -> release, `bFailed`; overrides off. Today an uncalibrated axis kept `bHomeAxis` (and the overrides) TRUE for ever after a stop |
| `bError` (e.g. `0x4B0A`) | release in the same cycle, `CancelHome`; as today |
| Second Start press while homing | no edge (bHomeAxis still TRUE), nothing happens; as BROTLib `Home()` ignores a call while active |
| `MoveAxis`/`Tracking` during a re-home | elevation already forces `bMoveAxis`/`bTracking` FALSE while `bHomeAxis` (`:66-70`); azimuth and derotator do not. Add the same two lines to both (D4), so a Visu Start on a tracking azimuth does not run `MC_Home` against the setpoint generator |
| MONETcommon power-on | requests only `NOT Calibrated AND NOT HomeAxis`; stage `home_axes` leaves only when all `Calibrated` and `NOT bBusy`; unaffected. A start timeout during the power-on makes `HomeAxis` FALSE and the stage re-requests next scan (the axis was `Ready` when requested, so a timeout means it lost Ready; acceptable, logged) |

## 3. Alternatives considered

| Option | Verdict |
|---|---|
| **A. `FB_HomeRequest` after `fbAxis`, handshake-based (above)** | **Recommended.** Fixes both clears, both scan orders, adds the start timeout and the stop/abort release, testable. |
| B. Minimal: add `AND NOT fbAxis.HomePending` to the four clear lines (i.e. use `F_FocusHomeRelease` everywhere) | Fixes the reported failure, but a request that never starts hangs, an aborted sweep of an uncalibrated axis keeps the overrides on, and `HomeDone AND NOT HomeActive` stays a second, inconsistent rule. Acceptable as a hotfix if A is too large for the next release. |
| C. Issue's suggestion: only clear on `HomeDone AND NOT HomeActive` | Still releases in a pending-not-active scan (stale `HomeDone`); no abort/timeout path. |
| D. Fix in BROTLib: `Home()` sets `HomeActive` immediately, or a `HomeRequested` output | Changes `HomeActive` semantics for every consumer (IAG50cm `FB_HourAngleControl`/`FB_DeclinationControl`, focus), needs a BROTLib release and re-test of IAG50cm; the information is already there as `HomePending`. Rejected. |
| E. Reject `HomeAxis` on a calibrated axis | Removes the Visu re-home feature (MONETS#77) and the focus manual re-home pattern. Rejected. |

## 4. Files and functions to change

HalfBROT (one PR to `develop`):
1. New `HalfBROT/HalfBROT/POUs/FB_HomeRequest.TcPOU` (+ `HalfBROT.plcproj` `<Compile Include>`).
2. `FB_AzimuthControl.TcPOU`: body `:31-35`, `:37-39` (add `bMoveAxis`/`bTracking` FALSE), `:78-80`; declaration (`fbHomeRequest`, `tHomeStartTimeout`).
3. `FB_ElevationControl.TcPOU`: body `:58-62`, `:107-109`; declaration.
4. `FB_DerotatorControl.TcPOU`: body `:34-40`, `:42-44` (add `bMoveAxis`/`bTracking` FALSE), `:52-58` (via property), `:82-84`; declaration.
5. Comment in `FB_BaseAxis`-facing docs: none in BROTLib; update the HalfBROT comment at the old clear lines (NC clears `Homed` at the start of `MC_Home`, sets it before the end).
6. `specs/plans/index.md`: add this plan.

MONETcommon (after the HalfBROT release): `MONETcommonTests` pin HalfBROT, new suite (section 5), `MONETcommonTests.plcproj:256-257`
`MAXNUMBEROFTESTSUITES` 40 -> 41 or more (`MAIN.TcPOU:46` notes 40 of 40). No change in `FB_MonetTelescopeControl` needed.

MONETS: no code change; `MAIN.TcPOU:106-122` already uses the property. Optionally gate the Visu Start buttons (Q2).

## 5. Tests (MONETcommonTests, TcUnit; HalfBROT has no test project)

New `FB_HomeRequest_Tests` (`HalfBROT.FB_HomeRequest`), style of `FB_FocusManualHome_Tests` (one FB instance per method, inputs
stepped cycle by cycle, `TEST_FINISHED()`), timer cases in the `nCycles` style of `MonetBrakeRelease_Tests` with `tStartTimeout := T#100MS`:

1. `Pending_Calibrated_Keeps` - first cycle of a re-home: request, pending, calibrated, not active -> no release. **(#136 core)**
2. `Active_Keeps_While_Calibrated_Drops_And_Returns` - active with calibrated TRUE, FALSE, TRUE -> no release until active falls.
3. `Done_After_Active` - active then idle, calibrated -> release, `bDone`, `bStarted`.
4. `Ended_Uncalibrated_Fails` - active then idle, not calibrated (stop/abort) -> release, `bFailed`.
5. `Dropped_Before_Start_Fails` - pending then idle without active (CancelHome/Stop/Reset) -> release, `bFailed`, not `bDone`.
6. `Error_Releases` - error while pending, while active, while idle.
7. `Start_Timeout` - pending, never active for > tStartTimeout -> one `bTimedOut`; `T#0S` never times out.
8. `One_Cycle_Low_Keeps` - pending, active, pending (FB_AxisCommand one-cycle low), active -> no release.
9. `New_Request_Resets_Started` - after a done request, a new rising request starts with `bStarted` FALSE.
10. `Idle_Without_Request_No_Output` - no request: no outputs whatever the inputs.

Scan-order regressions (same suite, one method per axis profile, a small in-test model of `FB_Axis`'s
`HomePending`/`HomeActive`/`Homed` sequence in the order the block calls things: setter -> overrides -> `fbAxis` -> `FB_HomeRequest`;
assert `bHomeAxis`, i.e. the override input, is TRUE in **every** cycle from the press to the end of `HomeActive`):

11. `Regression_Azimuth_Calibrated_Rehome` - cam sweep: Homed TRUE at press, FALSE after 20 cycles, TRUE 5 cycles before active ends.
12. `Regression_Elevation_Calibrated_Rehome` - same profile plus a not-`Ready` start (pending for 50 cycles before active).
13. `Regression_Derotator_Calibrated_Rehome` - `MC_ForceCalibration`: Homed stays TRUE, active for 2 cycles.
14. `Regression_Old_Rule_Would_Fail` (optional, documents the bug): the old az/el expression evaluated on the same trace is TRUE in cycle 1.

`F_FocusHomeRelease_Tests` stays. The real block bodies cannot run in a test (MC blocks, `AXIS_REF`; `FB_*Mock` skip the body):
the wiring is checked by review and the CI compile. Run the whole MONETcommonTests suite to confirm nothing else changed.

Building/running the tests (XAE, `tctool`) is not part of writing this plan: this machine is the CI runner, ask first.

### Hardware verification (MONET/S, supervised; needs the user's go-ahead)

Pinned build of MONETS with the fixed HalfBROT; recorder on `bHomeAxis`, `fbAxis.HomePending/HomeActive/Calibrated`,
`fbHomeRequest.*`, AcceptBlockedDriveSignal bit (`nDeCtrlDWord` bit 8), positive limit input, `bError`/`nErrorID`. Manual mode,
person at the E-stop, elevation started away from zenith.

1. Elevation calibrated -> Visu Start: `bHomeAxis` TRUE for the whole sweep, homing finishes, no `0x4B0A`. Repeat twice.
2. Uncalibrated elevation Start (control: unchanged behaviour).
3. Azimuth calibrated -> Start (only after 1 passed; watch the cable wrap, E-stop ready). Derotator calibrated -> Start.
4. Start with the axis disabled: WARN after 30 s, no motion when it is enabled later.
5. Stop during an elevation sweep: overrides off after the stop, `bHomeAxis` FALSE, power-on re-homes normally.
6. Full power-on from cold (regression of the normal path).
7. Elevation Start with closed covers (calibrated axis, covers closed by hand): request dropped, WARN, no motion, covers stay closed.

## 6. Releases and pins

1. HalfBROT minor bump (new block, new input with a default, behaviour change of the abort path): **0.11.0** via `release.yml`.
   No BROTLib change.
2. MONETcommon: pin HalfBROT 0.11.0 (`MONETcommon.plcproj:155-157` and the tests project), tests, `MAXNUMBEROFTESTSUITES`;
   patch release (0.14.1) since only the pin changes.
3. MONETS and MONETN: pin HalfBROT 0.11.0 and the new MONETcommon (local checkouts pin HalfBROT 0.10.2, BROTLib 0.16.0, MONETcommon
   0.11.2 default resolution; check `develop` before pinning, the issue run used BROTLib 0.17.0).
4. IAG: IAG50cm does not use HalfBROT, no pin. Add the hardware check as an MS row in the IAG test plan; note on MONETS#77.

## 7. Risks and open questions

Items 2-7 below are decided, see "Decisions" at the top; they are kept for the reasoning.

1. **`HomeActive` sticking** (BROTLib #92 plan, open question 2): if `MC_Home` reports Error rather than CommandAborted on a stop,
   `HomeActive` may stay TRUE and `bHomeAxis` with it; `bError` covers the error case. No sweep timeout is added (releasing the
   overrides mid-sweep is exactly this bug); the NC's own `0x4B0A` bounds a failed sweep.
2. **Q1 start timeout value** `T#30S`: elevation `Ready` needs enable + 3 s settle + 6 s; 30 s leaves margin. Per block input so MAIN
   can change it; `T#0S` = old "wait for ever".
3. **Q2 Visu re-home in automatic mode**: a re-home drops `Calibrated` for the sweep; in automatic mode `FB_MonetTelescopeControl`
   then sees an uncalibrated axis (bReady/bHomed FALSE, possibly an error park). Should MONETS `MAIN` accept the Start buttons only in
   manual mode / while not tracking? Proposed: yes, in MONETS, separate small change.
4. **Q3 elevation covers on re-home**: `FB_ElevationControl:41-45` opens the covers while `NOT bCalibrated`, so a re-home with closed
   covers starts opening them mid-sweep (brake already open). Acceptable or guard with `NOT bHomeAxis`?
5. **D2 placement**: `FB_HomeRequest` and `tHomeStartTimeout` per block or once in `FB_AxisControl` (cleaner, but the call must
   still sit after each block's `fbAxis` call, so a helper method `_HomeRequest(sAxis)` like `_CheckDiagnostic`). Proposed: helper
   method in `FB_AxisControl`.
6. **D3 focus**: migrate to `FB_HomeRequest` later or leave `F_FocusHomeRelease`? Proposed: later, separate issue.
7. **D4 azimuth/derotator drop Move/Tracking while homing**: in this PR (proposed) or separately.
8. Out of scope, from the issue comments: a failed sweep leaves the elevation at about -1e10 with software limits active (only an
   XAE Set Position recovers it), and a latched Visu `bMoveNeg` moves the axis at the next enable. Separate issues.
