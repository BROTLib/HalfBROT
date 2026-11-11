# Focus: drive powered before the lock opens

**Status: implemented (branch `fix/focus-power-before-unlock`); CI compiles and runs `FB_FocusUnlock_Tests`. Not activated or
run on a telescope.** Base: HalfBROT `origin/develop` @ `0e0f552` (BROTLib 0.20.0). Measurement: MONETN v1.7.0 (HalfBROT
0.14.0), 2026-11-11, iag `specs/plans/2026-11-10-monetn-update-test-checklist-DRAFT.md` section 4a.

## 1. Problem

`FB_FocusControl` opened the lock first and powered the drive `ptFocusDelay` (500 ms) later:

```
ObFocusUnlock := (bManualUnlock OR bEnable) AND NOT bError;
tonFocusDelay(IN := ObFocusUnlock, PT := ptFocusDelay, Q => bFocusUnlocked);
fbAxis(enable := bFocusUnlocked, ...);
```

For about 0.4 s after `ObFocusUnlock` nothing holds the focus. It falls under gravity until MC_Power closes the loop, and the
drive then pulls it back. Scope recordings `MONETN/MotorTuning/FocusEnable.svdx`, `_1`, `_2`:

| Elevation | Drop between unlock and drive power |
|---|---|
| 20 deg | -0.005 / -0.012 mm |
| 40 deg | -0.022 .. -0.032 mm |
| 85 deg | -0.050 / -0.056 mm |

The move still ends within 0.002 mm, so this is a cleanliness fix. The old order probably exists so that the motor does not work
against the brake. The new order keeps that rule: nothing moves until the lock has been open for `ptFocusDelay`.

## 2. New order (`FB_FocusUnlock`, pure logic)

1. `bManualUnlock OR bEnable` -> `bDriveEnable` (fbAxis.enable, MC_Power). The lock stays closed. The setpoint does not
   change, so the drive holds at standstill and does not drive against the brake.
2. `fbAxis.Ready` (MC_Power.Status) for `ptPowerSettle` (new, 100 ms) -> `bUnlock` (`ObFocusUnlock`, `stOut.bUnlock`).
3. Lock open for `ptFocusDelay` (unchanged, 500 ms) -> `bUnlocked` (`bFocusUnlocked`). The follower `bReady`, the jog gate
   (`fbJogRequest.bCanJog`), the homing start and `bReady` all wait for it.
4. No power status within `ptPowerTimeout` (new, 2 s) of the enable, or the power status lost while enabled -> `bPowerTimeout`.
   The lock stays closed (or closes), the drive is dropped, and `bError` is TRUE with `nErrorID = 16#FF01` (not an NC code).
   An ERROR event is logged. `Reset()` clears it and the sequence runs again.
5. Locking is unchanged. Without the request (`FB_FocusDriveEnable` after `ptBrakeDelay`, a lock request, or bManualUnlock off)
   or on `bError`, the lock closes and the drive drops in the same cycle.

## 3. Homing

`FB_Axis` starts a `Home()` as soon as the drive is Ready (`HomeCommand(Allowed := Ready)`). In the new order that is before the
lock opens, so a cam homing would drive against the lock. `_StartHoming` therefore only notes the request (`bHomeDeferred`). The
body calls `fbAxis.Home(fHomingReference)` once `bFocusUnlocked AND fbAxis.Ready AND NOT bError`. Wherever the body asks whether
a homing is waiting (`F_FocusHomeRelease`, `fbManualHome`, the `bHomeStopped` release), the deferred request counts as
`HomePending`. `Stop()` and the end of the homing request (`HomeAxis := FALSE`, an axis error releases `bHomeAxis`) drop it.

## 4. Review of the other users

| User | With the new order |
|---|---|
| `bManualUnlock` | Same path as `bEnable`: the drive is powered, then the lock opens. **Change:** a manual unlock without a drive (NC or terminal down) no longer opens the lock. It becomes a power timeout instead. |
| Pendant (`Enable` toggle, `MovePos`/`MoveNeg`) | Enable -> power -> lock -> 500 ms -> jog allowed. About `ptPowerSettle` plus the power-up time later than before. The lamps (`Ready`) stay off while the sequence runs, as before. A jog pressed while locked is still cleared (#148). |
| `fbJogLatch` / `fbJogRequest` | Gated on `bFocusUnlocked` (unchanged expression). |
| Automatic homing (uncalibrated, `fbAxis.Ready`) | `Ready` now comes before the lock opens. `HomeAxis := TRUE` is set earlier, but the `Home()` waits for `bFocusUnlocked` (section 3). |
| `fbManualHome` | Its request enables the drive (`bHomeWanted`). The `Home()` waits for the unlock. While it waits it counts as pending, so the re-home is not taken as ended. |
| `F_FocusTrackStoredPosition` (`NOT ObFocusUnlock` = locked) | Unchanged. While the drive is powered before the unlock, the lock is closed and the drive holds, so the focus cannot move and counts as locked. |
| Persistent write (`NOT ObFocusUnlock`) | Unchanged, same argument. |
| `bReady` | `fbAxis.Ready AND StandStill AND bFocusUnlocked OR NOT bDriveEnable`: FALSE while the sequence runs, as before when the drive only came up 500 ms after the unlock. |
| `bError` | Closes the lock and drops the drive in the cycle that sees it (unchanged). A power timeout is an error, so `fPosition := fActualPosition` and `fLastPosition := -1` apply as for an NC error, and the next restart homes on the cam. |

Not covered by CI: the wiring inside `FB_FocusControl` (no NC axis in the tests; `FB_FocusMock` in MONETcommon has no body).
That is what the telescope test checks. MONETcommon still pins HalfBROT 0.14.0. Its mock does not call `fbAxis.Home()`
(`fbCore` follows `bHomeAxis`), so the bump should not change its tests.

Open for the telescope: is `MC_Power.Status` TRUE while a feed enable is off (focus on a limit switch)? If not, an unlock on a
limit switch would end in a power timeout. The far switch is the homing cam, so a homing tests it.

## 5. Telescope test (Tim, XAE)

Scope `MONETN/MotorTuning/FocusEnable.tcscopex`. It already has `ObFocusUnlock`, `bFocusUnlocked`, `fbAxis.Axis_Power.Enable`
and `.Status`, and `fActualPosition`. Add:

- `MAIN.FocusControl.fbFocusUnlock.bDriveEnable` and `.bPowerTimeout`
- the load of the focus drive. The EL7342 (Focus Motor) has no torque in the NC. Use `MAIN.FocusControl.refAxis.NcToPlc.PosDiff`
  (lag distance: the position controller output follows it, so about 0 means no load). Where the PDO `DCM Synchron info data
  Channel 1` is assigned and info data 1/2 select the motor current (CoE 0x8022:11 / :19), add `Info data 1/2` of the focus
  channel as well.

Steps, each with the focus locked at rest for more than `ptBrakeDelay` before it:

1. El 85: a 2 mm focus move. Then the same back.
2. El 20: a 2 mm focus move. Then the same back.
3. Pendant: Enable, short jog, wait until it locks.
4. Optional: manual re-home (Visu), to check the homing waits for the unlock.

Pass:

- `Axis_Power.Status` rises before `ObFocusUnlock`. `ObFocusUnlock` follows about 100 ms later.
- `fActualPosition` does not drop between `ObFocusUnlock` and the move start. Today the drop is -0.005 mm at el 20 and -0.056 mm
  at el 85. Expect the encoder noise (about 0.001 mm).
- The move (or jog, homing) starts no earlier than 500 ms after `ObFocusUnlock`.
- `PosDiff` (or the current) stays at about 0 while the drive is powered and the lock is still closed.
- No focus error, no `bPowerTimeout`, and the move ends within `fHoldBand` as before.
- Locking: lock and drive enable drop together, `ptBrakeDelay` after the move (unchanged).

If `ObFocusUnlock` never comes and `bPowerTimeout` rises, `Axis_Power.Status` did not come within 2 s. Reset the focus and check
the drive. The lock stayed closed.
