# Focus after a manual jog: no move back to the old target, stored position tracked (#66)

**Status: implemented in HalfBROT (Q1, Q2, Q3, Q7 as recommended), not compiled or tested; tests and pins follow.** Plan text below as proposed; not run on a PLC or a telescope. Facts below were read
on 2026-10-22 from HalfBROT `develop` @ `1c2ed8b` (0.7.1), BROTLib `develop` @ `0668242`, MONETcommon working tree
@ `6f6293a` (`ci/pin-brotlib-0.10.4`, plcproj 0.9.7), MONETS `develop` @ `b1c8939`, MONETN `develop` @ `84c79ae`.
Line numbers are those files at those commits.

Issues: **A** has no issue yet (proposed title below, not filed); **B** is #66. Related, not fixed here: #108 / #111
(queued `MoveTo()` with a locked drive after a homing), #5 (pendant leaves motion commands set), #85, MONETcommon#118
(`SuspendAutomatic`), MONETcommon#122 (`FB_MoveFollower`), #8 (persistent write), #18 (`MC_Direct`).

**One plan, two parts (A, B).** Both bugs have the same root, a focus moved by hand (pendant jog in manual/off mode)
is not reflected in the focus block's state: A the target `fPosition`, B the stored position `fLastPosition`. They sit
in the same 30 lines of `FB_FocusControl`, share the guards (calibrated, not homing, locked), go into the same HalfBROT
patch and the same MONETcommon/MONETS/MONETN pins, are tested by the same new test suites, and are checked on site in
the same session (jog, then switch to automatic; jog, then restart). Two plans would duplicate the edge-case, release
and checklist sections. Each part is still its own commit, so either one can be reverted alone.

## 1. Problem

### A: the focus drives back to its old target after a manual jog (no issue yet)

Proposed issue title: *FB_FocusControl: after a pendant jog in manual mode the focus drives back to the pre-jog
target when the telescope returns to automatic*.

- `FB_FocusControl.TcPOU:196-197`: the jog goes straight to the NC: `Jog_Forward := bMovePos AND bFocusUnlocked`,
  `Jog_Backwards := bMoveNeg AND bFocusUnlocked`. `fPosition` is not touched.
- The pendant (MONETcommon `FB_MonetPendantControl.TcPOU:114-125`, selection 5) writes `fbFocus.Enable :=
  IbEnableButton`, `MovePos := IbDirectionUp`, `MoveNeg := IbDirectionDown`. Nothing writes `Position`.
- `fPosition` is overwritten only by: the clamp (`:86`), `Stop()` hold (`:110-115`, method `Stop` `:391-396`), error
  (`:118-120`), homing (`:182`), manual re-home end (`:228-230`), plus the callers (MQTT `focus=`, BROTLib
  `FB_Comm_MQTT_Influx.TcPOU:154-170`, rejected outside READY/automatic; park `FB_MonetTelescopeControl.TcPOU:943`).
  None of them runs on a manual jog.
- `FB_MoveFollower` is inactive outside automatic (`FB_FocusControl.TcPOU:102`, `bActive := eTelescopeMode =
  automatic AND NOT bManualUnlock AND fbAxis.calibrated AND NOT bHomeAxis`; `FB_MoveFollower.TcPOU:49-50` only clears
  `bMoving`). In the first automatic cycle it compares the stale `fTarget` with the jogged `fActual`
  (`FB_MoveFollower.TcPOU:58-65`): off by more than `fTolerance` (0.005 mm), so `bNeedEnable` (drive locked) and then
  `bStartMove`; `FB_FocusControl.TcPOU:126-131` enables the drive and calls `fbAxis.MoveTo()` to the pre-jog target.
  Uncommanded motion.
- Nothing re-syncs on the mode switch, verified:
  - BROTLib `FB_BaseAxis.TcPOU`: empty body (`:30`), `Position` is a plain setter (`:250-256`), no mode input.
  - MONETS `MAIN.TcPOU:69-73` derives the mode from `CabinetControl.KeyLocal`; `:103-122` calls `TelescopeControl`
    only in automatic, else `SuspendAutomatic()`; `:151-160` calls `FocusControl` with the mode; `:170-178` calls the
    pendant after the focus and without `eTelescopeMode`. MONETN `MAIN.TcPOU:67-71`, `:148-155` the same.
  - MONETcommon `FB_MonetTelescopeControl.SuspendAutomatic` (`:1176-1222`) stops only azimuth, elevation and
    derotator (`fbStopTx`), never touches `fbFocus`. `Stop()` (`:2208-2217`) calls `fbFocus.Stop()` only on a repeated
    stop in automatic.

### B: the stored position is not tracked in manual mode (#66)

- `FB_FocusControl.TcPOU:123-146`: `fLastPosition := fActualPosition` (`:137-139`, guarded by `NOT ObFocusUnlock AND
  NOT bHomeAxis`) is inside `IF eTelescopeMode = automatic AND NOT bManualUnlock` and `IF fbAxis.calibrated`.
- So a focus jogged in manual/off mode keeps the pre-jog `fLastPosition`. `_WritePersistentPosition` (`:250`,
  method `:303-375`) runs in every mode but finds nothing changed. After the next start (clean or not) `_StartHoming`
  (`:270-279`) homes with `MC_Direct` to that value, the axis is `Calibrated` at the old position and every focus
  position is off by the jog distance. Found in the #8 plan (`2026-10-12-persistent-write-focus-position.md`,
  "Found, separate decision").
- Further writes of `fLastPosition`: manual re-home accept `-1.0` (`:164`), `rtHomed` (`:206-211`, actual position,
  `bSaveRequested`), `bError` `-1.0` (`:242-244`), `_MigrateLastPosition` (`:289-293`).

### Dead variable `bManualUnlock`

`FB_FocusControl.TcPOU:36` is a private `VAR`, not `AT`-mapped, never assigned anywhere in HalfBROT, BROTLib,
MONETcommon, MONETS or MONETN (searched). It is read at `:89`, `:102`, `:123`, `:157` and is always FALSE unless
written online in XAE/ADS. **Decision in this plan: leave it untouched**, and write the new conditions with the same
`automatic AND NOT bManualUnlock` expression, so that a hand-release, if it is ever wired, also re-syncs the target
and does not track while unlocked. Removing it touches the tested `FB_FocusManualHome.Request` signature and
`E_FocusManualHome.ManualUnlock` (`E_FocusManualHome.TcDUT:11`, `FB_FocusManualHome_Tests` `Rejected_With_Manual_Unlock`);
proposed as a separate cleanup issue (Q6).

## 2. Change (one PR to HalfBROT `develop`, two commits)

Only HalfBROT changes: `FB_FocusControl` plus three small pure-logic POUs, in the pattern of `F_FocusHomeRelease` and
`FB_FocusManualHome` (no NC access, the caller does the axis calls). No new `VAR_INPUT`/`VAR_OUTPUT` on
`FB_FocusControl`, no consumer code change.

### 2.1 New pure-logic POUs (HalfBROT `POUs/`)

```
// TRUE while the target of the focus follows its actual position: outside automatic positioning (manual or off mode,
// manual unlock) or while a jog runs, the pendant or a hand moves the focus, so back in automatic the follower
// (FB_MoveFollower) has nothing to do. Only a calibrated focus, never during a homing (the homing owns fPosition).
FUNCTION F_FocusTargetFollowsActual : BOOL
VAR_INPUT
	bAutomatic		: BOOL;		// eTelescopeMode = automatic
	bManualUnlock	: BOOL;
	bJogging		: BOOL;		// FB_FocusJogLatch.bJogging
	bCalibrated		: BOOL;		// fbAxis.calibrated
	bHoming			: BOOL;		// bHomeAxis
END_VAR
F_FocusTargetFollowsActual := (NOT bAutomatic OR bManualUnlock OR bJogging) AND bCalibrated AND NOT bHoming;
```

```
// TRUE when the stored focus position (fLastPosition) takes the actual position: calibrated, at rest with the brake
// locked (drive off) and no homing running. Independent of the telescope mode (#66).
FUNCTION F_FocusTrackStoredPosition : BOOL
VAR_INPUT
	bCalibrated		: BOOL;		// fbAxis.calibrated
	bLocked			: BOOL;		// NOT ObFocusUnlock
	bHoming			: BOOL;		// bHomeAxis
END_VAR
F_FocusTrackStoredPosition := bCalibrated AND bLocked AND NOT bHoming;
```

```
// A jog is running from the jog command until the axis is at standstill again (the ramp down after the button is
// released is part of it). Call once per cycle before the follower.
FUNCTION_BLOCK FB_FocusJogLatch
VAR_INPUT
	bJogCommand		: BOOL;		// (bMovePos OR bMoveNeg) AND bFocusUnlocked, what goes to Jog_Forward/Jog_Backwards
	bStandStill		: BOOL;		// fbAxis.StandStill from its last call
END_VAR
VAR_OUTPUT
	bJogging		: BOOL;
END_VAR
bJogging := bJogCommand OR (bJogging AND NOT bStandStill);
```

`FB_FocusJogLatch` and the `bJogging` term are the recommended answer to the "mode switch during a jog" edge case
(section 3); if Tim declines Q2, drop the FB and pass `bJogging := FALSE`.

### 2.2 Commit A: target follows the actual position outside automatic

In `FB_FocusControl` `VAR`: `fbJogLatch : FB_FocusJogLatch;	// jog running, see F_FocusTargetFollowsActual`.

Body, after the error block (`:117-120`), before the automatic block (`:122`):

```
// a jog runs until standstill (the inputs are what fbAxis reported in its last call)
fbJogLatch(bJogCommand := (bMovePos OR bMoveNeg) AND bFocusUnlocked, bStandStill := fbAxis.StandStill);

// outside automatic positioning or while jogging the focus is moved by hand: the target follows it, so the follower
// does not drive back to the old target when the mode returns to automatic
IF F_FocusTargetFollowsActual(
	bAutomatic		:= eTelescopeMode = E_TelescopeMode.automatic,
	bManualUnlock	:= bManualUnlock,
	bJogging		:= fbJogLatch.bJogging,
	bCalibrated		:= fbAxis.calibrated,
	bHoming			:= bHomeAxis) THEN
	fPosition := fActualPosition;
END_IF
```

The jog latch must run before the follower call, so move the `fbFollower(...)` call (`:96-107`) below this block
(it only reads `fPosition`, `fActualPosition` and `fbAxis` outputs of the last cycle, so the move does not change its
inputs except `fTarget`, which is the point), and add `AND NOT fbJogLatch.bJogging` to its `bActive` (`:102`):

```
	bActive		:= eTelescopeMode = E_TelescopeMode.automatic AND NOT bManualUnlock AND NOT fbJogLatch.bJogging
					AND fbAxis.calibrated AND NOT bHomeAxis,
```

Alternative placement without moving the follower: put the jog latch and the sync between `:94` and `:96`. Same
effect; pick whichever reads better. The `Stop()` hold (`:109-115`) and the error sync (`:117-120`) stay as they are.

### 2.3 Commit B: track the stored position in every mode (#66)

Remove `:136-139` from the automatic branch. Insert right after the automatic block's `END_IF` (`:146`), before the
manual re-home block (`:148`):

```
// store the actual position when the focus is calibrated, locked and not homing, in every mode: a focus jogged in
// manual mode must not be restored to its pre-jog position by MC_Direct after a restart (#66)
IF F_FocusTrackStoredPosition(bCalibrated := fbAxis.calibrated, bLocked := NOT ObFocusUnlock, bHoming := bHomeAxis) THEN
	fLastPosition := fActualPosition;
END_IF
```

The order of the writes within a cycle stays: tracking, then manual re-home accept (`-1.0`, sets `bHomeAxis`), then
`rtHomed` (actual), then `bError` (`-1.0`), then `_WritePersistentPosition`. So `-1.0` still wins on an error or a
re-home in the same cycle.

**Optional B2 (Q3, recommended):** a jog of an uncalibrated focus invalidates the stored position. At power-on in
local mode the focus is uncalibrated (`fLastPosition` restored), and the pendant can jog it (`bFocusUnlocked` is the
only jog condition); the next automatic homing then calibrates with `MC_Direct` to a value the focus has left. After
the B block:

```
// a jog before calibration moves the focus away from the stored position: home on the cam next time (#66)
IF NOT fbAxis.calibrated AND NOT bHomeAxis AND fbJogLatch.bJogging THEN
	fLastPosition := -1.0;
END_IF
```

`_WritePersistentPosition` writes the `-1.0` through its validity check (`:324`).

### 2.4 Docs

README focus/calibration paragraph: the target follows the actual position outside automatic mode and during a jog;
the stored position is tracked in every mode. Plans index entry (done with this plan). Status line here on
implementation.

## 3. Edge cases

| Case | Behaviour with the change | Why |
|---|---|---|
| Manual/off jog, back to automatic | no move: `fTarget = fActual` in the first automatic cycle | A every manual cycle; `fActualPosition` at most one cycle old, the jog has ended (latch) |
| Mode switch **during** a jog (key to remote, jog button still held) | follower stays inactive while `bJogging` (also during the ramp down); target follows; after standstill no move | jog latch (Q2). Without it: the follower starts `MoveTo()` against the running jog, and the ramp-down distance can exceed 0.005 mm |
| Pendant jog **in automatic** (the pendant is not mode-gated: `FB_MonetPendantControl.TcPOU:90-93` empty, MAIN does not pass the mode) | the jog wins, the target follows; afterwards the focus stays | jog latch; today the follower drives back at once. Changes automatic behaviour, hence Q2 |
| Jogged outside `fMinPosition..fMaxPosition` (MONETS/N 0.5..91.5; switches at 0 and 92.02) | in manual the target follows (unclamped this cycle, clamped at `:86` next cycle). Back in automatic the clamp makes it 91.5 / 0.5 and the focus moves **into the range** (at most to the edge) | MONETcommon#122 range kept. Q1: accept, or skip the clamp while `fPosition = fActualPosition` |
| Error (`bError`) | unchanged: `:118-120` syncs when calibrated; follower `bError` blocks starts; B: tracking writes, then `:242-244` sets `-1.0` in the same cycle (`ObFocusUnlock` is FALSE on error) | order of the writes |
| Error reset while calibrated | B: tracking resumes and overwrites the `-1.0` | **pre-existing** in automatic, now also in manual; Q4 |
| Homing (`bHomeAxis`), automatic or started in automatic and switched to manual | no sync, no tracking while `bHomeAxis`; `:182` owns `fPosition`; after the end in manual the target follows the actual position (at the cam, ~91.99) | guard `bHoming`. Back in automatic: clamp to 91.5, see #111 / #108 (not fixed here) |
| Manual re-home (`bManualHome`) | requires automatic (`FB_FocusManualHome.TcPOU:62-63`), unchanged. If the mode goes to manual while it runs, `fRestoreTarget` (`:228-230`) is set once and overwritten by A next cycle; back in automatic the focus stays where the homing ended (subject to the clamp) | manual mode owns the focus; acceptable, documented |
| `Stop()` | unchanged in automatic. In manual `bHoldOnStop` writes the same value A writes. `FB_MoveFollower.bStopping` clears when A moves the target by more than `fTolerance` (`FB_MoveFollower.TcPOU:45-47`), harmless, target = actual | |
| Automatic move running, switch to manual | follower inactive, the NC move keeps running to the old target (`SuspendAutomatic` does not stop the focus); A follows; back in automatic no move | pre-existing that the move continues; Q5 |
| Brake / unlock delay | B tracks only while `NOT ObFocusUnlock`: in manual that is as soon as the pendant's enable button is released (`Enable := IbEnableButton`); `ptBrakeDelay` (`:133-135`) applies only in automatic. A focus left enabled in manual (pendant on another selection, #5) is not tracked until locked | pre-existing; Q5 |
| Uncalibrated jog | A: nothing (not calibrated). B: nothing; B2 sets `-1.0` | Q3 |
| Persistent write | after a manual jog `fLastPosition` changes once the brake locks; `_WritePersistentPosition` (`:326`: changed, locked, not homing, not `Busy`, 60 s rate limit) writes it at most 60 s later. No new trigger, no extra write rate (a jog session is one write after the last lock plus at most one per minute) | |
| `rtHomed` | unchanged; sets `bEnable := FALSE` and `fLastPosition := fActualPosition`. In manual, tracking writes the same value afterwards | |
| MQTT `focus=` in manual | already rejected (`FB_Comm_MQTT_Influx.TcPOU:155-161`); A would overwrite it anyway, so no queued target replays on the switch back | |
| Telemetry | `TARGETPOS` = `CURRPOS`, `TARGETDISTANCE` 0 while manual (`SendTelemetry :506-507`) | side effect, improvement |
| `bManualUnlock` set online | target follows, no tracking while unlocked, as manual mode | same expression |
| Park restarting on the switch back (`bPark` kept by `SuspendAutomatic`, park sets `focusPark`, `FB_MonetTelescopeControl.TcPOU:943`) | the focus moves to `focusPark`: commanded, not the bug | mention in the on-site test |

## 4. Unit tests (pure logic, no NC, no hardware)

HalfBROT still has no test project (oil plan D5), so the suites go into **MONETcommonTests** (references HalfBROT
and BROTLib), next to `F_FocusHomeRelease_Tests` and `FB_FocusManualHome_Tests`, and are registered in
`MONETcommonTests/.../POUs/MAIN.TcPOU` like `:19-20`. `FB_FocusMock` skips the real body (no `SUPER^()`), so the
wiring in `FB_FocusControl` itself is not testable there; that is the on-site check.

`F_FocusTargetFollowsActual_Tests`:
- `Manual_Calibrated_Follows`, `Off_Calibrated_Follows` (bAutomatic FALSE) TRUE.
- `Automatic_Idle_Does_Not_Follow` FALSE.
- `Automatic_Jogging_Follows` TRUE; `ManualUnlock_Follows` TRUE.
- `Not_Calibrated_Never` FALSE for every mode/jog combination.
- `Homing_Never` FALSE for every mode/jog combination.

`F_FocusTrackStoredPosition_Tests`:
- `Calibrated_Locked_Idle_Tracks` TRUE.
- `Unlocked_Does_Not_Track`, `Homing_Does_Not_Track`, `Not_Calibrated_Does_Not_Track` FALSE.
- `No_Mode_Input`: documents #66 (signature has no mode; a comment, one assert as the first).

`FB_FocusJogLatch_Tests`:
- `Command_Sets`, `Holds_Until_StandStill` (command off, StandStill FALSE for 3 calls, then TRUE), `Idle_Clear`,
  `StandStill_With_Command_Stays` (command wins).

`FocusModeSwitch_Sim_Tests` (sequence, like `StopLoop_Sim_Tests`; `BROTLib.FB_MoveFollower` + the functions, a local
`fTarget`/`fActual` model of the focus body's order):
- `Jog_Then_Automatic_No_Start`: automatic at 36.0 on target; manual for 10 cycles with `fActual` stepping to 37.0
  and the sync applied; automatic for 20 cycles with ready TRUE/FALSE: `bStartMove` and `bNeedEnable` never TRUE.
- `Jog_Then_Automatic_Without_Sync_Starts`: same without the sync: `bNeedEnable`/`bStartMove` in the first automatic
  cycle (documents the bug, guards against the test passing for the wrong reason).
- `Switch_During_Jog_No_Start`: automatic during the jog and 5 ramp cycles (`bStandStill` FALSE, `fActual` still
  changing): no start; after standstill none.
- `New_Target_After_Jog_Moves`: after the switch a new target 40.0 starts exactly one move (the follower still works).
- `Out_Of_Range_Moves_To_Edge` (only if Q1 is accepted as is): jog to 91.8, clamp to 91.5 in automatic: one move.

Running: by CI on the MONETcommon PR (TcUnit through the release/test workflow) or by Tim in XAE. **Not on this
machine without asking first (it is the CI runner).** Report: the TcUnit summary line with 0 failed tests.

## 5. Hardware verification (on site, MONET/S first, then MONET/N)

Someone at the cabinet and the pendant. Record (ADS watch or scope, 10 Hz): `FocusControl.fPosition`,
`fActualPosition`, `fLastPosition`, `fSavedPosition`, `bEnable`, `ObFocusUnlock`, `fbFollower.bActive/bMoving/
bStartMove/bNeedEnable`, `fbJogLatch.bJogging`, `fbAxis.MovePending`, `eTelescopeMode`; MQTT `TARGETPOS`/`CURRPOS`.
Telescope not parking (no park restarting on the switch back, else the focus moves to `focusPark` by design).

1. Baseline: automatic, focus calibrated, target 36.0 reached, brake locked (after `ptBrakeDelay`: 30 s MONET/S,
   4 s MONET/N default).
2. Key local (manual), pendant selection 5, enable key, jog up about 1 mm, release. `fPosition` follows
   `fActualPosition` during and after the jog; `TARGETPOS` = `CURRPOS`.
3. Key remote (automatic). For 60 s: no motion, `bStartMove`/`bNeedEnable` never TRUE, `bEnable` stays FALSE,
   `MovePending` FALSE, `fActualPosition` unchanged. **Fail:** any move back towards 36.0.
4. Then MQTT `focus=36`: exactly one move to 36.0, brake locks after `ptBrakeDelay`.
5. Mode switch during a jog: manual, hold jog, turn the key to remote, release the jog. After standstill no move
   back; note the ramp-down distance.
6. (Q2) Pendant jog in automatic (enable key on): the focus jogs and stays; nothing drives it back.
7. (Q1) Jog to about 91.8 mm in manual, key remote: per decision, either one move to 91.5 or none.
8. Stored position (#66): manual, jog about 1 mm, release the enable button. `fLastPosition = fActualPosition` within
   a cycle of the brake locking; within 60 s `fSavedPosition = fLastPosition`, `Port_851.bootdata` timestamp updated,
   no persistent-write error event.
9. Restart TwinCAT cleanly (Run mode restart) and, as #59 item 5, once with a hard reset after step 8's write. After
   the start, key remote: the startup homing uses `MC_Direct` (`eHomingMode`), the focus becomes `Calibrated`
   **without moving**, `fActualPosition` equals the jogged value of step 8. Cross-check against an independent
   reference (cam: optional `bManualHome` re-home afterwards, the position after it should agree within the cam
   repeatability). **Fail:** the pre-jog value comes back.
10. (B2) After a restart in local mode, jog the uncalibrated focus: `fLastPosition = -1.0`; key remote: cam homing
    (`MC_DefaultHoming`), not `MC_Direct`.
11. Regression: a normal automatic night sequence (several focus targets), `bManualHome` once (#104 path, expect the
    known #108 behaviour), `Stop()` during a focus move.

## 6. Release and downstream pins

- **HalfBROT 0.7.2** (patch: behaviour fix, three additive public POUs, no interface change on `FB_FocusControl`).
  A minor (0.8.0) would also be the release where `fLastPposition`/`_MigrateLastPosition` are due for removal (#49),
  which should not ride along with a behaviour fix; hence patch (Q7). Bump `ProjectVersion` in
  `HalfBROT/HalfBROT/HalfBROT.plcproj:25`, release by the release-via-PR flow; CI log ends `Build complete -- 0
  errors, 0 warnings`.
- **MONETcommon**: pin `HalfBROT, 0.7.2 (BROT)` in `MONETcommon/MONETcommon/MONETcommon.plcproj:132` and
  `MONETcommonTests/.../MONETcommonTests.plcproj:174`, add the suites of section 4, release the next MONETcommon
  patch (MONETS/MONETN `develop` pin 0.9.8 today, so 0.9.9 or whatever is next). No MONETcommon code change.
- **MONETS / MONETN**: pin HalfBROT 0.7.2 and the new MONETcommon in `MONETSRuntime.plcproj:110` /
  `MONETNRuntime.plcproj:104` (and the MONETcommon `DefaultResolution`), as in MONETS `e27c71d` "Pin HalfBROT 0.7.1
  and MONETcommon 0.9.7". No `MAIN` change. XAE build by Tim; activation only with the on-site check.
- IAG50cm, AstroBROT, MONETRoof do not reference HalfBROT: unaffected. IAG50cm has its own `FB_FocusControl`; whether
  it has the same two bugs was not checked (out of scope).
- Persistent data: no new persistent variable; the restore by name is unaffected.

## 7. Open questions (for Tim)

1. **Q1 out-of-range jog:** back in automatic a focus jogged beyond `fMinPosition..fMaxPosition` moves into the range
   (at most to the edge). Accept (recommended: the range keeps targets 0.5 mm off the switches, MONETcommon#122), or
   skip the clamp while the target equals the actual position?
2. **Q2 jog latch:** also hold the follower off and follow the target during any jog, including a pendant jog in
   automatic (recommended; covers the switch during a jog and the ramp down). Or only the mode term?
3. **Q3 B2:** a jog before calibration sets `fLastPosition := -1.0` (cam homing next time). Recommended.
4. **Q4:** the `-1.0` after an axis error is overwritten by the tracking once the error is reset while still
   calibrated (pre-existing in automatic). Latch it until the next homing?
5. **Q5:** should `SuspendAutomatic` (MONETcommon#118) also stop a running focus move, and should the brake-delay
   disable (`:133-135`) apply outside automatic? Both pre-existing, not changed here.
6. **Q6 `bManualUnlock`:** remove (with `E_FocusManualHome.ManualUnlock` and the `Request` input, tests updated) or
   make it a real input for a hand release? Proposed as its own issue.
7. **Q7:** patch 0.7.2 (recommended) or minor?
8. **Q8:** file the issue for A (title in section 1) and link it to #66, #108, #111?
