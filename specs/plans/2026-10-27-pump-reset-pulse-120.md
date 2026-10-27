# Pump starter reset pulse (#120)

**Status: implemented in HalfBROT (branch `fix/pump-reset-pulse-120`), not compiled, not run on a PLC. Tests in MONETcommonTests and the
on-site check (IAG test plan MS-75) are still to do.** Facts read on 2026-10-27 from HalfBROT `develop` @ `07d0c49` (0.8.0).
Line numbers are `FB_HydraulicsControl.TcPOU` at that commit.

Issue: HalfBROT#120. Evidence: IAG test plan MS-71 (MONET/S, 2026-10-26).

## 1. Problem

The main oil pump did not start for five power-ons in a row after a PLC cold reset. `power=1` enabled the pump, `bPumpRunning`
stayed FALSE, 10 s later `tonMainPumpWatchdog` (`:230`) latched `bError` and the telescope parked. Only a **forced** 1 s pulse
of both reset outputs in XAE cleared it. Nothing from the PLC side helped: `reset=1` does not touch the hydraulics, `bReset`
clears the latch but the pump is enabled again in the same cycle, ADS writes of the reset outputs are overwritten.

Why the PLC reset never worked:

| Line | What | Effect |
|---|---|---|
| `:136-137` | `ObResetOilPump := bReset; ObResetSuctionPump := bReset` | the outputs follow `bReset` |
| `:140-148` | `IF NOT bError THEN bReset := FALSE` | `bReset` lives 1 to 2 cycles (10 to 20 ms); without a latched error one cycle |
| `:151-156` | `rsOilPumpState` with `SET := bEnable` | with `bEnable` still TRUE the pump is enabled in the cycle the latch clears |
| `:230-233` | `tonMainPumpWatchdog`, `ObOilPumpEnable AND NOT bPumpRunning`, 10 s | trips again with the same message, which does not name the starter |

The reset is also only ever sent while a software error is latched. A starter that has its own fault while the PLC has no
error (the state after the cold reset) never gets one.

**Not known:** why the starters needed a reset (abrupt power-down of the enabled drives and pump, or a fault the PLC does not see),
and what the two outputs switch (motor protection relay, phase monitor, NO or NC). The 10 to 20 ms pulse being too short is
the likeliest cause, not proven: the only working case is the 1 s force.

## 2. Operator workflow this automates

The usual manual sequence when the pump does not start (operator, 2026-10-27):

1. disable the hydraulics,
2. clear the errors,
3. pulse the pump resets (XAE force, about 1 s),
4. enable the hydraulics.

| Manual step | In the code |
|---|---|
| 1 disable | `bHoldPump` keeps `ObOilPumpEnable` FALSE during the pulse and one cycle after it. `bEnable` is not touched: the telescope control writes it back to TRUE within 20 ms anyway, only the output is held. |
| 2 clear errors | `bReset` clears `srErrorState` as before. |
| 3 pulse | `FB_PumpResetPulse` sends `tPumpResetPulse` (1 s) on both outputs. |
| 4 enable | the hold ends, `ObOilPumpEnable` goes TRUE, and only now does the 10 s watchdog start. |

Clearing and pulsing happen in the same cycle; the hold means the pump never sees an enable before the pulse is over, so the
order is the same in effect.

## 3. Change (HalfBROT, one PR to `develop`)

### 3.1 New pure-logic `FB_PumpResetPulse` (`POUs/`)

State in its own VAR, no hardware reference, testable in MONETcommonTests like `FB_FocusTargetInit`. Inputs: `bResetRequest`,
`bPumpRequest`, `bPumpFailure`, `bPumpRunning`, `bSuctionRunning`, `tPulse`. Outputs: `bResetPump`, `bResetSuction`,
`bHoldPump`, `bActive`, `sReason`.

- A pulse (`TP`, so its length does not depend on how long `bReset` is TRUE) starts when
  - `bResetRequest` has a rising edge (reason `requested`), or
  - a pulse is due (`bNeeded`, initial TRUE so it is due after every PLC start and download, set again by a rising edge of
    `bPumpFailure`), the pump is requested and has no run feedback (reasons `before first start`, `after pump failure`).
- `bNeeded` is cleared when a pulse starts, or when the pump is requested and already running (no pulse for a running pump).
- `bResetPump` and `bResetSuction` are only TRUE while the respective pump has no run feedback. A reset button pressed while
  the pump runs neither resets nor holds it (holding it would drop `bPumpRunning` and close the brake).
- `bHoldPump` is TRUE during the pulse and in the one cycle after it (the optional point 4 of the issue), not for a running pump.
- `tPulse = T#0S`: old behaviour, the outputs follow `bResetRequest`, no hold, no automatic pulse.
- No automatic retry: after a new failure `bError` holds the pump off until the next reset, which pulses again.

### 3.2 `FB_HydraulicsControl`

- New input `tPumpResetPulse : TIME := T#1S`, new output `bPumpResetActive`.
- `bResetRequested := bReset` before the error latch clears it; the reset outputs come from `fbPumpReset` after `rsOilPumpState`;
  `ObOilPumpEnable := rsOilPumpState.Q1 AND NOT fbPumpReset.bHoldPump`.
- Log: the old `Main hydraulics pump / pressure: FAILURE` stays (test plans look for it). New ERROR
  `main pump: no run feedback 10 s after the enable, check the starter, motor protection and phase monitor in the cabinet; a reset pulse is sent to the starters on the next reset`,
  kept until the next reset (`bMainPumpFailure` is a one-cycle pulse), and a HINT `pump starters: reset pulse (<reason>)`.

## 4. Decisions

- **D1:** the automatic pulse is sent only before the first start after a PLC start and after a main pump failure, plus on every reset (as in the
  issue). A pulse before every start would also catch starter trips the PLC never saw, at 1 s more per power-on: left open until the on-site check
  shows whether it is needed.
- **D2:** 1 s default, from the one working force. It is an input, so the value can change without a new library release.
- **D3:** no `statusWord` bit and no MQTT topic for the reset (the word is decoded outside this repo, not checked).
- **D4:** version bump through the release workflow (patch assumed: the new input has a default, consumers compile unchanged).

## 5. Backward compatibility

With `tPumpResetPulse := T#0S` the block behaves as before. With the default the first power-on after a PLC start and the first
power-on after a pump failure start 1 s later. ADS writes of the reset outputs still have no effect (written every cycle); only an
XAE force overrides them. MONETS and MONETN need no `MAIN` change; they pin the new HalfBROT.

## 6. Risks

- **Unknown wiring of the reset outputs.** If a reset is an NC contact in the starter control circuit, TRUE stops that motor. The run-feedback gating keeps a running pump out of it; the wiring still has to be checked.
- Without hardware (test bench) `bPumpRunning` is always FALSE: every start gets a pulse and still fails after 10 s, as today.
- A PLC stop and start over ADS does not reinitialise (MS-70), so `bNeeded` is not set again; a manual reset still works.
- Cause of the starter fault is still unknown; if the pulse does not help, look at the motor protection and phase monitor first, before forcing anything.

## 7. Still to do

1. MONETcommonTests: `FB_PumpResetPulse_Tests` (first request pulses and holds, second request no pulse, pulse after a failure edge, one-cycle `bResetRequest` gives the full pulse, held `bResetRequest` one pulse, reset with the pump running gives no pump reset or hold, running pump at first request clears `bNeeded` without a pulse, `tPulse = T#0S` equals the old behaviour) and a multi-cycle scan model of the error latch, pump state, `FB_PumpResetPulse` and the watchdog.
2. Release, pins in MONETS and MONETN.
3. On-site check: IAG test plan MS-75.
4. Separate issues: `reset=1` in MONETcommon could call `HydraulicsControl.Reset()`; the derotator NC error 18000 in the same millisecond as the hydraulics error is unexplained.
