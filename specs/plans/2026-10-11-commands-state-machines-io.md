# Commands through methods, explicit state machines, I/O through structs

**Status: draft.** Nothing implemented. Facts below were read from `develop` @ `4ef3b7b` (v0.5.1) and the
consumers' `develop` on 2026-10-11; nothing was run on a PLC or a telescope.

Issues: #31 (D3, umbrella), #32 (D4, tests). The bug class it removes: #4 (cover stale outputs), #5 (pendant
stale motion commands), and it touches #7 (position restore) and #20 (derotator stop re-asserted).

## Problem

1. **Inputs used as state.** The blocks write their own `VAR_INPUT`s, so a caller that assigns an input every
   cycle overrides what the block just decided (e.g. the block drops `bOpenBrake` on pressure loss, the caller
   sets it again next cycle).
2. **Behaviour in latched booleans.** Each operation is a flag that some completion or error clears in another
   place. Forget one clear path and an output stays set (#4, #5).
3. **I/O declared inside reusable blocks.** 45 `AT %I*`/`%Q*` variables in six blocks, mapped by each
   application's system project. Such a block cannot be instantiated in a test project without hardware (#32).

## Current state

### Inputs the blocks write themselves

| Block | Inputs written inside the block |
|---|---|
| `FB_AzimuthControl` | `fPosition` (clamp, `fCalibPosition`), `bHomeAxis`, `bMoveAxis`, `bReset`, `bStopAxis` |
| `FB_ElevationControl` | `fPosition` (clamp 0–90, `fCalibPosition`), `bHomeAxis`, `bMoveAxis`, `bTracking`, `bReset`, `bStopAxis` |
| `FB_DerotatorControl` | `fPosition` (clamp, `fCalibPosition`), `bHomeAxis`, `bMoveAxis`, `bReset`, `bStopAxis` |
| `FB_FocusControl` | `fPosition` (clamp, last/homing position), `bEnable`, `bMoveAxis`, `bHomeAxis`, `bReset` |
| `FB_HydraulicsControl` | `bEnable`, `bReset`, `bOpenBrake`, `bCloseBrake`, `bManualSuction`, `bManualPumpStop` |
| `FB_CoverControl` | `bOpen`, `bClose`, `bReset` |

`bEnable`, `bReset`, `bHomeAxis`, `fPosition` of the axis blocks are inherited from BROTLib's `FB_BaseAxis`.
Some writes are property/method setters (`MoveAxis`, `StopAxis`, `Tracking`, `OpenBrake()`, `CloseBrake()`,
`Enable`), i.e. the public API already exists but only forwards into the input.

### Who writes these inputs from outside

| Caller | Write | Note |
|---|---|---|
| MONETS `MAIN` (l. 82), MONETN `MAIN` (l. 79) | `HydraulicsControl.bOpenBrake := brakeClearing` | every cycle in manual/off mode; level semantics from the cabinet "brakes cleared" switch |
| MONETcommon `FB_MonetTelescopeControl` (l. 403–405) | `fbElevation/fbAzimuth/fbDerotator.bTracking := FALSE` | |
| MONETcommon `FB_MonetPendantControl` (l. 101–102) | `fbCovers.bOpen/bClose` | on `FB_MonetCoverControl` (MONETcommon), **not** HalfBROT's |

### Who uses which block

- MONETS, MONETN: `FB_HydraulicsControl`, `FB_FocusControl`, `FB_AzimuthControl`, `FB_ElevationControl`,
  `FB_DerotatorControl` (instances in `MAIN`, references in MONETcommon).
- HalfBROT `FB_CoverControl`, `FB_PendantControl`: no IAG telescope instantiates them (MONETS/MONETN use
  MONETcommon's `FB_MonetCoverControl`/`FB_MonetPendantControl`). They stay in the library (#29) and get the
  same treatment as the other blocks; external users (Potsdam) are not a constraint (decided 2026-10-11).
  Consequence: changes to them can only be checked by TcUnit (step 4), never on an IAG telescope, and they
  have no I/O links in MONETS/MONETN, so step 3 costs no re-mapping for them.

### `AT` declarations

`FB_HydraulicsControl` 16, `FB_PendantControl` 16, `FB_AxisControl` 4, `FB_CoverControl` 4, `FB_FocusControl` 3,
`FB_TelescopeAuxiliary` 2. MONETS/MONETN map them in `MONETS.tsproj`/`MONETN.tsproj` (by symbol path, e.g.
`MAIN.HydraulicsControl.<var>`).

## Plan

Four steps, in this order, each its own PR and release. Block order inside each step: hydraulics first
(MONET-only, most latches, the per-cycle caller), then focus and the three axes, cover and pendant last (not on
any IAG telescope, so only testable with TcUnit).

### Step 1: commands only through methods (small change in MONETcommon/MONETS/MONETN)

- Move the command inputs a block writes itself into `VAR` (internal request flags); keep the existing
  methods/properties (`OpenBrake()`, `CloseBrake()`, `Enable`, `Reset()`, `MoveAxis`, `StopAxis`, `Tracking`)
  as the only way in. For the axis blocks, check first what BROTLib v0.7.0's `FB_Axis` command methods already
  cover, so HalfBROT does not duplicate them; the inherited `FB_BaseAxis` inputs can only move if BROTLib does
  the same (separate BROTLib issue if needed).
- `fPosition`: the block keeps its own clamped target in `VAR` instead of overwriting the caller's input.
- Consumers: MONETS/MONETN `MAIN` replace the per-cycle `bOpenBrake := brakeClearing` with a call that keeps
  the level semantics (see open question 1); MONETcommon replaces `.bTracking := FALSE` with the property.
- Release: HalfBROT minor (inputs removed from the interface), MONETcommon/MONETS/MONETN bumps.
- Test: XAE build of HalfBROT, MONETcommon, MONETS, MONETN (Tim); hardware-test issue for the brake in manual
  mode (open by switch, drops on pressure loss and is **not** re-opened while the switch stays on).

### Step 2: I/O through structs, no `AT` in the library (breaking: re-linking in MONETS/MONETN)

Decided 2026-10-11 over the alternative (keep `AT` in the blocks, route it through overridable
`ReadInputs()`/`WriteOutputs()` methods so a derived test block can feed values): moving the I/O out keeps
hardware out of the library (#30), makes the blocks directly testable, and needs no extra indirection. The
risk of a silently lost link is covered by checking all links in XAE before activation.

- Per block an `ST_<Block>Inputs` / `ST_<Block>Outputs` passed as `VAR_IN_OUT` (see open question 3); the
  `AT` declarations move to an I/O GVL in each application (e.g. `GVL_IO.stHydraulicsIn`), linked there.
- Re-linking: MONETS and MONETN only (hydraulics 16, focus 3, axes 4 variables; cover/pendant are not linked
  there, so moving them costs nothing). A script rewrites the `VarA` symbol paths of the `<Link>` entries in
  `MONETS.tsproj`/`MONETN.tsproj` (`PlcTask Inputs^MAIN.HydraulicsControl.IbOilLow` → new path) and checks
  that the link count per block is unchanged.
- Before activation: Tim checks in XAE that every new I/O variable is linked and nothing is left unlinked.
- After activation: read the I/O values over ADS before and after the update and
  compare (same telescope state, e.g. hydraulics off, covers closed).
- Release: HalfBROT minor, MONETcommon (if its references change), MONETS/MONETN bumps.

### Step 3: TcUnit tests of the current behaviour (#32)

- `HalfBROTTests` project like `MONETcommonTests`, run by Tim in XAE (CI has no runtime). Possible for every
  block once step 2 is done (I/O is just struct members).
- Characterisation tests first: pin down what the blocks do today, including the known bugs (#4, #5) as
  expected-failure or documented cases, so step 4 shows exactly what changes.
- First targets: hydraulics (start-up, pressure loss closes the brake, watchdog timeouts, reset from error),
  cover sequencing 1→3→2 / 2→3→1, and the step 1 rule that a caller cannot re-open the brake.

### Step 4: explicit state machine per block (behaviour change, hardware test)

- One `E_<Block>State` enum, one `CASE` with the transitions, one section at the end of the body that writes
  every output from the state (no output written anywhere else). Errors are a state, left only by `Reset()`.
- Pattern: IAG50cm's `FB_CoverState`/`FB_PendantState` (`POUs/CoverControl/States/`,
  `POUs/PendantControl/States/`).
- Written against the step 3 tests; tests updated only where the behaviour is meant to change.
- Hydraulics: states roughly `Off`, `Starting` (suction/pressure build-up), `Running`, `BrakeOpening`,
  `BrakeOpen`, `Stopping`, `Error`; the four watchdog timers become per-state timeouts. #10 (140 s suction
  watchdog) and #11 (sensor polarity) should be settled before, so they are not re-implemented wrong.
- Cover/pendant: fixes #4/#5 on the way; checked by TcUnit only.
- Each MONET block gets a hardware-test issue (on site), like the FB_Axis command methods (#52).

## Open questions

1. MONETS/MONETN manual brake: is the cabinet switch meant as a level ("brakes open while switched on") or a
   request ("open once")? Today it is a level: after a pressure loss the block clears `bOpenBrake`, `MAIN`
   sets it again the next cycle, so the brake probably re-opens as soon as `tonBrakeOpenDelay` allows it
   (read from code, not tested). Is that wanted?
2. Should the axis command inputs move in BROTLib's `FB_BaseAxis` as well (fleet-wide, IAG50cm too), or only
   in HalfBROT's wrappers?
3. Struct layout: one struct per block, or inputs/outputs split (outputs then cannot be overwritten by the
   application)? And for the axes: the 4 `AT` variables sit in `FB_AxisControl`, so one struct type for all
   three axes?
