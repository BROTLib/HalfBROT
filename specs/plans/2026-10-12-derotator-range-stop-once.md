# Derotator: stop once when leaving the software range (#20)

**Status: implemented on `develop` (see the PR linked from #20), not run on a PLC or a telescope.** Built by CI
(TcBuild) only. Hardware check: added to #59.

## Problem

`FB_DerotatorControl` set `StopAxis := TRUE` in every cycle while `ActualPosition` is outside
`fMinPosition`..`fMaxPosition`. `StopAxis` is edge-triggered and `bStopAxis` is cleared on `StopDone`, so a new
`fbAxis.Stop()` started after every finished stop, for as long as the axis was out of range. Since BROTLib #63
`Stop()` also cancels pending `MoveTo()`/`Home()`, and other motion is rejected while `MC_Stop` executes.

Read from code:

- a `MoveTo()` back into range is cancelled again and again; only jogging inward (the outward direction is
  blocked by `Enable_Positive/Negative`) gets through, between stops;
- the derotator has an absolute encoder, so `ActualPosition` is valid before homing: a derotator out of range at
  start-up has its `Home()` cancelled repeatedly, never becomes `Calibrated`, and MONETcommon's start-up stalls;
- a cancelled `Home()` left `bHomeAxis` TRUE, so the edge-triggered `HomeAxis` setter (BROTLib `FB_BaseAxis`)
  never requested it again.

Ranges: MONETS -90..360, MONETN -170..300, so this needs a manual move or a wrong limit to happen.

## Change

- `bOutOfRange` and an `R_TRIG`: `StopAxis := TRUE` only on the rising edge (leaving the range, or out of range
  in the first cycle).
- With that stop, `bMoveAxis` and `bHomeAxis` are cleared, so a move can be commanded again and homing is
  re-requested in the next cycle while not calibrated.
- Unchanged: `Enable_Positive := ActualPosition < fMaxPosition`, `Enable_Negative := ActualPosition >
  fMinPosition` (no outward motion), `fPosition := LIMIT(fMinPosition, fPosition, fMaxPosition)` (every target is
  inside the range, so every move leads back in).

### Hardware limit inputs: not changed

`InDigitalInputs` is linked in MONETS/MONETN to the drive's "Digital inputs, state" (AT 1). `InPosLimit`/
`InNegLimit` (TRUE = at the limit) are used only by the pendant (MONETcommon `FB_MonetPendantControl`). Whether
the drive evaluates the limit switches itself is not visible in the code, and wiring them into
`Enable_Positive/Negative` without knowing the polarity would be a blind behaviour change. Comments on the two
properties say so.

## Checks

- CI: TcBuild of HalfBROT.
- Hardware (#59): with the derotator calibrated and at rest, set `DerotatorControl.fMaxPosition` over ADS just
  below the current position: one stop (`bStopAxis` rises once, not repeatedly), then a `MoveTo` to a position
  inside the range runs. Restore `fMaxPosition`. Optionally the same before homing (PLC restart with the limit
  set): the derotator is homed (`Calibrated`) after the single stop.
