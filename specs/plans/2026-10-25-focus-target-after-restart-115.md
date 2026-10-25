# Focus target after a PLC restart: first calibration takes the actual position (#115)

**Status: proposed, Q1-Q5 decided (section 6). Nothing implemented, compiled or run on a PLC for the fix.** The restart case is reproduced on
hardware (D1, below); the power-on case (second #115 comment) is not explained yet and is not part of the code change
proposed here. Facts read on 2026-10-25 from HalfBROT `develop` @ `4627b62` (0.7.2), BROTLib `develop`, MONETcommon
working tree and MONETS `develop`. Line numbers are `FB_FocusControl.TcPOU` at that commit.

Issue: HalfBROT#115. Related: #66 (the stored position now persists, which makes this visible), #111 / #108 / #110 /
#114 (plan `2026-10-25-focus-enable-handling-108-110-111-114.md`, branch `fix/focus-enable-handling`, other session),
MONETS#53 (near switch not found: a run to 0.5 is a hardware risk), MONETcommon#122 (range 0.5..91.5).

## 1. Problem

### 1.1 Restart (verified on hardware)

`fPosition` is `VAR_INPUT` of `FB_BaseAxis` without an initial value (`FB_BaseAxis.TcPOU:10`), so it is 0 after a PLC
start. The NC axis keeps its calibration over a PLC restart, so `fbAxis.calibrated` is TRUE from the start and the
`ELSE` branch (`:152`, homing) and with it `bHomeAxis` and `:205` never run. Nothing sets the target:

| Line | What | Effect in the first scan with a calibrated axis |
|---|---|---|
| `:87` | `fPosition := LIMIT(fMinPosition, fPosition, fMaxPosition)` | 0 becomes 0.5 |
| `:102-112` | `fbFollower`, active when automatic, not manually unlocked, not jogging, calibrated, not homing | target 0.5, actual 36: `bStartMove`, `bNeedEnable` |
| `:142-146` | `MoveTo()`, `bEnable := TRUE` | the focus drives to 0.5 at 0.2 mm/s, the brake is open |
| `:129-136` | `F_FocusTargetFollowsActual` | only outside automatic, manual unlock or a jog: does not help here |

Measured (#115 comment of 13:43, HalfBROT 0.7.2, MONET/S, ADS 50 ms): PLC stop and start at 35.997 mm, automatic mode, no
command. First record: `calibrated` TRUE, `bHomeAxis` FALSE, `fPosition` 0.000, `bEnable` and `ObFocusUnlock` TRUE. 58 ms
later `fPosition` 0.500 and `bMoving` TRUE.

Consequences seen in the issue: the focus drives 35 mm towards the low end without a request, `focus=36` over MQTT does
not help in time, the running move is not reversed by a new target (#103, not fixed here), and `bEnable := FALSE` as the
only way to stop it raised 0x4260 and set `fLastPosition` to -1 (`:266`), so the stored position was lost.

### 1.2 Power-on that calibrates from a stored position (H-205 confirmed by D6, fix proposed in 2.4)

Second #115 comment: the target was 0.5 at the end of a power-on. From reading the code: no outside writer (power-on
stages, park stage 0 = `focusPark` 36.0, MQTT range check). Forced run D4 with a valid `fLastPosition` in MC_Direct
mode: no 0.5. The remaining hypothesis is H-205: `:205` sets `fPosition := fLastPosition` every cycle while `bHomeAxis`
is TRUE in MC_Direct mode; with `fLastPosition` -1 (hand write, `:168`, `:187`, `:266`) the target is -1 and the clamp
makes it 0.5. Confirmed by D6 (below). Still open: why the focus calibrated from the stored value after -1 was written is open (stale
`fLastPposition`, or -1 written after `_StartHoming`). D6 (2026-10-25, 0.7.2, MONET/S, ADS 2.9 ms per read, telescope unpowered): uncalibrated focus, `fLastPposition` -1 read before, `fLastPosition := 36.27`, `power=1` lifted the block; with `bHomeAxis` TRUE and `eHomingMode` 4 (MC_Direct) `fLastPosition := -1` was written. 0 ms: `fPosition` -1; 141 ms: `calibrated` TRUE at 36.27, `fPosition` still -1; 161 ms: homing over (`rtHomed` set `fLastPosition` back to the actual position), `fPosition` 0.500 and it stays. The move to 0.5 then sat queued with a locked drive (#111/#108) and would have run with the next enable; it was re-targeted by hand. H-205 is confirmed with a forced -1; which writer produced -1 in occurrence 2 (`:168`, `:187`, `:266`, a stale-name restore) is still not known.

## 2. Change (HalfBROT, one PR to `develop`)

### 2.1 New pure-logic `FB_FocusTargetInit` (`POUs/`)

A function block, state in its own VAR, no hardware reference, testable in MONETcommonTests like `FB_FocusJogLatch`.

```
FUNCTION_BLOCK FB_FocusTargetInit
VAR_INPUT
	bCalibrated	: BOOL;		// fbAxis.calibrated
	bHoming		: BOOL;		// bHomeAxis
END_VAR
VAR_OUTPUT
	bSync		: BOOL;		// TRUE for one cycle: the target takes the actual position
END_VAR
VAR
	bKnown		: BOOL;		// the target of this calibration is settled
END_VAR

IF NOT bCalibrated THEN
	bKnown := FALSE;
	bSync := FALSE;
ELSIF bKnown THEN
	bSync := FALSE;
ELSE
	bKnown := TRUE;
	bSync := NOT bHoming;	// a homing owns the target (:205), it is settled by that
END_IF
```

`bKnown` is not persistent, so it is FALSE after every PLC start: the restart case syncs, the calibration at the end of
a homing (target owned by `:205`) and a manual re-home on a calibrated axis (already known) do not. A loss of the
calibration (error, `bError` branch `:123`) clears it, the next calibration outside a homing syncs again.

### 2.2 `FB_FocusControl` (one commit)

At the top of the body, before the clamp (`:87`), from the outputs of `fbAxis` in the last scan:

```
fbTargetInit(bCalibrated := fbAxis.calibrated, bHoming := bHomeAxis);
IF fbTargetInit.bSync THEN
	fPosition := fActualPosition;
END_IF
```

Before the clamp and before `fbFollower` (`:102`) so that the follower never sees the stale 0. `fbAxis.calibrated` and
`fActualPosition` are those of the previous scan; in the very first scan `fbAxis` has not run, so `calibrated` is FALSE
and nothing happens, in the second scan `calibrated` is TRUE and `fActualPosition` was written by the first call
(`:222`). The clamp stays after it (MONETcommon#122 range kept, see Q2). Comment at `:201-202` unchanged (the homing still
owns the target).

Not changed: `:205`, the `rtHomed` block, `F_FocusTargetFollowsActual`, the follower, `bEnable` handling.

### 2.3 Release

HalfBROT 0.7.3 or a patch of its own, Q1. No interface change (`FB_FocusTargetInit` is additive). MONETcommon and
MONETS/MONETN pins as for any HalfBROT patch (no `MAIN` change).

### 2.4 Part B (proposed, Q5): the homing target is fixed when the homing starts

Cause (D6): `:205` re-reads `fLastPosition` every cycle of the homing, but `_StartHoming` decided the mode and the reference once.
Proposal: `_StartHoming` stores the reference it passed to `fbAxis.Home()` in a new non-persistent `fHomingReference : LREAL`, and
`:205` becomes `fPosition := fHomingReference;` (cam mode: `fHomingPosition` as now, MC_Direct: the `fLastPosition` of the start).
No later write of `fLastPosition` (hand, `:168`, `:187`, `:266`) can then reach the target; the homing result (`rtHomed`,
the other branch) sets the target after the homing. The pick of the mode and the reference moves into a pure function
`F_FocusHomingReference(fLastPosition, fHomingPosition, bDirect => , fReference => )` (`fLastPosition > 0` as now), tested in
MONETcommonTests: valid value gives direct + value, -1 and 0 give cam + `fHomingPosition`.

Why not "take `fActualPosition` when `fLastPosition` is invalid" (the guard in the first draft of Q3): during an `MC_Direct`
homing the axis is still uncalibrated and `fActualPosition` is the NC placeholder (40.907 mm in the forced D4, 0 after an
activation), so the guard could give 0 and the clamp 0.5 again.

What the unit tests cannot show: the block keeps the reference while `fLastPosition` changes (needs the NC). Hardware: repeat
D6 on the fixed build, expected `fPosition` 36.27 during and after the homing, no 0.5, no queued move.

## 3. Edge cases

| Case | Behaviour |
|---|---|
| PLC restart, calibrated, automatic | target = actual at the second scan, no move (the fix) |
| PLC restart, calibrated, manual/off mode | target = actual (as `:129` already does) |
| PLC restart, calibrated and outside 0.5..91.5 (e.g. 91.99 at the cam) | target clamped (`:87`) and the focus moves into the range, as after a homing; accepted, Q2 |
| PLC restart, not calibrated | unchanged: homing, `:205` owns the target |
| Calibration during a homing (MC_Direct, cam) | `bHoming` TRUE: no sync, and `bKnown` is set so the end of the homing does not sync either |
| Manual re-home of a calibrated axis | already known: no sync; `fbManualHome.fRestoreTarget` restores the target as now |
| Error, reset, calibrated again | `bKnown` cleared by the loss, then sync (same value as `:124`) |
| TwinCAT restart (NC too), calibrated before | the NC restores the calibration and the position itself, calibrated at the start, no homing (D3): same as the PLC restart, the sync applies |
| Activation (download), NC reset | uncalibrated at start, position 0: homing path, `:205` owns the target (forced D4: right target with a valid `fLastPosition`) |
| Online change | state of `FB_FocusTargetInit` is kept |

## 4. Tests

### 4.1 Unit and sim (no NC, no hardware), MONETcommonTests

`FB_FocusTargetInit_Tests` (new, as `FB_FocusJogLatch_Tests`):

- `Calibrated_First_Call_Syncs`: `bCalibrated` TRUE from the first call, `bHoming` FALSE: `bSync` TRUE once, FALSE after.
- `Not_Calibrated_Never_Syncs`.
- `Calibrated_During_Homing_Does_Not_Sync`: `bHoming` TRUE at the rising edge; after the homing ends (`bHoming` FALSE,
  still calibrated) still no sync.
- `Recalibrated_After_Loss_Syncs`: calibrated, lost, calibrated again outside a homing: a second `bSync`.
- `Already_Known_Calibration_Does_Not_Sync_Again`: stays calibrated for many cycles.

`FocusTargetInit_Sim_Tests` (new, in the style of `FocusModeSwitch_Sim_Tests`): `FB_FocusTargetInit`, the clamp and
`BROTLib.FB_MoveFollower` composed as in `FB_FocusControl`, target 0 and actual 36 as at a restart:

- `Restart_Calibrated_No_Start_Move`: with the sync no `bStartMove` and no `bNeedEnable` in any cycle; **without** the
  `bSync` assignment the same test must fail (`bStartMove` TRUE, the regression pin for #115).
- `Restart_Then_New_Target_Moves`: after the sync a target of 40 starts a move (the sync is one-shot).
- `Restart_Outside_Range_Moves_Into_Range` (Q2).

The block `FB_FocusControl` itself is not run in these tests (it needs the NC); the composition test is the closest
without hardware. The call order (sync before the clamp and the follower) is checked on hardware (section 5).

### 4.2 Existing suites

`FocusModeSwitch_Sim_Tests`, `F_FocusTargetFollowsActual_Tests`, `F_FocusTrackStoredPosition_Tests`,
`FB_FocusManualHome_Tests`, `FB_FocusJogLatch_Tests` unchanged and green.

## 5. Hardware verification (MONET/S first, then MONET/N)

Same recording as plan section 5.1 (Scope by symbol address, `fMinPosition` about 35 in the test build while the fix is
not proven). **D1 and D4 above were run on 0.7.2 on 2026-10-25.**

1. **D1 again on the fixed build:** PLC stop and start, calibrated and locked, automatic. `fPosition` equals
   `fActualPosition` from the second scan, no `bStartMove`, `bEnable` stays FALSE (it was TRUE in the first records on
   0.7.2), the brake stays locked, no movement. **Fail:** any motion or `bEnable` TRUE without a request.
2. **D2:** the same from manual mode, then automatic: no move.
3. After the start `focus=40`, then `focus=36`: moves and holds (the sync is one-shot).
4. Restart at a position outside 0.5..91.5 (jog to the edge first): moves into the range (Q2), no error.
5. Regression: power-on from an uncalibrated axis (cam homing and MC_Direct), manual re-home (`bManualHome`), jog in
   manual then automatic (#66).
6. **D6 (H-205):** read `fLastPposition`; start a power-on with a valid `fLastPosition`, and while the MC_Direct homing
   runs write `fLastPosition := -1`. Expected if H-205 holds: `fPosition` -1 then 0.5 in the next cycle after the
   homing. Decides whether a second change is needed (Q3).

## 6. Decisions (Tim, 2026-10-25, all as recommended)

Q1 decided: #115 alone first as a patch, the enable-handling branch after it. Q2 decided: keep the clamp. Q3 decided: wait for D6
before touching `:205`. Q4 decided: test build with `fMinPosition` about 35 until D1 passes on the fixed build. The text of the
questions follows as asked.

1. **Release:** this plan alone as a patch (HalfBROT 0.7.3, the restart risk is a hardware risk) and the enable-handling
   branch after it (0.7.4 or 0.8.0), or both in one 0.7.3? Recommended: separate, #115 first; the two touch different
   blocks of `FB_FocusControl` (top of the body here, `bEnable` and `rtHomed` there) and the other session has agreed not
   to touch the first-scan value.
2. **Calibrated outside 0.5..91.5 after a restart:** keep the clamp (the focus moves into the range, at most about 0.5
   mm from a switch), or skip the clamp for the first sync? Recommended: keep (MONETcommon#122).
3. **Power-on / H-205 (decided: wait for D6; D6 is done, see 1.2 and Q5):** wait for D6 before deciding, or add now a guard in `:205` (no `fLastPosition` below the minimum:
   take `fActualPosition`)? Recommended: wait for D6.
4. **Test build for the hardware runs:** `fMinPosition` about 35 in the MONETS `MAIN` until D1 passes on the fixed build?
   Recommended: yes.

5. **Part B (new after D6; decided: same 0.7.3, as recommended):** fix the homing target when the homing starts (2.4: `fHomingReference` and
   `F_FocusHomingReference`, in the same PR as the first-calibration sync) or leave `:205` alone and only add the sync?
   Recommended: Part B in the same 0.7.3 (the same hardware risk, a small change, unit tests for the pure function).

## 7. Not in this plan

#103 (a new target during a move is ignored), #108 / #110 / #111 / #114 (enable handling, other plan and branch), the
cause of `bEnable` TRUE in the first scan (expected to follow from the missing `bNeedEnable`, checked in 5.1), the stored
position lost by the 0x4260 on `bEnable := FALSE` (`:266`, consequence of the error branch, not a cause), occurrence 2
beyond H-205.
