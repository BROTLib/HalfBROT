# Elevation: enable the axis only after the brake has settled (#130)

**Status: implemented for the elevation only (default 3 s, every enable), not compiled or run on a PLC. Azimuth (2.4), the MONETcommonTests cases (4) and the on-site check (6) are still open.** Facts read on 2026-10-28 from HalfBROT `develop` @ `2bf4d2b`
(0.9.0, oil level clamp #123). Line numbers are at that commit. The local checkout is on `ci/pin-brotlib-0.15.0` (one commit
ahead of `develop`, BROTLib pin only); the work branches from `origin/develop`, suggested name `fix/elevation-brake-settle-130`.

Issue: HalfBROT#130. Evidence: MONET/S, 2026-10-28 12:46, first `power=1` after a PLC restart (log in the issue).
Related: MONETcommon#121 (Ready only while enabled), MONETcommon#76 (automatic Wake&Shake), #113/#151/#155 (Wake&Shake
exemption and window in MONETcommon).

## 1. Problem and root cause

With the AX5000 parameter `P-0-0165` *Activation = On enable request* the drive runs the Wake&Shake (`P-0-0160`, commutation
offset) by itself at the first enable after a drive start. The Wake&Shake moves the motor; it needs the brake released. On
MONET/S the elevation failed 0.475 s after the PLC log `Brake is open` with `F10B` / `F601` / `F616` (*limited range of motion
during P-0-0160*); the azimuth in the same run did not fail.

`bBrakeOpen` is not measured. The chain in the code:

| Where | What | Time after the open command |
|---|---|---|
| `FB_HydraulicsControl.TcPOU:197-204` | `tonBrakeOpenDelay` (3 s pump + pressure), then `rsBrakeState` sets `ObBrakeAzEl` | 0 (output) |
| `FB_HydraulicsControl.TcPOU:206-210` | `tonBrakeOpenedDelay`, 1000 ms, `Q => bBrakeOpen` | 1 s |
| `FB_ElevationControl.TcPOU:84` | `fbAxis(Enable := bEnable AND bBrakeOpen, ...)` | 1 s, **no wait** |
| `FB_ElevationControl.TcPOU:123` | `tonReadyDelay(IN := bEnable AND bBrakeOpen, PT := T#6000MS, Q => bReady)` | 7 s |
| `FB_AzimuthControl.TcPOU:42-48, 56` | `tonRestTimer(IN := bBrakeOpen AND ABS(ActVelo) < 0.01, PT := T#2S)` -> `bAtRest`; `Enable := bEnable AND bAtRest AND bBrakeOpen` | >= 3 s (uncalibrated) |

So the azimuth enables at least 2 s after `bBrakeOpen` (by its "power chain" hack, not by design) and the elevation at
once. In the failing run the brake was therefore still not free about 1.5 s after the output went TRUE. The fault is in the
elevation enable timing, not in the hydraulics.

**Not known:** the real release time of the brake (no feedback input), whether it depends on oil temperature, and how long the
Wake&Shake itself takes on the elevation (relevant for the 6 s ready delay, section 7).

## 2. Change (HalfBROT, one PR to `develop`)

### 2.1 New pure-logic `FB_BrakeSettleEnable` (`POUs/`)

The `FB_ElevationControl` body cannot run in a test (MC blocks, `AXIS_REF`; `FB_ElevationMock` in MONETcommonTests skips
`SUPER^()`), so the timing goes into a small block without hardware references, as `FB_PumpResetPulse` (#120) and
`FB_FocusTargetInit` (#115) did. Sketch:

```
// Enable an axis only after the brake has been reported open for tSettle (#130). bBrakeOpen is inferred by the hydraulics
// (1 s after the output), the brake needs longer; an AX5000 with P-0-0165 = On enable request runs the Wake&Shake at the
// first enable and fails with F616 against a brake that is not yet released.
FUNCTION_BLOCK FB_BrakeSettleEnable
VAR_INPUT
	bEnable		: BOOL;		// the axis is wanted enabled
	bBrakeOpen	: BOOL;		// brake reported open (and any other enable precondition of the caller)
	tSettle		: TIME;		// wait after bEnable AND bBrakeOpen, T#0S = old behaviour
	tReadyDelay	: TIME;		// bReady after the axis enable
END_VAR
VAR_OUTPUT
	bAxisEnable	: BOOL;		// to fbAxis.Enable
	bSettling	: BOOL;		// enable and brake open, waiting for tSettle
	bReady		: BOOL;		// bAxisEnable for tReadyDelay (MONETcommon#121)
END_VAR
VAR
	tonSettle	: TON;
	tonReady	: TON;
END_VAR

tonSettle(IN := bEnable AND bBrakeOpen, PT := tSettle);
bAxisEnable := bEnable AND bBrakeOpen AND tonSettle.Q;
bSettling := bEnable AND bBrakeOpen AND NOT tonSettle.Q;
tonReady(IN := bAxisEnable, PT := tReadyDelay, Q => bReady);
```

`bAxisEnable` repeats `bEnable AND bBrakeOpen` so it drops in the same scan as either input, independent of how the TON is
called. (This is equivalent to the issue's `tonBrakeSettle`; the block only makes it testable and reusable for the azimuth.)

### 2.2 `FB_ElevationControl`

- New `VAR_INPUT tBrakeSettle : TIME := T#3S;` (comment: wait after the brake reports open before the axis is enabled, #130;
  to be confirmed on site). New `VAR fbBrakeSettle : FB_BrakeSettleEnable;`. Remove `tonReadyDelay` (`:13`).
- Call it **before** `fbAxis` (after `bBrakeOpen` is read at `:33-36`, e.g. just before `:79`), so `fbAxis` sees the enable of
  this scan:
  ```
  fbBrakeSettle(bEnable := bEnable, bBrakeOpen := bBrakeOpen, tSettle := tBrakeSettle, tReadyDelay := T#6000MS);
  ```
- `:84` `Enable := fbBrakeSettle.bAxisEnable,`
- `:121-123` `bReady := fbBrakeSettle.bReady;` with the #121 comment kept and extended: ready only while `fbAxis.Enable` is
  TRUE, the settle time included (bReady comes 9 s after `bBrakeOpen` with the default).
- `:159-160` `tonBrakeWarning` **unchanged** (`IN := NOT bEnable AND bBrakeOpen`). During the settle `bEnable` is TRUE, so no
  warning; the warning is about the brake being open without the axis being *wanted* (operator/ADS/pendant), which is also what
  MONETcommon `F_BrakeMustClose` and `tpPowerOnWarning` look at (`fbElevation.bEnable`, not `fbAxis.Enable`). Changing it to
  `NOT fbAxis.Enable` would warn on every power-on after 500 ms. Add a one-line comment saying so.
- Optional log: HINT `Elevation: brake open, enabling the axis in <tBrakeSettle>` on the rising edge of `bSettling` (helps the
  on-site timing). Decide in review; not needed for the fix.

### 2.3 Behaviour in the edge cases

| Case | Result |
|---|---|
| `bEnable` drops during the settle | settle aborted, `bAxisEnable` stays FALSE; `enableTrigger` closes the brake as today; the next enable waits the full `tSettle` again |
| brake reported closed during the settle (pressure loss, limit switch, E-stop, pump off) | same: no enable, full wait on the next open |
| brake closes after the axis is enabled | `bAxisEnable` FALSE in the same scan (as today with `bBrakeOpen`), `bReady` FALSE at once |
| one-scan glitch of `bBrakeOpen` | restarts the settle (as today the glitch disables the axis); acceptable, `bBrakeOpen` comes from an RS latch and a TON, glitches are not expected |
| brake already open when `bEnable` rises (manual azimuth, Wake&Shake window #151, Visu) | still waits `tSettle`; conservative, costs 3 s (decision D4) |
| `tBrakeSettle := T#0S` | identical to 0.9.0 (enable and ready timer in the scan the brake reports open) |
| `bConnected` FALSE | `bBrakeOpen` is FALSE, never enabled (unchanged) |

Every elevation enable gets 3 s later, not only the first one after a restart: the power-on (stage 10 of `_PowerOn`, no own
timeout, 12 h command timeout) and the pendant elevation mode (`FB_MonetPendantControl:156`, `Enable := IbEnableButton`,
hold-to-run: the axis moves 3 s later after each press, on top of the existing pump/brake delays).

### 2.4 Azimuth (decision D3)

`tonRestTimer` covers the uncalibrated path only by accident: once `bCalibrated` is TRUE, `bAtRest` is TRUE at once (`:44-45`)
and `Enable` follows `bBrakeOpen` without a wait. That is fine for the Wake&Shake as long as the azimuth is never calibrated
at a drive start (it homes at every PLC start since #7). It is not fine if the drive restarts or loses its commutation while
the NC keeps `Calibrated` (drive power cycle, some F errors); the Wake&Shake would then run against the closing brake again.

Recommendation: the same block in `FB_AzimuthControl`, same input `tBrakeSettle : TIME := T#3S`:
`fbBrakeSettle(bEnable := bEnable AND bAtRest, bBrakeOpen := bBrakeOpen, tSettle := tBrakeSettle, tReadyDelay := T#3000MS)`,
`Enable := fbBrakeSettle.bAxisEnable`, `bReady := fbBrakeSettle.bReady` (today `:23` already times Ready from `fbAxis.Enable`,
but from the previous scan's value; the block removes that one-scan lag). Uncalibrated: 3 s instead of 2 s after `bBrakeOpen`
(both timers run in parallel). Calibrated: 3 s instead of 0. `tonRestTimer` and `bAtRest` stay (they are about the power chain
moving the azimuth, a different concern). If the reviewers prefer a minimal fix, leave the azimuth for a follow-up issue; the
elevation change does not depend on it.

## 3. Decisions (proposed)

- **D1 input, not constant:** `tBrakeSettle` is a `VAR_INPUT` with default `T#3S` (as `tPumpResetPulse`, #120 D2), so the value
  found on site can be set in MONETS/MONETN `MAIN` without a library release, and `T#0S` restores the old behaviour. A
  `VAR CONSTANT` would need a release per try; a hard-coded `T#3S` as in the issue the same.
- **D2 3 s default:** from the issue. The evidence only says "more than about 1.5 s after the output". Confirm on site (section 6).
- **D3 azimuth symmetric:** yes, in the same PR, separate commit (section 2.4). Open for the reviewer.
- **D4 timer input `bEnable AND bBrakeOpen`** (as in the issue), not `bBrakeOpen` alone: simpler to reason about, the wait
  always restarts with a new enable. A timer on `bBrakeOpen` alone would skip the wait when the brake was open long before
  (Wake&Shake window, manual azimuth), slightly faster but one more state to test. Keep the issue's form.
- **D5 not in the hydraulics:** lengthening `tonBrakeOpenedDelay` (`FB_HydraulicsControl:207-210`) would fix both axes in one
  place, but `BrakeOpen` is read by MONETcommon (telescope `bReady`, `tpPowerOnWarning`, the Wake&Shake window and brake checks)
  and by `bBrakeClosed`/`tonHydraulicsOffTimer`; it would also delay "brake open" for consumers that do not need the settle. The
  settle is a property of the axis that has to move against the brake, so it belongs in the axis block.
- **D6 version:** minor bump through `release.yml` (new input with a default, consumers compile unchanged, but every elevation
  enable is 3 s later, which is a behaviour change on the telescope). Release after the pending BROTLib 0.15.0 pin release
  (MONETcommon already has a branch pinning HalfBROT 0.10.0), so likely 0.10.x or 0.11.0. The repo has no CHANGELOG; the release
  notes are the GitHub release text: "Elevation (and azimuth) enable `tBrakeSettle` (3 s) after the brake reports open, so the
  AX5000 Wake&Shake on enable does not fail with F616 (#130). Every enable is 3 s later; `tBrakeSettle := T#0S` restores the
  old timing."

## 4. Tests (MONETcommonTests, TcUnit)

HalfBROT has no test project (`HalfBROTTests`, step 0 of the cover plan, is still open). Pure-logic HalfBROT blocks are tested
in MONETcommonTests against the pinned HalfBROT, as `F_OilSuctionLatch_Tests` and `FB_FocusTargetInit_Tests`. New
`FB_BrakeSettleEnable_Tests`, multi-cycle tests in the `nCycles` / `TEST_FINISHED()` style of `MonetBrakeRelease_Tests`, with
short times (`tSettle := T#100MS`, `tReadyDelay := T#200MS`, 10 ms task) so a case finishes in a few dozen cycles:

1. `EnableAfterSettle`: `bEnable` and `bBrakeOpen` TRUE: `bAxisEnable` FALSE and `bSettling` TRUE until `tSettle`, then
   `bAxisEnable` TRUE and `bSettling` FALSE.
2. `NoEnableWithoutBrake`: `bEnable` TRUE, `bBrakeOpen` FALSE for 3 x `tSettle`: never enabled, `bReady` FALSE.
3. `NoEnableWithoutRequest`: `bBrakeOpen` TRUE, `bEnable` FALSE: never enabled (the Wake&Shake window case).
4. `BrakeLostDuringSettle`: brake drops at half `tSettle`, opens again: enable only a full `tSettle` after the reopen.
5. `EnableDroppedDuringSettle`: same with `bEnable`.
6. `BrakeLostAfterEnable`: `bAxisEnable` and `bReady` FALSE in the same cycle `bBrakeOpen` drops.
7. `EnableDroppedAfterEnable`: same with `bEnable`.
8. `ReadyAfterEnable`: `bReady` exactly `tReadyDelay` after `bAxisEnable`, never TRUE while `bAxisEnable` is FALSE (MONETcommon#121
   invariant checked in every cycle of cases 1 to 7).
9. `ZeroSettleIsOldBehaviour`: `tSettle := T#0S`: `bAxisEnable` in the first cycle with `bEnable AND bBrakeOpen`.
10. `BrakeOpenBeforeEnable`: brake open for 5 x `tSettle`, then `bEnable`: still waits `tSettle` (D4).

The wiring in `FB_ElevationControl` / `FB_AzimuthControl` (the block called before `fbAxis`, `Enable` and `bReady` from it,
`tonBrakeWarning` unchanged) cannot run in a test; it is checked by review and the CI compile. The telescope tests
(`FB_ElevationMock`, `FB_MonetRig`) are not affected: the mock does not run the real body and the new input has a default.
Run the MONETcommonTests suite unchanged to confirm.

**Cannot be tested off site:** the brake release time, whether 3 s is enough, the Wake&Shake itself (`F616`), its duration
against the 6 s ready delay, the effect of oil temperature, and the feel of the pendant with the extra 3 s.

Building and running the tests (XAE / `tctool`) is not part of writing this plan; this machine is the CI runner, ask first.

## 5. Consumers checked

- MONETcommon `FB_MonetTelescopeControl`: power-on stage 10 sets `Enable` and waits for `fbElevation.Ready` before `HomeAxis`
  (`:1220-1237`); later by 3 s, no timeout shorter than 12 h. Telescope `bReady` (`:786-791`) uses `fbElevation.bEnable` and
  `fbBrake.BrakeOpen`, not the axis enable, unchanged. `F_BrakeMustClose` and `tpPowerOnWarning` (`:479`) use `bEnable`,
  unchanged (the brake is not closed during the settle).
- MONETcommon #113/#151/#155 (`F_WakeShakeErrorExempt`, `F_WakeShakeWindow`, `bFirstPowerOnPending`): built for "the first
  power-on after a restart faults on the elevation". With *On enable request* and this fix the first power-on should no longer
  fault; the exemption and window stay as a fallback, no change now. Note the result on MONETcommon#76 (the automatic
  Wake&Shake it asks for is then the drive setting plus this wait) after the on-site check.
- `FB_MonetPendantControl` elevation/azimuth modes: 3 s more per enable press (section 2.3).
- MONET/N: `P-0-0165` there was read as Activation *Manual* (IAG test plan, 2026-10-19 entries); the wait is harmless with a
  manual Wake&Shake. Whether MONET/N switches to *On enable request* is a separate decision.

## 6. On-site verification (supervised, MONET/S)

The first run after the fix must be supervised: a failed Wake&Shake leaves the elevation drive in F616 until a reset of the
drive diagnostics (`S-0-0099`, from XAE or `bSoEReset` / `_SoEReset`).

**Build:** MONETS pinned to the HalfBROT release with #130 (note `Global_Version` of HalfBROT, BROTLib, MONETcommon); `MAIN`
leaves `tBrakeSettle` at its default. Roof closed, person at the E-stop, covers free to open, elevation near balance, key on Remote.
Suggested as a new row in the IAG test plan (next free MS number).

0. Read `P-0-0165` on both drives (activation, first phase ramp-up) and write it down. If the elevation is still in F616 from
   the 2026-10-28 run: reset via `S-0-0099`, confirm no F error.
1. Recorder at task rate (TwinCAT Scope) or 20 Hz over ADS: `HydraulicsControl.ObBrakeAzEl`, `bBrakeOpen`, `fbElevation.bEnable`,
   `fbElevation.fbBrakeSettle.bSettling/bAxisEnable`, `fbElevation.bReady`, `fbElevation.fbAxis.Enable`, elevation actual position,
   torque, drive diagnostic, `bError`/`nErrorID`; the same for the azimuth if D3 is in; PLC log.
2. Cold restart of the PLC (the drives need a fresh start for the automatic Wake&Shake; note whether the drives were power cycled).
3. `power=1`. Expect: `ObBrakeAzEl` TRUE, 1 s later `Brake is open`, 3 s later the elevation enable, a Wake&Shake motion without
   F10B/F601/F616, `bReady` 6 s after the enable, homing, power-on completes. Record the times between the four edges and the
   Wake&Shake duration (from the drive state or the motion).
4. Repeat steps 2 and 3 at least twice (three cold starts in all), once with cold oil if possible (first start of the day).
5. Power-on without a restart (`park=1`, `power=1`): no Wake&Shake, enable 3 s after `Brake is open`, no error.
6. Brake lost during the settle: `power=1` and, within 3 s of `Brake is open`, `park=1` or `stop=1`/E-stop as the procedure allows:
   the elevation never enables, no drive error, brake closed.
7. Pendant elevation mode: hold the enable button, the axis enables 3 s after the brake reports open; release, the brake closes.
8. If a step 3 fails with F616: reset via `S-0-0099`, set `tBrakeSettle := T#5S` in `MAIN` (no library release needed), activate,
   repeat from step 2; record the value that works. Do not try values below 3 s on the telescope (each failure is an F616 and a reset).

**Pass:** three cold starts without F616, enable delay as set, no new warnings (`Brake was opened without elevation axis enabled!`
must not appear during a normal power-on). **Fail:** F616 with the brake reported open for more than `tBrakeSettle`: the brake is
not the (only) cause; keep the recording and look at the `P-0-0165` ramp-up and the balance (as the MONETS#29 torque notes).

## 7. Risks and open questions

1. **Settle value (Q1):** 3 s is a guess with only a lower bound from one run (> about 1.5 s after the output). Oil temperature may
   lengthen the release. Proposal: 3 s default, raise in `MAIN` if a cold-oil start fails.
2. **Ready delay vs Wake&Shake duration (Q2):** `bReady` (and with it `HomeAxis` in the power-on) comes 6 s after the enable. If the
   Wake&Shake on enable takes longer, homing is requested while the drive is still commutating. Measure in step 3; if needed,
   time `bReady` from the drive being operational (`fbAxis.Ready`/`Status`) instead of a fixed delay, as a separate change.
3. **Azimuth symmetric timer (Q3, D3):** in this PR or later?
4. **Every enable 3 s later (Q4):** acceptable for the pendant? An alternative is to wait only for the first enable after a PLC
   start (when the Wake&Shake runs); rejected here because the brake mechanics are the same at every enable and enabling against
   a not-yet-free brake is bad for a normal enable too, but the operators should agree.
5. **Unbalanced telescope during the settle:** the brake is open and the elevation not enabled for 3 s more than today (today 1 s
   from the output). With the elevation balanced this is what the Wake&Shake window (#151) already accepts for minutes; on an
   unbalanced telescope it can drift. Watch the position in step 3.
6. **MONET/N (Q5):** keeps `P-0-0165` on *Manual*? The fix is harmless either way; the on-site check is MONET/S only.
7. Pure-logic block tested in MONETcommonTests means the tests only run once MONETcommon pins the HalfBROT release (as for #120);
   until then the change is checked by the CI compile only.

## 8. Still to do

1. HalfBROT: `FB_BrakeSettleEnable`, elevation wiring (2.2), azimuth (2.4, if D3), PR to `develop`, CI compile.
2. Release (D6), MONETcommon pin, `FB_BrakeSettleEnable_Tests` in MONETcommonTests, MONETS/MONETN pins.
3. On-site check section 6 (supervised), then set D2 and close #130; comment on MONETcommon#76.
