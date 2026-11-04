# Focus: stop during a homing ends in 0x4260 and a stuck re-home (#114 H2)

**Status 2026-11-04: E (PR #141), BROTLib F (#153) and the standstill fix (PR #145) are on `develop`; #141 and #153 are released (HalfBROT 0.11.1, BROTLib 0.17.1, in MONETS/MONETN 1.6.4), #145 is not released yet (the next release waits for more changes). On hardware (MONET/S, 1.6.4): step 4 (stop during `bManualHome`, 2 s) passed on the PLC variables; step 4c (stop during the power-on cam homing) failed and is fixed by #145, which has not been run on a PLC. G is on branch `feat/focus-error-lock-114` (hardening, not needed for #114, Q1 open). Section 7a lists what passed, failed and is open.**

Follows `2026-10-25-focus-enable-handling-108-110-111-114.md` (PR #117), section 1.5 (H2), section 5 step 4 and Q4.
Inputs: #114 (comments 2026-10-25, 2026-10-30, 2026-11-03), PR #117 comment of 2026-11-03 08:24, #138.

## 1. Observations

| Run | Data |
|---|---|
| 0.7.2, 2026-10-25 (#114) | stop 61 s and 1.8 s after `bManualHome`: `bError` TRUE, `bEnable` TRUE, `bHomeAxis` FALSE, `bBlockAutoHome` FALSE, 0x4260 |
| 0.10.2, 2026-10-30 14:44 (#114, ADS ~15 ms) | `stop=1` 2.17 s after `bManualHome`. **+44 ms `nErrorID` 19210 (0x4B0A)**, +144 ms `bError` with 16992 (0x4260), `ObFocusUnlock`, `bHomeAxis`, `Busy`, `Ready` FALSE. 65 s later `bEnable` TRUE (never dropped), `bRunning` and `HomeActive` TRUE. `reset=1` cleared `bRunning`/`HomeActive` and **only then** set `bBlockAutoHome` |
| 0.10.2, 2026-11-03 10:23 (PR #117, step 4 run 1) | `stop=1` 2.1 s after `bManualHome`; +154 ms `bFocusUnlocked` FALSE, `fbDriveEnable.bEnableOn` FALSE, `fbAxis.Error` 0x4260, `bHomeAxis` FALSE, `HydraulicsControl.bEnable` FALSE, `TelescopeControl.bError`, `READY_STATE` -1. +60 s `bRunning`, `HomeActive` TRUE, `bAborted`, `bBlockAutoHome` FALSE. `reset=1` clears it |

## 2. Root cause

### 2.1 The chain, cycle by cycle

| Step | Where | What happens | Status |
|---|---|---|---|
| 1 | MONETcommon `FB_MonetTelescopeControl.TcPOU:2832-2839`, `:714-737`, `:2435-2441` | `stop=1`: `bStop`, the stop branch, `_StopTelescope` calls `fbFocus.Stop()` once. The telescope does **not** write `fbFocus.Enable` or `HomeAxis` (only the power-on abort `:1384` does, and no power-on was running) | code |
| 2 | HalfBROT `FB_FocusControl.TcPOU:448-455` | `Stop()`: follower stop, `CancelMove()`, `CancelHome()` (only a pending one), and because the homing axis is `Ready AND NOT StandStill`: `fbAxis.Stop()` | code |
| 3 | BROTLib `FB_Axis.TcPOU:445-449`, `:291-302` | `StopCommand` requests MC_Stop; `StopAllowed` holds while `Homing`: MC_Stop is executed on the axis that runs MC_Home | code |
| 4 | NC / Tc2_MC2 | **a function-block error 0x4B0A (19210)** about 40 ms after the stop (10-30 trace). Range 0x4Bxx = Tc2_MC2 library errors, not an NC axis error. Which block (`Axis_Home` or `Axis_Stop`) and its meaning are not known: the ID is folded into `fbAxis.ErrorID` (`FB_Axis.TcPOU:317-353`, last writer wins). | **hypothesis H2a**: MC_Home reports an error (not `CommandAborted`) when MC_Stop interrupts the cam homing |
| 5 | `FB_FocusControl.TcPOU:262`, `:192`, `:195`, `:251` | `bError` TRUE. Next cycle: `ObFocusUnlock := (bManualUnlock OR bEnable) AND NOT bError` is FALSE, `tonFocusDelay.Q` (`bFocusUnlocked`) drops at once, `fbAxis(enable := FALSE)`: MC_Power drops the controller enable while the NC still runs the homing/stop | code (the bError gate) |
| 6 | NC | 0x4260 "lost its controller enable while executing a command" (as in #110). `Axis_Status.ErrorID` is written last (`FB_Axis.TcPOU:351-353`), so 0x4260 overwrites 0x4B0A in `ErrorID` | inferred, high confidence (timing 44 ms -> 144 ms, `Ready` FALSE) |
| 7 | `FB_FocusDriveEnable.TcPOU:36` | `bEnableOn := bAutomatic AND NOT bError AND bWanted`: FALSE because of `bError`. That is the observed `bEnableOn` FALSE | code |
| 8 | `FB_FocusDriveEnable.TcPOU:30`, `:37` | `bEnable` is **not** written FALSE: `bHomeWanted` (`FB_FocusControl.TcPOU:182`) holds through `fbAxis.HomeActive` in every cycle, so `bEnableOff` (`NOT bWanted AND ...`) is never TRUE. Matches `bEnable` TRUE "never dropped" (10-30). The unlock is removed by the `bError` gate only | code + 10-30 data |
| 9 | `F_FocusHomeRelease.TcPOU:16`, `FB_FocusControl.TcPOU:281-283` | `bError` releases `bHomeAxis` (direct write, no `CancelHome`). So `bHomeAxis` FALSE and the far-switch bypass release (`:248`) are **consequences**, not the cause (the axis is at 36-37 mm, the far switch is at 92) | code |
| 10 | BROTLib `FB_Axis.TcPOU:270`, `:279`; `FB_AxisCommand.TcPOU:9`, `:47-48` | `HomeActive := HomeCommand.Execute OR Axis_Home.Busy`. `FB_AxisCommand` releases `Execute` only on `Done` or `Aborted`; **on an error it keeps `Execute` TRUE until the next request or `Reset`** (documented). So after an MC_Home error `HomeActive` stays TRUE until `reset=1` | code (H2a explains it; if the error is MC_Stop's, MC_Home also failed, with 0x4260) |
| 11 | `FB_FocusManualHome.TcPOU:32-44` | a running re-home ends only when `NOT bHomeActive AND NOT bHomePending` (`:35`). `bError` only sets `bAbortSeen` (`:34`, via `bAbortPulse`, `FB_FocusControl.TcPOU:290`). With `HomeActive` stuck the block never ends: `bRunning` TRUE, no `bAborted`, `bBlockAutoHome` FALSE. `reset=1` releases `Execute` (`FB_AxisCommand.TcPOU:36-39`), `HomeActive` drops, the block ends aborted and sets `bBlockAutoHome` (exactly the 10-30 recovery) | code + 10-30 data |
| 12 | MONETcommon `:391-397`, `:655-664`, `:1240-1248`, `:1307-1309` | `fbFocus.Error` is part of `bHwError`: `bError`, `READY_STATE` -1, emergency park (queued behind the stop), park in place, `fbHydraulics.Enable := FALSE` once the brake is closed. `HydraulicsControl.bEnable` FALSE is a consequence | code; order to confirm in the trace |

### 2.2 In one sentence

A Tc2_MC2 error (0x4B0A, most likely MC_Home answering the MC_Stop with an error instead of `CommandAborted`) sets `bError`;
the `bError` gate on the brake output (`FB_FocusControl.TcPOU:192`) then cuts the drive enable while the NC still runs the
homing/stop, which the NC reports as 0x4260; and because BROTLib keeps `Execute` of an errored MC_Home, `HomeActive` stays
TRUE until a reset, so `FB_FocusManualHome` (which only ends on `NOT HomeActive`) never aborts and never blocks the
automatic homing. PR #117 removed H1 correctly; this is H2, and it has three parts: the trigger (step 4), the cascade
(step 5) and the stuck follow-up (steps 10-11).

### 2.3 Confirmed vs. to confirm

Confirmed from the code (with the logged data): what clears `ObFocusUnlock` (the `bError` gate, not `bEnable`); why
`bEnableOn` is FALSE; that `bEnable` is never written FALSE; that `bHomeAxis` FALSE comes from `bError`; why `HomeActive`,
`bRunning` stay TRUE and why only `reset=1` ends them; that the hydraulics and `READY_STATE` follow the focus error.

Needs a task-cycle Scope trace (section 6, D7): which block reports 0x4B0A and what it means (H2a vs. H2b "MC_Stop is
refused during the cam homing"); that the PLC enable drop precedes 0x4260 by one cycle; whether `Axis_Home.Busy` or only
`HomeCommand.Execute` holds `HomeActive`; whether `fbAxis.Ready` (MC_Power `Status`) drops at once on an NC axis error.
Available now without a run: the TwinCAT event log (NC messages with `StateDWord`, `ActVelo`) at 2026-11-03 10:23:00.6-10:23:01
and 2026-10-30 14:44:30.5, and the Beckhoff Tc2_MC2 error table entry for 0x4B0A.

## 3. Same flaw elsewhere

| Path | Verdict | Why |
|---|---|---|
| Stop during a normal follower move (#114 regression, step 8) | no H1/H2 expected | `fbFollower.Stop()` ends `bMoving` (`FB_MoveFollower.TcPOU:134-138`), nothing requests a lock, `tonRest` waits for `StandStill`; MC_Stop on MC_MoveAbsolute aborts it normally (`CommandAborted`). **The cascade remains**: any MC error during a move (e.g. an MC_Stop error) cuts the enable at once via `:192` and becomes 0x4260. Not observed |
| Stop during a pending re-home (`HomePending`, drive not yet ready) | fine | `CancelHome()` (`:450-452`), no MC_Stop; `FB_FocusManualHome` ends aborted (`Abort_Before_Start` test) |
| Stop during an MC_Direct homing | same path, small window | the homing lasts ~160 ms (#138); a stop in it sends MC_Stop to a homing axis (H2a/H2b alike); no motion, so 0x4260 is less likely. Low risk, not worth a test on site |
| Stop during the automatic cam homing (power-on) | **same trigger and cascade**; follow-up differs | telescope stop aborts the power-on (`:1372-1384`: `fbFocus.HomeAxis := FALSE`) and calls `fbFocus.Stop()`: MC_Stop on the homing, 0x4B0A, 0x4260. No `fbManualHome`, but `HomeActive` sticks until reset; no lock request exists, so **after the reset `bEnable` is still TRUE**, the brake opens again, and `:163` restarts the automatic homing by itself (automatic, uncalibrated, `Ready`, not blocked): the stop is undone |
| Stop during the automatic homing started by `:163` (not the power-on), if MC_Home ends cleanly | **gap, code only** | nothing clears `bHomeAxis` (`F_FocusHomeRelease` needs `calibrated` or `bError`), so `bHomeWanted` stays TRUE: drive enabled and brake open with no homing running, for ever. If something clears it, `:163` re-homes at once |

## 4. Proposed change

Three commits, each revertable alone. E is code-confirmed and needed whatever the trace shows; F depends on D7; G is a
safety decision.

### Commit E (HalfBROT, now): an error ends a running homing, the lock request survives it

`FB_FocusManualHome`: new `VAR_INPUT bError : BOOL; // the axis reports an error: MC_Home is over even if HomeActive stays
TRUE (BROTLib keeps Execute of a failed command until Reset)`. Body `:35`:

```
	IF (NOT bHomeActive AND NOT bHomePending) OR bError THEN
```

The rest unchanged: `bAbortSeen` is TRUE (bError is in `bAbortPulse`), so the end is `bAborted`, `bBlockAutoHome := NOT
bCalibrated`. Caller `FB_FocusControl.TcPOU:286-290`: `bError := bError`. The abort branch (`:296-303`) then runs in the
first error cycle: `HomeAxis := FALSE`, `fbDriveEnable.RequestLock()`, the log line.

`FB_FocusDriveEnable` `:30`: a move or homing that ended in an error is not wanted any more (MC blocks in error stay
`Execute`/`HomeActive` until Reset):

```
bWanted := (bMoveWanted OR bHomeWanted) AND NOT bError;
```

Without this the `RequestLock()` of the abort is cleared in the next call (`:32-34`) by the stuck `HomeActive`, and after
`reset=1` the drive stays enabled with the brake open on an uncalibrated focus. `bEnableOn` is unchanged (already `NOT
bError`). With `:38` (`bLockPending AND (bAtRest OR bError)`) the lock completes in the error cycle, as before the change
of G.

Stop of the automatic homing (section 3, last two rows): `Stop()` remembers a stopped homing (`bHomeStopped := bHomeAxis OR
fbAxis.HomeActive OR fbAxis.HomePending`); the body ends it once the axis reports no homing (`NOT HomeActive AND NOT
HomePending`) or `bError`: `HomeAxis := FALSE`, `RequestLock()`, and block the automatic homing (`fbManualHome.Block()`, a new
one-line method; `ClearBlock()` stays the way out, also through the power-on's `HomeAxis`, `:210-212`). Clearing `bHomeAxis`
only at that end keeps the far-switch bypass (`:248`) until the axis is at rest, which also covers old Commit D for a
stop near the cam. **Needs Q2.** If Tim prefers, this part ships alone later; the manual re-home does not need it.

### Commit F (BROTLib, after D7 confirms H2a): a homing stopped by `Stop()` ends as aborted, not as an error

`FB_Axis`: remember that `Stop()` ran while `HomeActive` (`HomeStopped`). If `Axis_Home.Error` comes while `HomeStopped`,
treat it as the abort it is: `HomeCommand(... Aborted := Axis_Home.CommandAborted OR (HomeStopped AND Axis_Home.Error))`
(releases `Execute`, so `HomeActive` drops), `CommandAborted` pulses, and that `Axis_Home.Error` is not part of `Error`
(`:361-370`) nor of `ErrorID`. `HomeStopped` clears on the next `Home()` or `Reset`. Only the exact 0x4B0A ID from D7 should
be swallowed; any other MC_Home error stays an error. With F the stop takes the H1 path PR #117 already handles: no error,
`bEnable` TRUE until `StandStill`, then the lock, `bBlockAutoHome` TRUE.

If D7 shows H2b (MC_Stop refused while homing, `Axis_Stop.Error`): F becomes "stop a homing by the means the NC accepts"
(per the Tc2_MC2 docs for the homing state; candidates MC_Halt or MC_Reset after the drive is at rest), decided with the
trace. E and G stay as they are.

### Commit G (HalfBROT, decision Q1): the error lock waits until the axis can be locked

Today any error, also a pure function-block error with the NC still controlling a moving axis, cuts the enable in the next
cycle (`:192`) and creates the 0x4260 that hides the first error. Proposal: `FB_FocusDriveEnable` gets `bDriveReady`
(`fbAxis.Ready`, last call) and `tErrorHold` (constant, e.g. `T#1S`; the focus decelerates in ms at 0.2 mm/s) and a latched
output:

```
tonErrorHold(IN := bError, PT := tErrorHold);
bErrorLock := bError AND (bErrorLock OR bAtRest OR NOT bDriveReady OR tonErrorHold.Q);
```

`:38` becomes `(bLockPending AND (bAtRest OR bErrorLock))`, and `FB_FocusControl.TcPOU:192`:

```
ObFocusUnlock := (bManualUnlock OR bEnable) AND NOT fbDriveEnable.bErrorLock;
```

An NC axis error (drive fault, following error) drops MC_Power `Status`, so the brake still locks at once (verify in D7);
an MC error with the axis still under control keeps the brake open until rest, at most `tErrorHold`. `bErrorLock` is
FALSE without an error, so normal operation is unchanged.

### Docs

README focus paragraph (stop during a homing: aborted, uncalibrated, automatic homing blocked until `bManualHome` or a
power-on). Plans index entry. Comment on #114 with the chain of 2.1.

## 5. Alternatives

| Alternative | Verdict |
|---|---|
| Old Commit D (hold the far-switch bypass until rest) | not the cause here: `bHomeAxis` falls only after `bError` and the axis is far from the cam. Its effect is folded into E (bypass held until the stopped homing has ended) |
| Do not stop a running homing (only `CancelHome`), let it finish | rejected: 5 min cam homing, a stop must stop |
| Remove the `bError` gate from `ObFocusUnlock` | rejected: a drive fault must lock the brake. G narrows it instead |
| Ignore 0x4260 after a stop in HalfBROT | rejected: masks real lost-enable faults (#110) |
| `FB_AxisCommand` releases `Execute` on every error | rejected for BROTLib in general: the MC block would drop its `Error` with `Execute`, the error disappears from `FB_Axis.Error` for all axes. F limits it to a homing the caller stopped |
| Only E (no F, no G) | removes the stuck state and the open brake after reset, but the stop still raises 0x4B0A, 0x4260 and an emergency park: step 4 still fails |

**Recommendation:** E now (pure logic, independent of the trace, fixes the stuck `bRunning`/`HomeActive` follow-up and the
lock after reset); run D7 on the current build; then F (the trigger; this is what makes step 4 pass) and G (the cascade,
for every other MC error during a move or homing) together in one release. Confidence: high for the cascade and the stuck
follow-up (code plus the 10-30 data), medium for H2a as the trigger (the 0x4B0A source is not recorded).

## 6. Tests

Pure logic, in MONETcommonTests next to the existing suites (MONETcommon branch `test/focus-drive-enable`, not pushed,
holds `FB_FocusDriveEnable_Tests` and `FocusEnable_Sim_Tests`). Run by CI or by Tim; **not on this machine without asking
(it is the CI runner).**

`FB_FocusManualHome_Tests` (new input `bError` in every call):
- `Abort_By_Error_While_HomeActive_Stays`: running, `bHomeActive` TRUE, then `bError` TRUE with `bHomeActive` still TRUE: `bAborted` once, `bRunning` FALSE, `bBlockAutoHome` TRUE (uncalibrated). Fails on 0.10.2.
- `Abort_By_Error_Calibrated_No_Block`; `Error_Ends_Once` (10 more calls with error and `HomeActive`: no second pulse).
- `Abort_By_Error` (existing): keep, fix its comment; its model (HomeActive drops after the error) is not what BROTLib does.
- `Request_Rejected_While_Error_Then_Accepted_After_Reset_While_Blocked`.

`FB_FocusDriveEnable_Tests`:
- `Error_Ends_Wanted`: `bHomeWanted` TRUE, `bError` TRUE: `bEnableOn` FALSE, `RequestLock()` not cleared.
- `Lock_Request_Survives_Stuck_HomeActive`: request, error, `bHomeWanted` stays TRUE: `bEnableOff` once (with G: once `bErrorLock`).
- G: `Error_Lock_Waits_For_Rest` (`bDriveReady` TRUE, `bAtRest` FALSE, `T#1H`: `bErrorLock` FALSE; `bAtRest` TRUE: TRUE),
  `Error_Lock_At_Once_When_Drive_Not_Ready`, `Error_Lock_After_Hold` (`T#0S`), `Error_Lock_Latched_Until_Error_Clears`,
  `No_Error_No_Error_Lock`.
- `On_And_Off_Never_Together`: extend the loop to `bError`, `bDriveReady`.

`FocusEnable_Sim_Tests` (drive model: on stop during a homing, the model's MC_Home goes to error and keeps Execute; enable
drop while homing/stopping counts `nLostEnable`):
- `Stop_During_Rehome_Home_Error_Old_Sticks` (0.10.2 order: `bRunning` TRUE after 200 cycles, `nLostEnable` 1).
- `Stop_During_Rehome_Home_Error_Ends_And_Locks` (E+G: `bAborted`, `bBlockAutoHome`, `nLostEnable` 0, `bEnable` FALSE once at rest).
- `Reset_After_Home_Error_Drive_Stays_Locked` (E: after the reset `ObFocusUnlock` FALSE, no automatic homing).
- `Stop_During_AutoHoming_Blocks_And_Locks` (E, stop of the automatic homing).
- `Error_During_Move_Keeps_Enable_Until_Rest` (G).

BROTLib (F): `FB_AxisCommand_Tests` unchanged if F only feeds `Aborted`; the `FB_Axis` part has no NC-free test, it is
covered by D7 and step 4 on site.

## 7. On site (MONET/S, then MONET/N)

Record with the TwinCAT Scope at task cycle, by symbol (prefix `FocusControl.`), from before `bManualHome`:
`fbAxis.Axis_Home.{Execute,Busy,Done,CommandAborted,Error,ErrorID}`,
`fbAxis.Axis_Stop.{Execute,Busy,Done,CommandAborted,Error,ErrorID}`, `fbAxis.Axis_Power.{Enable,Status,Error,ErrorID}`,
`fbAxis.Axis_Status.{Error,ErrorID}`, `fbAxis.Axis_Status.Status.{Homing,Stopping,StandStill,ErrorStop,Disabled}`,
`fbAxis.HomeCommand.{Execute,Pending}`, `fbAxis.StopCommand.{Execute,Pending,Joined}`,
`fbAxis.{Error,ErrorID,Ready,HomeActive,HomePending,Busy,StandStill,Stopping,CommandAborted,StopDone}`,
`refAxis.NcToPlc.{StateDWord,ErrorCode,ActVelo,ActPos}`, `refAxis.PlcToNc.ControlDWord`, `bEnable`, `ObFocusUnlock`,
`bFocusUnlocked`, `bError`, `nErrorID`, `bHomeAxis`, `IbLimitFar`, `fbDriveEnable.{bEnableOn,bEnableOff,bLockPending}`
(+`bErrorLock` after G), `fbManualHome.{bRunning,bAborted,bBlockAutoHome}`, `bHoldOnStop`, `eTelescopeMode`; plus
`TelescopeControl.{bStop,bError,TCS_command,bFocusErrorPark,eParkStage}`, `HydraulicsControl.bEnable`, the EL7342 focus
channel status word. Save the NC event log of each run.

- **D7 (before any fix, on 0.10.2):** step 4 run 1 again with the channels above. Answers 2.3. Recover with `reset=1`,
  then `bManualHome` (accepted while blocked).
- **Step 4 run 2 (60 s variant), on 0.10.2 too**, same recording: same block/ID expected; if not, the trigger depends on
  the homing phase (search vs. cam).
- Expected after E+F: step 4 both runs: no 0x4B0A, no 0x4260, `bEnable` TRUE until `StandStill`, then FALSE once; `bRunning`
  FALSE, `bAborted` one cycle, `bBlockAutoHome` TRUE, `READY_STATE` unchanged, no emergency park; then `bManualHome`
  again: full re-home, ends as step 3.
- Step 4c: `stop=1` during the power-on cam homing (uncalibrated focus): stops, no error, stays locked and
  uncalibrated, no automatic homing until the next `power=1` or `bManualHome`.
- With G only, a forced MC error during a move is not reproducible on purpose; record whatever errors the session gives.

### 7a. Hardware status (MONET/S, 2026-11-04)

Rows are in the IAG test plan results (`2026-10-12-monet-telescope-test-plan-results.md`); the open items are in T17 of
`2026-10-12-monet-telescope-test-plan.md` (the IAG test plan is split in two files since 2026-11-04).

| Test | Build | Result | Row |
|---|---|---|---|
| D7 and step 4, 2 s and 60 s | 0.10.2 / 0.11.0 | **failed** as described in section 1: NC 0x4260 22 ms after `STOP commencing`, stuck `bRunning`/`HomeActive`, emergency park | MS-95 (10), MS-100 |
| Step 4, `stop=1` 2 s after `bManualHome`, three runs | 1.6.4 | **passed on the PLC variables**: `bRunning`/`HomeActive` FALSE within 1 s, `bEnable` FALSE, uncalibrated, no telescope error, no park, no `reset=1`. Not read: NC event log (0x4260 not excluded), Scope trace, `bAborted` (read 30 s after the stop only) | MS-103 |
| Re-home after the stopped homing | 1.6.4 | **passed**: cam homing 5 min 2 s, calibrated, locked, 91.99 mm (not 91.5 mm, not investigated) | MS-104 |
| Step 4, 60 s variant (from far away) | 1.6.4 | **open** (all runs started near the cam) | T17 step 1 |
| Step 4c, `stop=1` during the power-on cam homing | 1.6.4 | **failed**, twice: `READY_STATE` -2 but the focus homing ran to its end and ended calibrated; see section 10 | MS-105, MS-106, MS-107 |
| Step 4c repeated | with #145 | **open**, needs a release | T17 step 1b |
| Steps 2, 6 (#117 plan) | 1.6.3 | passed | MS-97 |
| Step 5 (#110 `bManualUnlock`) | 1.6.3 | did not behave as the plan expects (to be judged) | MS-97 |
| Step 7 (brake-delay race) | 1.6.3 | passed | MS-98 |
| Step 8 (part): `Stop()` during a move, retarget during a long move | 1.6.3 | passed | MS-99 |
| Step 8, pendant jog (#66) and a clean `MC_Direct` restart | | no row found, open | |

Scope channels still to add for the next run: `fbDriveEnable.bLockPending`, `fbDriveEnable.bEnableOff`,
`fbManualHome.bBlockAutoHome`, and for 4c `fbFocus.HomeAxis` / `ePowerOnStage` (not in `FocusEnable.tcscopex`).

## 8. Risks and regressions

- **G changes the brake reaction to errors**: an MC error with the drive still `Ready` keeps the brake open up to
  `tErrorHold`. Needs Tim's safety view (Q1). If MC_Power `Status` does not drop on an NC axis error, G would delay the
  lock on drive faults too: D7 checks it; fall back to locking at once when `Axis_Status.Error` (needs a BROTLib output).
- E's `AND NOT bError` in `bWanted`: during an error the brake-delay timer (`tonRest`) may run; it only acts when
  `bCalibrated` and at rest, i.e. after the error at standstill, which locks earlier than today (fine).
- E's stop of the automatic homing blocks it: a power-on or `bManualHome` is needed. After a `reset=1` nothing homes by
  itself any more (today it does, via `:163`). Q2.
- F swallows one MC_Home error: restrict it to the ID seen in D7 and to `HomeStopped`, or a real homing fault is hidden.
- **Manual home outside automatic mode** (PR #117 removed `bEnable := TRUE` on `Accepted`): `Request()` rejects
  `NotAutomatic` and `ManualUnlock` (`FB_FocusManualHome.TcPOU:62-65`), so an accepted re-home always starts with
  `bEnableOn` (`bHomeWanted`, `bAutomatic`), as step 3 showed. If the mode leaves automatic during the homing, `bEnableOn`
  stops but nothing writes `bEnable` FALSE (still wanted), so the homing continues; but the pendant writes `Enable :=
  IbEnableButton` on selection 5 (MONETcommon pendant), which would cut the enable during the homing and give 0x4260
  (pre-existing, not from #117). A re-home still `HomePending` when the mode leaves automatic waits until automatic
  returns. #138 (accept an uncalibrated focus) must keep `NotAutomatic`, as it proposes.
- `FB_FocusManualHome` and `FB_FocusDriveEnable` get new inputs: only `FB_FocusControl` and the MONETcommon test branch
  call them (no consumer change, HalfBROT patch release).
- Line numbers of `FB_FocusControl` move with E; the 2026-10-25 plan's references stay at their commit.

## 9. Open questions (for Tim)

1. **G**: keep the brake open after an MC error until rest/`NOT Ready`/`tErrorHold` (recommended, `T#1S`), or keep the
   immediate lock and accept 0x4260 as the error shown?
2. **Stop of the automatic homing**: block it until `power=1`/`bManualHome` (recommended, same rule as the manual
   re-home), or let `:163` home again after a reset as today?
3. **F scope**: swallow only the D7 error ID of a stopped MC_Home in BROTLib (recommended), or report it as a warning
   event as well?
4. Was `nErrorID` 19210 also visible in the 11-03 recording (the comment lists only 0x4260)? And can the NC event log of
   both runs be exported (gives `StateDWord`/`ActVelo` at 0x4260 without a new run)?
5. Order: E alone first as 0.10.3 (the stuck state goes away, step 4 still fails with the error), or E+F+G together after
   D7 (recommended)?

## 10. Step 4c: stop during the power-on cam homing (2026-11-04)

**Observation (MS-105, MS-106, Scope analysis MS-107, `FocusEnable_8/_9.svdx`).** `stop=1` 10 s into the power-on cam homing:
the telescope aborts the power-on (`READY_STATE` -2, `TCS_command` 7 > 5 > 0 within 30 ms), but `Axis_Stop.Execute`, `StopCommand`,
`bHoldOnStop` and every error channel stay 0, and the homing runs on and ends calibrated about 12 s later. `bHomeAxis` was not cleared
by the power-on abort either.

**Cause (confirmed by the Scope, MS-107).** The cam homing has two phases (about 9.5 s each) with a 0.3 s `StandStill` gap between them,
while `MC_Home` keeps `Execute` and `Busy`. Both stops landed in that gap (`StandStill` 57.84 s, stop 57.93 s; run 8: 62.41 s, 62.52 s).
`FB_FocusControl.Stop()` sent the MC_Stop only under `fbAxis.Ready AND NOT fbAxis.StandStill`, so the stop was skipped. The telescope
calls `fbFocus.Stop()` once per stop transaction (`_StopTelescope`, `bStopTxBegun`), so there was no second try. Step 4 passed because the
manual home was stopped 2 s in, while the axis was moving.

**Fix.** HalfBROT PR #145: the condition is `fbAxis.Ready AND (NOT fbAxis.StandStill OR fbAxis.HomeActive OR fbAxis.HomePending)`.
Not compiled by hand, not run on a PLC. No PLC test in HalfBROT; a rig test (stop with `StandStill` TRUE during a homing) belongs in
MONETcommon (`FB_FocusMock`) and is not written.

**Open.**
- Re-run 4c on a build with #145, and with other delays than 10 s (3 to 5 s into phase 1, 15 s into phase 2, MS-107 caveat).
- After the stop, `fbFocus.HomeAxis := FALSE` of the power-on abort did not clear `bHomeAxis` (MS-107); check on the re-run with
  `bHomeAxis`/`ePowerOnStage` in the Scope.
- The same gap exists in the manual re-home from far away (phase change): the 60 s variant of step 4 covers part of it.
