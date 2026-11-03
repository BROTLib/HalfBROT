# Addendum: the D7 event log of 2026-11-03 13:55 and the #114 H2 chain

**Status: analysis only, no code changed, nothing compiled or run.** Addendum to
`2026-11-03-focus-stop-during-homing-114-h2.md` (the "plan"). Read on 2026-11-03 from the versions that ran in D7, i.e. what
MONETS `v1.6.3` (`6f725d7`) pins in `MONETSRuntime.plcproj`: HalfBROT `v0.11.0`, BROTLib `v0.17.0`, MONETcommon `v0.14.1`.
Line numbers are file lines at those tags (MONETS `MAIN.TcPOU` and `MONETS.tsproj` at `v1.6.3`). **Hypothesis** marks
anything the code and the logs do not settle.

Input: #114 and PR #141 comments of 2026-11-03 12:10 (D7 / step 4 on 0.11.0, NC event log run 1).

## 1. Result in short

The new log does **not** contradict the plan's chain. It is the chain, timed to the PLC cycle:

- `0x4B0A` cannot show up in the NC event log. It is a Tc2_MC2 function-block ErrorID that `MC_Home` returns to the PLC
  (BROTLib `NCError_TO_STRING.TcPOU:28`: "Error during referencing"), not an NC axis error. The NC logs only its own
  errors, so 0x4260 is the first and only TCNC entry. The PLC record of the first error is HalfBROT's own log line
  `Focus Axis Error: <text>` (`FB_FocusControl.TcPOU:493-500`, see 3.1), not a TCNC entry.
- In the code, nothing other than the `bError` gate (`FB_FocusControl.TcPOU:192`) can remove the focus controller enable
  while `HomeActive` is held. The telescope, the park, the hydraulics and the pendant never write it in this situation
  (section 2). The emergency park at .314 and the enable drop are **two reactions to the same focus error**, one PLC
  cycle after it. The park does not cause the drop.
- The 2 ms / 22 ms timing matches the task order exactly (section 3). Neither the 10-30 trace nor the new data needs a
  second mechanism.

Confidence: **high** that the enable is removed by the `bError` gate after an MC error (code: it is the only path).
**Medium-high** that the error is `Axis_Home` 0x4B0A caused by the MC_Stop (H2a). Three things support it: the 10-30
`nErrorID` 19210, the meaning of the ID, and the `StateDWord` at 0x4260, which shows the stop received and the homing no
longer busy (3.2). The Scope trace already recorded (`FocusEnable_5/6.svdx`) is not read yet.

## 2. Every writer of the focus enable, checked

| Path | Where | Can it act within 22 ms of the stop, before an error? |
|---|---|---|
| MC_Power `Enable` | BROTLib `FB_Axis.TcPOU:128-134` gets `enable := bFocusUnlocked` (`FB_FocusControl.TcPOU:251`) | this is the drop itself |
| `bFocusUnlocked` | `tonFocusDelay(IN := ObFocusUnlock)` `:195`: falls with `ObFocusUnlock` in the same call | follows `ObFocusUnlock` |
| `ObFocusUnlock` | `:192` `(bManualUnlock OR bEnable) AND NOT bError` (`bError` from the previous `fbAxis` call, `:262`) | only through `bError` or `bEnable` |
| `bEnable` by `fbDriveEnable.bEnableOff` | `FB_FocusDriveEnable.TcPOU` (0.11.0) `bEnableOff := NOT bWanted AND ...`; `bHomeWanted := bHomeAxis OR HomeActive OR HomePending` (`FB_FocusControl.TcPOU:182`) | **no**: `bHomeAxis` falls only on an error or when calibrated (`F_FocusHomeRelease`, `:281-283`) or on `fbManualHome.bAborted`, which needs `NOT HomeActive`; `HomeActive` stays TRUE (observed) |
| `bEnable` by the telescope | MONETcommon `FB_MonetTelescopeControl` writes to `fbFocus` only `Reset()` `:378` (only on `bReset`), `Position` `:1230` (park, only `NOT fbFocus.Error`), `HomeAxis := FALSE` `:1384` (power-on abort, no power-on was running), `Enable/HomeAxis := TRUE` `:1429-1431` (power-on), `Stop()` `:2439`, `:2836` | **no** `fbFocus.Enable := FALSE` anywhere |
| `bEnable` by the pendant | MONETcommon `FB_MonetPendantControl.TcPOU:176` (selector 5, key on), `:280` (`_DropEnable`, after the pendant let go) | no: key off during the test; nothing else |
| MQTT | BROTLib `FB_Comm_MQTT_Influx` writes only `Focus.Position` | no |
| Hydraulics | MONETcommon `:685` (E-stop/safety edge), `:1309` (park secure stage) `fbHydraulics.Enable := FALSE`; HalfBROT `FB_HydraulicsControl` drives pump/suction/Az-El brake only, no focus signal | no (and the hydraulics went off only after the error, run 08:24) |
| `ControlDWord` direct writes | none in HalfBROT/MONETS/MONETcommon; `MC_SetAcceptBlockedDriveSignal` (`:248`) sets bit 8 only | no |
| Az/El/Der error branch | MONETcommon `:553-557`, `:565-568`, `:987-991` | disables Az/El/Der only, never the focus |

The focus `Stop()` itself does not drop the enable: `FB_FocusControl.TcPOU:451-461` calls `fbFollower.Stop()`,
`CancelMove()`, `CancelHome()` (pending only; BROTLib `FB_AxisCommand.Cancel` leaves a running `Execute` alone) and,
because the homing axis is `Ready AND NOT StandStill`, `fbAxis.Stop()`. BROTLib `FB_Axis.Stop()` `:445-449` and `StopAllowed`
`:291-295` (includes `Homing`) send MC_Stop to the axis that runs MC_Home, as in plan 2.1 steps 2-3.

So with `HomeActive` held, **the only way to 0x4260 is `bError`** (code-confirmed). `bError` is `fbAxis.Error`
(`FB_Axis.TcPOU:361-370`), i.e. an MC block error or `Axis_Status.Error`. `Axis_Status.Error` (an NC axis error) would have
its own TCNC entry before .316, and the log has none, so it is an MC block error.

## 3. Cycle by cycle

Task setup (`MONETS.tsproj:2149`, `:2156`): `PlcTask` 10 ms, `NC-Task 1 SAF` 2 ms. Call order in `MAIN.TcPOU`: the
telescope `:159`, ..., `FocusControl` `:207`, `PendantControl` `:226`, `fbComm` `:255`. `STOP commencing` is the telescope's
own line (`_StopEvents`, `FB_MonetTelescopeControl.TcPOU:2375-2381`, logged at the end of its body `:1010`, source shown as
the task name `PlcTask`). The PLC task does not stop.

| Cycle (10 ms) | Telescope (runs first) | FocusControl (runs after) | NC (2 ms) | Log |
|---|---|---|---|---|
| N-1 | | | | `fbComm` handles `stop=1`: `Stop()` sets `bStop` (`:2827-2839`) |
| N | stop branch `:714-737`, `_StopTelescope` `:860`, `fbFocus.Stop()` once `:2435-2441` -> `fbAxis.Stop()` (request) | `StopCommand` -> `Axis_Stop.Execute` TRUE (`FB_Axis.TcPOU:297-302`) | gets MC_Stop: `HasBeenStopped`, homing procedure ends | **.294 `STOP commencing`** |
| N+1 | | `Axis_Home.Error`, ID 0x4B0A (H2a) -> `bError` TRUE (`:262`); `HandleEvents` (`:326`) logs it | decelerates under control, enable still on | **expected ~.304: `Focus Axis Error: Error during referencing`** (ERROR, `:493-500`) |
| N+2 | `fbFocus.Error` -> `bHwError` edge (`:391-397`): `F_ErrorParkRequest` -> `Park()` (queued, `bParkAfterStop`, `:655-664`, `:2686-2687`) and `bFocusErrorPark` (`:487-500`); both lines logged at the end of the body (`:1866-1870`, `:1902-1906`) | `ObFocusUnlock := ... AND NOT bError` FALSE (`:192`), `bFocusUnlocked` FALSE (`:195`), `fbAxis(enable := FALSE)`: MC_Power drops the controller enable | next SAF cycle: enable gone while the stop command still runs | **.314 emergency park**, **.316 0x4260** |

The two log entries 2 ms apart are expected. The telescope line is written in cycle N+2 before `FocusControl` runs; the
enable leaves with the outputs of N+2 and the 2 ms NC task sees it in its next cycle. From `STOP commencing` to 0x4260 is
two PLC cycles plus one NC cycle (~22 ms). Every step in the table is code. Only the source of the N+1 error is H2a.

### 3.1 What makes this checkable without a new run

If the chain is right, the full event log (all sources, not only TCNC) of run 1 contains, between .294 and .314, the
HalfBROT line `Focus Axis Error: Error during referencing` (the text of 0x4B0A, `NCError_TO_STRING.TcPOU:28`). At .314 it
also contains both `emergency park started: axis error` and `focus error: controlled stop of the axes if they move, then
emergency park to the park position`. The second one is logged only when the focus error is the only error
(`bFocusErrorOnly`, `:472-473`).
- If that `Focus Axis Error` line is there: settled. Its text names the ID (`Error during referencing` = 0x4B0A,
  `Axis is not ready` = 0x4B09, etc.).
- If it is missing and the park line is `hydraulics error: ...` or `power failure ...`, a different hardware error started
  the park. The focus enable drop is then still unexplained by the code (section 2), and the H2 chain has to be reopened.

### 3.2 The NC state at 0x4260

`StateDWord 0x22120381` = bits 0, 7, 8, 9, 17, 20, 25, 29. With the Tc2_MC2 `NCTOPLC_AXIS_REF_STATE` bit order (from
memory, verify in InfoSys) these are: Operational, **HasBeenStopped (7)**, HasJob, positive direction, NewTargetPos,
ControlLoopClosed, PTP mode, **MotionCommandsLocked (29, stop active)**. Not set: Homed (1) and **HomingBusy (11)**. So 22 ms
after the stop the NC had taken the MC_Stop and the homing procedure was no longer running, which is what H2a needs: MC_Home
sees its procedure end without `Homed` and reports an error instead of `CommandAborted`. `ActVelo 0.207961` (the 0.2 mm/s
homing speed) means the axis had hardly slowed down after ~20 ms. The plan's "the focus decelerates in ms" (G, section 4)
is not borne out (see 5).

### 3.3 The 10-30 trace (0.10.2, MONETcommon 0.11.2, ADS ~15 ms)

It is the same mechanism. The +44 ms / +144 ms are ADS recorder sample times relative to the MQTT send, not PLC cycles:
`nErrorID` 19210 at +44 ms with `bError` only at +144 ms cannot be real. `bError` and `nErrorID` are outputs of the same
`fbAxis` call (`:262-263`), and `ErrorID` is 0 whenever `Error` is FALSE (`FB_Axis.TcPOU:385-387`). Only the order is
meaningful: 0x4B0A first, then 0x4260, which overwrites it because `Axis_Status.ErrorID` is written last (`:351-353`). The
same applies to "590 ms" (run 1) vs "105 ms" (run 2) after the stop command: that is MQTT/recorder latency. The PLC-internal
gap in run 1 is 22 ms. MONETcommon 0.11.2 -> 0.14.1 changed the telescope's reaction (the 0.14 focus-error park
`bFocusErrorPark`, `:470-500`), not what removes the focus enable. HalfBROT 0.10.2 -> 0.11.0 (#136, #138) did not touch
`Stop()`, `:192` or `FB_FocusDriveEnable`. 0.11.0 is a valid D7 build for "before the fix".

## 4. E, F, G after the new data

| Commit | Verdict | Why |
|---|---|---|
| **E** manual re-home ends on `bError`; `bWanted ... AND NOT bError` | **valid, needed** | the stuck `bRunning`/`HomeActive` reproduced on 0.11.0 (MS-100). Code-only, independent of the trigger. With F it becomes defence for every other homing error (drive fault, a real 0x4B0A without a stop) |
| **E** (second part) stop of the automatic homing: `bHomeStopped`, `HomeAxis := FALSE`, `RequestLock()`, `Block()` | **valid, needs Q2** | with F the stopped homing ends through `NOT HomeActive AND NOT HomePending`. Without this part the section 3 gap of the plan (bHomeAxis held, drive enabled with no homing, for ever) stays open after F |
| **F** (BROTLib) a homing stopped by `Stop()` ends as aborted | **is the fix of step 4**, moves from "after D7" to "now", D7 read only to confirm the ID | E+G alone still leave `fbFocus.Error` -> `bHwError` -> `bError`, `READY_STATE` -1 and the focus-error emergency park (`:487-500`) on every stop of a homing. With F and HalfBROT 0.11.0 unchanged the stop takes the H1 path of PR #117: no error, `CommandAborted`, `HomeActive` drops, `fbManualHome.bAborted`, `HomeAxis := FALSE`, `RequestLock()`, lock at rest, `bBlockAutoHome` |
| **G** error lock waits for rest / `NOT Ready` / `tErrorHold` | **valid as hardening, off the #114 critical path; split out** | it would remove the 0x4260 cascade but not the error and park, so step 4 still fails with E+G. Q1 (safety) is open. `T#1S` is not known to cover the stop: `ActVelo` was still 0.208 mm/s at +22 ms, so the stop deceleration of the focus axis (NC parameter, MC_Stop has no `Deceleration` input in `FB_Axis.TcPOU:299-302`, so it uses the axis default) has to be read first. The `MC_Power.Status` question of plan 8 is still open as well |

## 5. Proposed smallest fix

**F in BROTLib, one commit on `v0.17.0`, released as 0.17.1; MONETS pins BROTLib 0.17.1** (HalfBROT has no own resolution,
the project pins all libraries; check that no HalfBROT/MONETcommon release is needed for the pin). Sketch for `FB_Axis`:

```
VAR
	HomeStopped		: BOOL;		// Stop() hit a running homing (HalfBROT#114)
	HomeStopAbort	: BOOL;		// this call: MC_Home failed because Stop() ended it, reported as an abort
END_VAR

// Stop(), before HomeCommand.Cancel():
IF HomeActive THEN
	HomeStopped := TRUE;
END_IF

// Home() and Reset clear HomeStopped; so does HomeActive FALSE

// body, after Axis_Home(...):
HomeStopAbort := HomeStopped AND Axis_Home.Error AND Axis_Home.ErrorID = 16#4B0A;
```

- `HomeCommand(... Aborted := Axis_Home.CommandAborted OR HomeStopAbort)` (`:270`). It uses last cycle's value, so `Execute`
  falls in the next call and MC_Home drops its error with it.
- Leave `Axis_Home.Error` out of `ErrorID` (`:339-341`) and `Error` (`:367`) while `HomeStopAbort`.
- `CommandAborted` (`:380-383`) `OR HomeStopAbort`.

This gives a one-cycle abort pulse, `Error` never TRUE, and `HomeActive` FALSE one call later. Only 0x4B0A after a stop is
swallowed; any other MC_Home error, or 0x4B0A without a stop, stays an error (plan Q3).

Tests (pure logic is limited here: `FB_Axis` has no NC-free test). Extract the decision into a function, e.g.
`F_HomeStopAbort(bHomeStopped, bError, nErrorID)`, and test it in `BROTLibTests`: stopped + 0x4B0A -> TRUE; not stopped ->
FALSE; stopped + another ID -> FALSE. Add `FB_AxisCommand_Tests` `Aborted_Releases_Execute_After_Error` (Execute TRUE,
error, then `Aborted` TRUE: Execute FALSE). HalfBROT: the plan's `FocusEnable_Sim_Tests` drive model gets a variant
"MC_Home reports CommandAborted on stop" to check that 0.11.0 + F gives `bAborted`, `bBlockAutoHome`, `nLostEnable` 0. Run
by CI or by Tim, not on this machine without asking.

Order: E (HalfBROT 0.11.1, PR #141 reduced) and F (BROTLib 0.17.1) are independent and may ship together in one MONETS pin
bump. G gets its own PR after Q1 and the decel check.

### 5.1 What settles the remaining doubt (no new run needed for the first two)

1. **Full event log of run 1** (all sources), 13:55:21.28-.33: look for the `Focus Axis Error: ...` line at ~.304 and the
   exact park line(s) at .314 (3.1). Also export run 2.
2. **The existing Scope files `FocusEnable_5/6.svdx`**, per task cycle from N-1 to N+3:
   - `fbAxis.Axis_Home.{Error,ErrorID,Busy,CommandAborted}` (expect Error with 19210 in N+1, no `CommandAborted`)
   - `fbAxis.Axis_Stop.{Execute,Error,ErrorID}` (expect no error)
   - `fbAxis.Axis_Power.{Enable,Status}` (Enable FALSE in N+2, `Status` until then: answers G's `bDriveReady`)
   - `bError`, `ObFocusUnlock`, `bFocusUnlocked`, `refAxis.PlcToNc.ControlDWord` bit 0
   - `refAxis.NcToPlc.StateDWord` bits 7/11/29
   - `TelescopeControl.{bFocusErrorPark,bStop}`
3. **Add for the post-fix run**:
   - `refAxis.NcToPlc.ActVelo` and `fbAxis.{StandStill,Stopping}`, to measure the stop time for G's `tErrorHold`
   - the NC axis parameter "Deceleration" (focus, Axis-ID 4) from the NC configuration
   - `fbAxis.CommandAborted`, `fbAxis.HomeActive`, `HomeStopped`/`HomeStopAbort` (BROTLib 0.17.1)
   - `fbDriveEnable.bLockPending` and `bEnableOff`
   - `TelescopeControl.bHwError`

   Pass: no `Focus Axis Error` line, no 0x4260, no park line, `READY_STATE` unchanged, `bEnable` FALSE once at
   `StandStill`, `bBlockAutoHome` TRUE.
4. Check the Beckhoff InfoSys text of Tc2_MC2 0x4B0A against BROTLib's "Error during referencing", and the
   `NCTOPLC_AXIS_REF_STATE` bit order used in 3.2.

## 6. PR #141

**Reduce it to E.** Keep:
- `FB_FocusManualHome.bError`, `Block()`
- `FB_FocusDriveEnable` `bWanted := (bMoveWanted OR bHomeWanted) AND NOT bError`, with `:38` back to
  `(bLockPending AND (bAtRest OR bError))`
- `FB_FocusControl` `bError := bError`, `bHomeStopped` (subject to Q2)

Move G to a branch of its own:
- `bDriveReady`, `tErrorHold`, `tonErrorHold`, `bErrorLock`
- `:192` with `fbDriveEnable.bErrorLock`

E alone is behaviour-neutral for the brake: an error still cuts the enable at once, as in 0.11.0. It removes the stuck
re-home, the clearing of the lock request by a stuck `HomeActive`, and the open brake after `reset=1`. The PR text and the
plan's status line should say that step 4 passes only with BROTLib F, and that E is the follow-up fix. Update plan Q5
accordingly: E + F together, G later.
