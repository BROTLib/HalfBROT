# A latched jog request must not start the axis at the next enable (#148)

**Status: implemented in HalfBROT (section 4, items 1-7), not compiled or run on a PLC; CI build pending. The `FB_JogRequest_Tests` suite (section 5) and the pins (section 6) are follow-ups in MONETcommon.** Facts read on 2026-11-05 from HalfBROT `origin/develop` @
`0382fa6` (0.11.1), BROTLib `origin/develop` @ `84c53c0` (0.18.0), MONETcommon `origin/develop` @ `f9c7d5d` (pins HalfBROT 0.11.1),
MONETS `origin/develop` @ `e7247a4`, MONETN `origin/develop` @ `b2f9d29`. Line numbers are at those commits. Suggested branch
`fix/latched-jog-148` from `origin/develop` (the local HalfBROT checkout is 18 commits behind; do not branch from it).

Issue: HalfBROT#148, split out of #136 (plan `2026-11-03-home-calibrated-axis-136.md`, section 7 item 8). MONET/S, 2026-11-03,
MS-95: after a failed homing a latched `bMoveNeg` started the axis at the next enable. Open question of the issue: clear the manual
move requests when the axis is disabled or a homing fails, so an enable never starts a move by itself. Related: code review H4
(`2026-09-20-code-review.md`, pendant leaves `MovePos`/`MoveNeg` set), MONETcommon#3 (`FB_MonetPendantControl._ReleaseMove`).

## 1. Root cause

### 1.1 The request is a plain stored level

| Where | What |
|---|---|
| BROTLib `FB_BaseAxis` `VAR_INPUT` (`FB_BaseAxis.TcPOU:12-13`) | `bMoveNeg`, `bMovePos : BOOL`; properties `MoveNeg`/`MovePos` (`:202-239`) only copy the value. Nothing in BROTLib or HalfBROT ever clears them. |
| BROTLib `FB_Axis` (`FB_Axis.TcPOU:264-266`) | `JogForward := Jog_Forward AND Ready AND Enable_Positive AND NOT ExtActive`, `JogBackwards := Jog_Backwards AND Ready AND Enable_Negative AND NOT ExtActive`. `Ready` is the `MC_Power` status of the same call (`:126-136`), so a request that is TRUE while the axis is disabled gives a rising `JogBackwards` (MC_Jog starts) in the first cycle the drive is powered. The same happens when a limit releases (`Enable_Negative` rises) or tracking ends (`ExtActive` falls). |
| HalfBROT `FB_AzimuthControl:61-62`, `FB_ElevationControl:111-112`, `FB_DerotatorControl:67-68` | `Jog_Forward := bMovePos`, `Jog_Backwards := bMoveNeg`, no gate. |
| HalfBROT `FB_FocusControl:259-260` | `Jog_* := bMove* AND bFocusUnlocked`; `fbJogLatch` (`:106`) takes the same expression. `bFocusUnlocked` is the delayed `(bManualUnlock OR bEnable) AND NOT bError` (`:193-196`), so a latched request fires when the focus is unlocked (pendant Start toggles `Enable`, `bManualUnlock`). `FB_FocusControl` extends `FB_BaseAxis` directly, **not** `FB_AxisControl`. |
| `FB_HomeRequest` / `_HomeRequest` (`FB_AxisControl:61-97`) | Ends a failed homing (`bFailed`, `bTimedOut`), drops `bHomeAxis`; does not touch the jog requests. |

### 1.2 Who writes the requests

| Writer | Behaviour |
|---|---|
| Visu, `Azimuth/Elevation/Derotator/Focus.TcVIS` in HalfBROT, MONETS (`MONETSRuntime/VISUs`) and MONETN (`MONETNRuntime/VISUs`), elements `GenElemInst_49` etc. (`:2693`, `:2795`; focus `:3867`, `:3969`) | `VisuFbElemPushSwitch` on `MAIN.<Axis>Control.bMoveNeg/bMovePos`, with a member set to `"TOGGLE"` in the same element: most likely a toggle input (one click TRUE, next click FALSE). Such a request **stays TRUE by design** until clicked again, through a disable, an error or a failed homing. This is the MS-95 path. To verify in XAE (element input configuration), see Q4. |
| HalfBROT `FB_PendantControl:179-180, 192-193, 200-201, 215-216, 260-263` | Written every cycle while the selector branch is active (some outside `IF key_switch`); leaves the last value when the selector changes (code review H4, still open in HalfBROT). |
| MONETcommon `FB_MonetPendantControl:180-227`, `_ReleaseMove:248-264` | Production pendant: every cycle while selected, set FALSE once when the pendant lets go (#3). Hold-to-run: a direction held while Start enables the axis starts the jog. |
| MONETcommon `FB_MonetTelescopeControl:1589-1594` | Sets all six FALSE while a suspend stop runs (NC rejects a jog during `MC_Stop`). |
| BROTweb / HMI gateway | No reference to the jog flags today (grep, 2026-11-05). |
| IAG50cm `FB_HourAngleControl:74`, `FB_DeclinationControl:73`, `FB_FocusControl:72` | Same direct wiring; not a HalfBROT consumer (see Q2). |

So the bug is not limited to a failed homing: any request that is TRUE while the axis cannot jog (disabled, error, homing, at the
limit, tracking) fires as soon as the condition goes away.

## 2. Design

Rule: **a jog runs only for a request that rose while the axis could jog.** A request seen while the axis cannot jog is cleared
(so the Visu toggle, the HMI and the telemetry show it released) and, if the writer keeps writing it (pendant held), it is
ignored until it has been FALSE once while the axis can jog.

### 2.1 New pure-logic `FB_JogRequest` (HalfBROT `POUs/`)

```
// Jog requests of an axis block (#148). A jog reaches fbAxis only for a request that rose (or was released and pressed again)
// while the axis could jog; while it cannot (not Ready, error, homing, tracking) the requests are cleared and disarmed, so an
// enable, a reset, the end of a homing or of tracking never starts a jog by itself. Per direction, the limit disarms too.
// Call once per cycle before fbAxis(...); on bClearPos/bClearNeg the caller writes bMovePos/bMoveNeg := FALSE.
FUNCTION_BLOCK FB_JogRequest
VAR_INPUT
	bMovePos, bMoveNeg	: BOOL;	// the stored requests (Visu, pendant, HMI)
	bCanJog				: BOOL;	// fbAxis.Ready of the last call AND NOT bError AND NOT homing AND NOT tracking (per block, 2.2)
	bAllowPos, bAllowNeg: BOOL;	// the Enable_Positive/Enable_Negative the block passes to fbAxis (limit free)
END_VAR
VAR_OUTPUT
	bJogPos, bJogNeg	: BOOL;	// to Jog_Forward/Jog_Backwards
	bClearPos, bClearNeg: BOOL;	// clear the stored request in this cycle
END_VAR
VAR
	bArmedPos, bArmedNeg: BOOL;	// the request was FALSE at least once while the direction could jog
END_VAR

bArmedPos := bCanJog AND bAllowPos AND (bArmedPos OR NOT bMovePos);
bArmedNeg := bCanJog AND bAllowNeg AND (bArmedNeg OR NOT bMoveNeg);
bJogPos := bMovePos AND bArmedPos AND NOT bMoveNeg;	// both pressed: no jog (today MC_Jog gets both and errors)
bJogNeg := bMoveNeg AND bArmedNeg AND NOT bMovePos;
bClearPos := bMovePos AND NOT (bCanJog AND bAllowPos);
bClearNeg := bMoveNeg AND NOT (bCanJog AND bAllowNeg);
```

Properties:
- Scan-order safe: `bCanJog` uses `fbAxis.Ready` of the last call, so in the cycle `Ready` rises inside `fbAxis` the gate is still
  closed; next cycle the request (if still TRUE because the pendant rewrites it) is not armed. No one-cycle jog pulse.
- A held pendant direction while Start enables the axis does not jog; release and press again. A held jog that loses `Ready` (drive
  error, enable dropped) stops (MC_Jog falling input, as today) and needs a new press.
- Clearing happens every cycle while blocked, not only on the edge: a Visu toggle clicked on a disabled axis pops back at once
  (visible feedback that it was not accepted).
- "Both directions" is a small extra (today `MC_Jog` gets both inputs); see Q5.

### 2.2 Wiring

`FB_AxisControl` (azimuth, elevation, derotator): declaration `fbJogRequest : FB_JogRequest;` and a helper method like `_HomeRequest`:

```
// Jog requests (#148), call every cycle right before fbAxis(...), with the Enable_Positive/Negative the block passes to it.
METHOD PROTECTED _JogRequest
VAR_INPUT
	bAllowPos, bAllowNeg : BOOL;
END_VAR

fbJogRequest(
	bMovePos  := bMovePos,
	bMoveNeg  := bMoveNeg,
	bCanJog   := fbAxis.Ready AND NOT bError AND NOT bHomeAxis AND NOT fbAxis.HomeActive AND NOT fbAxis.HomePending
	             AND NOT bTracking AND NOT fbAxis.isTracking,
	bAllowPos := bAllowPos,
	bAllowNeg := bAllowNeg);
IF fbJogRequest.bClearPos THEN bMovePos := FALSE; END_IF
IF fbJogRequest.bClearNeg THEN bMoveNeg := FALSE; END_IF
```

Each block, right before its `fbAxis(...)` call (the limit expressions are the ones already in the call; the `OR bHomeAxis` homing
override is irrelevant because homing closes `bCanJog`):

| Block | Call | `fbAxis` inputs |
|---|---|---|
| `FB_AzimuthControl` (before `:53`) | `_JogRequest(bAllowPos := inDigitalInputs.1, bAllowNeg := inDigitalInputs.0)` | `Jog_Forward := fbJogRequest.bJogPos`, `Jog_Backwards := fbJogRequest.bJogNeg` (`:61-62`) |
| `FB_ElevationControl` (before `:103`) | `_JogRequest(bAllowPos := inDigitalInputs.0, bAllowNeg := inDigitalInputs.1)` | same (`:111-112`) |
| `FB_DerotatorControl` (before `:59`) | `_JogRequest(bAllowPos := fbAxis.ActualPosition < fMaxPosition, bAllowNeg := fbAxis.ActualPosition > fMinPosition)` | same (`:67-68`) |

`FB_FocusControl` (extends `FB_BaseAxis`, so it gets its own instance, no helper): before `fbJogLatch` (`:106`)

```
fbJogRequest(
	bMovePos  := bMovePos,
	bMoveNeg  := bMoveNeg,
	bCanJog   := fbAxis.Ready AND bFocusUnlocked AND NOT bError
	             AND NOT bHomeAxis AND NOT fbAxis.HomeActive AND NOT fbAxis.HomePending,
	bAllowPos := NOT IbLimitFar,
	bAllowNeg := NOT IbLimitNear);
IF fbJogRequest.bClearPos THEN bMovePos := FALSE; END_IF
IF fbJogRequest.bClearNeg THEN bMoveNeg := FALSE; END_IF
fbJogLatch(bJogCommand := fbJogRequest.bJogPos OR fbJogRequest.bJogNeg, bStandStill := fbAxis.StandStill);
```

and `Jog_Forward := fbJogRequest.bJogPos`, `Jog_Backwards := fbJogRequest.bJogNeg` (`:259-260`). `bFocusUnlocked` here is the value
of the last cycle (it is computed at `:196`, after the latch), as the latch uses it today. Update the comment in
`FB_FocusJogLatch:8` (`bJogCommand` is now the gated jog).

Not changed: `FB_BaseAxis` (BROTLib), the property setters, `FB_PendantControl` (H4 stays its own fix), `FB_HomeRequest` (a failed
homing is covered because homing and error close `bCanJog`, and a failed homing leaves the request cleared).

### 2.3 Behaviour after the change

| Situation | Before | After |
|---|---|---|
| Visu toggle `bMoveNeg` set, homing fails, axis disabled, enabled again (MS-95) | jogs at `Ready` | request cleared during the homing; no motion |
| Visu toggle clicked on a disabled axis | stays TRUE, jogs at enable | pops back FALSE |
| Pendant direction held, Start pressed | jogs at `Ready` | no jog until released and pressed again |
| Jog held, drive error, Reset | jogs again after Reset if still TRUE | request cleared on the error; new press needed |
| Visu toggle `bMoveNeg` set at the negative limit, axis moved off the limit by `MoveTo` | jogs back into the limit | cleared at the limit |
| Jog requested during tracking | waits, jogs when tracking ends | cleared |
| Normal jog (enabled, idle, press, release) | jogs | unchanged |
| Focus jog while locked | waits for unlock | cleared; press after the unlock |

## 3. Alternatives considered

| Option | Verdict |
|---|---|
| **A. `FB_JogRequest` in HalfBROT, clear + arm (above)** | **Recommended.** Covers Visu toggle (clear) and the pendant (arm), all blocking conditions, testable as pure logic, HalfBROT-only release. |
| B. Clear only: `bMovePos/bMoveNeg := FALSE` in each block while `NOT fbAxis.Ready OR bError OR homing` (issue's suggestion) | Fixes MS-95 (the Visu writes only on click). A writer that rewrites every cycle (pendant held through Start, an HMI that re-sends its state) still starts the axis at `Ready`. Acceptable as a hotfix if A is too large. |
| C. Arm only, no clear | Pendant fixed, but the Visu toggle stays shown as pressed while nothing jogs, and jogs again after a release-and-press the operator does not expect. Rejected. |
| D. In BROTLib `FB_Axis`: jog only on a rising `Jog_*` while `Ready` | Fixes IAG50cm too, but `FB_Axis` cannot clear the caller's stored request (Visu stays latched), changes `Jog_*` semantics ("a level, read only") for every consumer, and needs BROTLib -> HalfBROT -> MONETcommon -> MONETS/MONETN releases. Rejected for #148; see Q2 for moving `FB_JogRequest` to BROTLib later. |
| E. In BROTLib `FB_BaseAxis`: clear in the setters or a base method | `FB_BaseAxis` has no body that runs every cycle and does not know `Ready`/limits of the concrete block; still a BROTLib release. Rejected. |
| F. Visu only: change the push switches to momentary (tap) | Hold-to-run in the Visu is a good idea anyway (Q4), but a web Visu that loses the mouse-up (connection drop, tab closed) still leaves the flag TRUE; the PLC must not rely on it. Complementary, not a fix. |

## 4. Files and functions to change

HalfBROT (one PR to `develop`):
1. New `HalfBROT/HalfBROT/POUs/FB_JogRequest.TcPOU` (+ `HalfBROT.plcproj` `<Compile Include>`).
2. `FB_AxisControl.TcPOU`: declaration `fbJogRequest : FB_JogRequest;`, new method `_JogRequest(bAllowPos, bAllowNeg)` (+ LineIds).
3. `FB_AzimuthControl.TcPOU`: `_JogRequest(...)` before `fbAxis(` (`:53`), `Jog_*` at `:61-62`.
4. `FB_ElevationControl.TcPOU`: same before `:103`, `:111-112`.
5. `FB_DerotatorControl.TcPOU`: same before `:59`, `:67-68`.
6. `FB_FocusControl.TcPOU`: declaration `fbJogRequest : FB_JogRequest;`, body `:105-106`, `:259-260`.
7. `FB_FocusJogLatch.TcPOU`: comment of `bJogCommand` (`:8`).
8. `specs/plans/index.md`: add this plan.

No change in BROTLib. No code change in MONETcommon/MONETS/MONETN (pins only, section 6); the Visu change is Q4.

## 5. Tests (MONETcommonTests, TcUnit; HalfBROT has no test project)

New `FB_JogRequest_Tests` (`HalfBROT.FB_JogRequest`), style of `FB_HomeRequest_Tests` / `FB_FocusJogLatch_Tests` (one FB instance per
method, inputs stepped cycle by cycle, `TEST_FINISHED()`):

1. `Press_While_Ready_Jogs` - can jog, request rises -> `bJogNeg` TRUE, no clear; release -> FALSE.
2. `Latched_Before_Ready_No_Jog` - request TRUE, not ready -> `bClearNeg` TRUE, no jog; ready rises with the request still TRUE
   (writer rewrites) -> no jog for 10 cycles. **(#148 core, MS-95)**
3. `Release_Then_Press_After_Ready_Jogs` - continue 2: request FALSE one cycle, TRUE -> jog.
4. `Held_Through_Enable_Pendant` - request TRUE every cycle across `bCanJog` FALSE -> TRUE -> no jog until a FALSE cycle.
5. `Ready_Lost_While_Jogging` - jogging, `bCanJog` FALSE -> jog FALSE, clear TRUE; `bCanJog` TRUE again with request rewritten -> no jog.
6. `Homing_Clears` / `Error_Clears` - documented as `bCanJog` FALSE cases on the block wiring (the FB sees only `bCanJog`); one
   method each with the expected expression evaluated in the test, to pin the intent.
7. `Limit_Clears_Direction_Only` - `bAllowNeg` FALSE: neg cleared and not jogging, pos still jogs.
8. `Limit_Released_Does_Not_Start` - neg request at the limit (rewritten every cycle), limit frees -> no jog.
9. `Both_Directions_No_Jog` - both TRUE -> neither output.
10. `Idle_No_Output` - no request: no jog, no clear, whatever the other inputs.
11. `Regression_Old_Wiring_Would_Jog` (optional, documents the bug): the old `Jog_Backwards AND Ready` expression on the trace of 2
    is TRUE in the first ready cycle.

`FB_FocusJogLatch_Tests` stays (the latch is unchanged). The block bodies cannot run in a test (MC blocks, `AXIS_REF`; the `FB_*Mock`
skip the body): the wiring is checked by review and the CI compile. Register the suite in `MAIN`; `MAXNUMBEROFTESTSUITES` is 41 on
`develop` (`MONETcommonTests.plcproj:232-233`), raise it if the count in `MAIN` reaches it. Run the whole MONETcommonTests suite to
confirm nothing else changed (`FB_MonetPendantControl_Tests` asserts command levels on mocks and is not affected).

Building/running the tests (XAE, `tctool`) is not part of writing this plan: this machine is the CI runner, ask first.

### Hardware verification (MONET/S, supervised; needs the user's go-ahead)

Pinned MONETS build with the fixed HalfBROT; recorder on `bMoveNeg/bMovePos`, `fbJogRequest.*`, `fbAxis.Ready`, `bHomeAxis`,
`bError`, actual velocity. Manual mode, person at the E-stop, elevation away from its limits.

1. MS-95 repro: elevation disabled, Visu `-` clicked -> button pops back; enable -> no motion.
2. Visu `-` clicked on an enabled elevation -> jogs; click again -> stops. Same for azimuth, derotator, focus (focus unlocked first).
3. Visu `-` set during a homing -> cleared, homing continues; make the homing fail (Stop) -> enable again -> no motion.
4. Pendant: hold a direction, press Start -> no motion; release, press -> jogs.
5. Jog, then disable by the pendant -> stops; enable -> no motion.
6. Jog toward a limit until the switch -> request cleared; move off with a target -> no jog back.
7. Regression: power-on from cold, tracking, normal pendant jogs on every axis.

## 6. Releases and pins

1. HalfBROT minor bump (new block, behaviour change of the jog inputs): **0.12.0** via `release.yml`, unless `develop` has
   moved by then. No BROTLib change.
2. MONETcommon: pin HalfBROT 0.12.0 (`MONETcommon.plcproj:131-133` and `MONETcommonTests.plcproj:191-193`), new suite; patch
   release.
3. MONETS and MONETN: pin HalfBROT 0.12.0 and the new MONETcommon; optional Visu change (Q4) in the same PRs or separately.
4. IAG50cm: not affected by the release (no HalfBROT), same latent bug, see Q2.

## 7. Risks and open questions

1. **Q1 Clear while blocked, or only arm?** Clearing changes values written by the Visu/HMI from the PLC side; a writer that compares
   its own last write with the PLC value may be confused. Today only the Visu and the pendants write them (BROTweb does not).
   **Recommendation:** clear + arm (option A); clearing is what makes the Visu show the truth.
2. **Q2 Layer: HalfBROT now, BROTLib later?** IAG50cm wires `bMoveNeg` the same way and has the same bug. **Recommendation:** fix in
   HalfBROT for #148 (MONET/S incident, no BROTLib release chain); open a BROTLib issue to move `FB_JogRequest` (it has no HalfBROT
   dependency) to BROTLib and use it in IAG50cm, after which HalfBROT can switch to the BROTLib block.
3. **Q3 Tracking in `bCanJog`.** `FB_Axis` already ignores a jog while `ExtActive`, but keeps it and starts it when tracking ends.
   `ExtActive` is private; `bTracking OR fbAxis.isTracking` is the closest public signal (misses the short "being disabled" phase,
   still covered by `FB_Axis`'s own gate). **Recommendation:** include it (same bug class); focus has no tracking.
4. **Q4 Visu push switches are toggles.** The Visu `PushSwitch` elements on `bMoveNeg/bMovePos` carry a `"TOGGLE"` member, which
   reads as toggle input: one click starts a jog that runs until the next click. Verify in XAE; if so, a momentary (tap, hold-to-run)
   input is safer for a motion button. **Recommendation:** separate MONETS/MONETN (and HalfBROT demo Visu) change to momentary after
   the HalfBROT fix; the PLC fix is needed either way (lost mouse-up).
5. **Q5 Both directions at once.** Today both reach `MC_Jog`. **Recommendation:** suppress both (in `FB_JogRequest`); drop it if the
   review prefers a pure #148 change.
6. **Q6 Clear on a short `Ready` drop.** The azimuth `bAtRest`/brake logic and the elevation brake settle can drop `fbAxis.Ready`
   briefly; a held Visu toggle is then cleared and must be clicked again. Intended (any enable must not start a move), but note it
   for the operators.
7. **Q7 Pendant H4 (HalfBROT `FB_PendantControl`).** Still leaves requests set on a selector change; with this fix they no longer
   start the axis after a disable, but an enabled axis keeps jogging. **Recommendation:** keep H4 as its own small change (MONETcommon
   already has `_ReleaseMove`).
8. **Risk: focus jog interplay.** `fbJogLatch.bJogging` now follows the gated jog. A request cleared while locked no longer counts as
   a jog in `F_FocusTargetFollowsActual` / the `fLastPosition := -1` rule (`:205-207`), which is correct (nothing moved), but review
   `F_FocusTargetFollowsActual_Tests` assumptions.
9. **Risk: scan order.** `_JogRequest` must run before `fbAxis(...)` and use last-cycle `fbAxis.Ready`; calling it after `fbAxis`
   would let the first ready cycle pass the old value. Covered by test 2 and review.
