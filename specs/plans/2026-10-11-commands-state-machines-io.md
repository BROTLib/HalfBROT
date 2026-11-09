# Commands through methods, explicit state machines, I/O through structs

**Status: draft.** Nothing implemented. Updated 2026-11-09: the three open questions are decided (see
"Decisions"), steps 1–3 adjusted. Updated 2026-11-06: VISU bindings added to the current state and to step 1.
Facts below were read from `develop` @ `4ef3b7b` (v0.5.1) and the consumers' `develop` on 2026-10-11 (`AT`
counts re-checked on `develop` 2026-11-09); nothing was run on a PLC or a telescope.

Issues: #31 (D3, umbrella), #32 (D4, tests). The bug class it removes: #4 (cover stale outputs), #5 (pendant
stale motion commands), and it touches #7 (position restore) and #20 (derotator stop re-asserted).

## Problem

1. **Inputs used as state.** The blocks write their own `VAR_INPUT`s, so a caller that assigns an input every
   cycle overrides what the block just decided (e.g. the block drops `bOpenBrake` on pressure loss, the caller
   sets it again next cycle).
2. **Behaviour in latched booleans.** Each operation is a flag that some completion or error clears in another
   place. Forget one clear path and an output stays set (#4, #5).
3. **I/O declared inside reusable blocks.** 43 `AT %I*`/`%Q*` variables in five blocks, mapped by each
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

`bEnable`, `bReset`, `bHomeAxis`, `fPosition` of the axis blocks are inherited from BROTLib's `FB_BaseAxis`
(inputs `bEnable`, `bReset`, `bSoEReset`, `bHomeAxis`, `fPosition`, `fVelocity`, `bMoveNeg`, `bMovePos`; it
already has the properties/methods `Enable`, `HomeAxis`, `Position`, `Velocity`, `MoveNeg`, `MovePos`,
`Reset()`). Besides the MONET axes, IAG50cm's `FB_AxisControl`, `FB_DeclinationControl` and
`FB_HourAngleControl` extend it.
Some writes are property/method setters (`MoveAxis`, `StopAxis`, `Tracking`, `OpenBrake()`, `CloseBrake()`,
`Enable`), i.e. the public API already exists but only forwards into the input.

### Who writes these inputs from outside

| Caller | Write | Note |
|---|---|---|
| MONETS `MAIN` (l. 82), MONETN `MAIN` (l. 79) | `HydraulicsControl.bOpenBrake := brakeClearing` | every cycle in manual/off mode; level semantics from the cabinet "brakes cleared" switch |
| MONETcommon `FB_MonetTelescopeControl` (l. 403–405) | `fbElevation/fbAzimuth/fbDerotator.bTracking := FALSE` | |
| MONETS/MONETN `VISUs/*.TcVIS` (identical in both) | 24 element bindings to block inputs, see below | operator pages: DipSwitches write, lamps read |
| MONETcommon `FB_MonetPendantControl` (l. 101–102) | `fbCovers.bOpen/bClose` | on `FB_MonetCoverControl` (MONETcommon), **not** HalfBROT's |

Manual brake today (read from code, not tested): `brakeClearing` (the cabinet "brakes cleared" switch) is also
passed as the block's `bBrakeClearing` input, so the switch both requests and permits. `FB_HydraulicsControl`
clears `bOpenBrake` on pressure loss (2 s), stopped pump or `CloseBrake()`; `MAIN` sets it again next cycle;
`rsBrakeState` sets when `tonBrakeOpenDelay` (3 s pump running with pressure OK) AND `bOpenBrake` AND
`bBrakeClearing`. So the brake probably re-opens by itself after pressure returns while the switch stays on.

### Visualisation bindings (read from the `.TcVIS` files on `develop`, 2026-11-06; not opened in XAE)

MONETS and MONETN bind VISU elements directly to block inputs (`MAIN.<Block>.<input>`). A visualisation cannot
reach a `VAR` member or call a method, so every input that step 1 moves into `VAR` breaks its binding (compile
error in XAE) unless the binding is moved first.

| Page | Bound inputs |
|---|---|
| `Hydraulics` | `bEnable`, `bReset`, `bOpenBrake`, `bCloseBrake`, `bManualSuction`, `bManualPumpStop` |
| `Azimuth`, `Elevation`, `Derotator` | `bEnable`, `bReset`, `fPosition`, `bMoveAxis` (each) |
| `Focus` | `bEnable`, `bReset`, `fPosition`, `bMoveAxis` (lamp, read only), `bManualUnlock` |
| `Cover` | `bReset` (of `MAIN.CoverControl`) |
| `Telescope` | `MAIN.TelescopeControl.bTracking` (MONETcommon block, not HalfBROT) |

Which of these are DipSwitches/buttons (write) and which lamps/text (read) has to be checked per element.
`bEnable`, `bReset`, `fPosition` of the axis blocks are BROTLib `FB_BaseAxis` inputs and stay inputs in this
work (decision 2), so their bindings are untouched.

### Who uses which block

- MONETS, MONETN: `FB_HydraulicsControl`, `FB_FocusControl`, `FB_AzimuthControl`, `FB_ElevationControl`,
  `FB_DerotatorControl` (instances in `MAIN`, references in MONETcommon).
- HalfBROT `FB_CoverControl`, `FB_PendantControl`: no IAG telescope instantiates them (MONETS/MONETN use
  MONETcommon's `FB_MonetCoverControl`/`FB_MonetPendantControl`). They stay in the library (#29) and get the
  same treatment as the other blocks; external users (Potsdam) are not a constraint (decided 2026-10-11).
  Consequence: the pendant can only be checked by TcUnit (steps 3 and 4), never on an IAG telescope, and has no
  I/O links in MONETS/MONETN, so step 2 costs no re-mapping for it. The covers are linked once MONETS/MONETN
  use HalfBROT's `FB_CoverControl` (see step 2).

### `AT` declarations

`FB_HydraulicsControl` 16 (11 in, 5 out), `FB_PendantControl` 16 (11 in, 5 out), `FB_CoverControl` 4 (2 in,
2 out, each `ARRAY[1..3] OF BOOL`, i.e. 12 links), `FB_AxisControl` 4 (all in: digital inputs word, diagnostic,
torque, STO), `FB_FocusControl` 3 (2 in, 1 out). A plain text search finds 17 in the pendant and 5 in the
cover file: one match each is a comment. `FB_TelescopeAuxiliary` (2) was removed in #21 (v0.5.2).
MONETS/MONETN map them in `MONETS.tsproj`/`MONETN.tsproj` (by symbol path, e.g.
`MAIN.HydraulicsControl.<var>`).

## Plan

Order of work remains steps 1 to 4. Steps 1 and 2 are breaking and have no test safety net until step 3, which
is why each is its own PR and release with an XAE check by Tim (steps 3 and 4 are also one PR and release
each). Block order inside each step: hydraulics first
(MONET-only, most latches, the per-cycle caller), then focus and the three axes, cover and pendant last (not on
any IAG telescope, so only testable with TcUnit).

### Step 1: commands only through methods (small change in MONETcommon/MONETS/MONETN)

- Move the command inputs a block writes itself into `VAR` (internal request flags); keep the existing
  methods/properties (`OpenBrake()`, `CloseBrake()`, `Enable`, `Reset()`, `MoveAxis`, `StopAxis`, `Tracking`)
  as the only way in. For the axis blocks, check first what BROTLib v0.7.0's `FB_Axis` command methods already
  cover, so HalfBROT does not duplicate them.
- Axis wrappers (decision 2): HalfBROT only. The inherited `FB_BaseAxis` inputs stay inputs; the wrappers write
  them only through the base properties/methods (`Enable`, `HomeAxis`, `Position`, `Reset()`, ...), never by
  assignment. No BROTLib release and no IAG50cm change in this work.
- `fPosition`: the block keeps its own clamped target in `VAR` and passes it through `Position` instead of
  overwriting the caller's input.
- Manual brake (decision 1): `FB_HydraulicsControl` gets a method/property that takes the cabinet switch level
  each cycle. The block detects the rising edge and opens the brake once; after the brake closed for any reason
  (pressure loss, pump stop, `CloseBrake()`, azimuth limit) it stays closed until the switch goes off and on
  again. The permission part stays: switch off closes the brake via `bBrakeClearing`. MONETS/MONETN `MAIN`
  replace `bOpenBrake := brakeClearing` with this call; MONETcommon replaces `.bTracking := FALSE` with the
  property.
- Visualisations (MONETS and MONETN `VISUs`): every HalfBROT-owned input that moves into `VAR` loses its binding
  (hydraulics `bEnable`, `bReset`, `bOpenBrake`, `bCloseBrake`, `bManualSuction`, `bManualPumpStop`; focus
  `bMoveAxis`, `bManualUnlock`; axis `bMoveAxis`). Order: (a) `MAIN` gets one variable per operator switch
  (e.g. `bManualBrakeOpen`, `bManualPump`) and passes it to the block's method every cycle; (b) the DipSwitch
  or button is re-bound to it; (c) lamps bind to read-only status properties or outputs of the block; only then
  (d) the input moves into `VAR`. Do (a)+(b) in a first MONETS/MONETN release that still works with the old
  library, so the library change itself needs no VISU edit. The `Hydraulics` DipSwitch bound to
  `HydraulicsControl.bOpenBrake` is re-bound to the `MAIN` variable that feeds the brake call above.
- Test for the VISU part: Tim opens every VISU page of both telescopes in XAE (no unresolved variable) and
  operates each switch once on the bench with the PLC in manual mode.
- Release: HalfBROT minor (inputs removed from the interface), MONETcommon/MONETS/MONETN bumps.
- Test: XAE build of HalfBROT, MONETcommon, MONETS, MONETN (Tim); hardware-test issue for the brake in manual
  mode on site: opens by switch; drops on pressure loss; is **not** re-opened while the switch stays on;
  re-opens after the switch goes off and on. The brake change is visible to operators and safety-relevant:
  Tim confirms the behaviour is wanted before release.

### Step 2: I/O through structs, no `AT` in the library (breaking: re-linking in MONETS/MONETN)

Decided 2026-10-11 over the alternative (keep `AT` in the blocks, route it through overridable
`ReadInputs()`/`WriteOutputs()` methods so a derived test block can feed values): moving the I/O out keeps
hardware out of the library (#30), makes the blocks directly testable, and needs no extra indirection. The
risk of a silently lost link is covered by checking all links in XAE before activation.

- Struct layout (decision 3): hydraulics, pendant, cover and focus get an `ST_<Block>In` passed as
  `VAR_INPUT` (copied, so the block cannot change it) and an `ST_<Block>Out` passed as `VAR_IN_OUT` (the
  application-owned global struct, linked to the terminals, is written by the block). The axes share one
  `ST_AxisIn` (azimuth, elevation, derotator; all four `AT` variables are inputs), no out struct. The
  application can still write a global Out struct: convention, not enforcement.
- The `AT` declarations move to an I/O GVL in each application (MONETS/MONETN, e.g. `GVL_IO.stHydraulicsIn`),
  linked there.
- Re-linking: MONETS and MONETN only, per telescope hydraulics 16, focus 3, axes 4 each, covers 12; HalfBROT's
  pendant is not linked there. **The covers are linked** (`MAIN.CoverControl.*`) once MONETS/MONETN use
  HalfBROT's `FB_CoverControl`, see `2026-10-12-cover-control-single-copy.md` (correction of 2026-10-12, they
  cost a re-link in step 2). A script rewrites the `VarA` symbol paths of the `<Link>` entries in
  `MONETS.tsproj`/`MONETN.tsproj` (`PlcTask Inputs^MAIN.HydraulicsControl.IbOilLow` → new path) and checks
  that the link count per block is unchanged.
- Before activation: Tim checks in XAE that every new I/O variable is linked and nothing is left unlinked.
- After activation: read the I/O values over ADS before and after the update and
  compare (same telescope state, e.g. hydraulics off, covers closed).
- Release: HalfBROT minor, MONETcommon (if its references change), MONETS/MONETN bumps.

### Step 3: TcUnit tests of the current behaviour (#32)

- Already there (2026-11-09): the pure-logic helpers of HalfBROT are tested in `MONETcommonTests`, not in a
  HalfBROT project: `FB_FocusJogLatch`, `FB_FocusManualHome`, `FB_FocusTargetInit`, `FB_HomeRequest`,
  `FB_JogRequest` and the `F_*` functions, plus `FB_BrakeSettleEnable`, `FB_FocusDriveEnable`,
  `FB_HomeFailPosition` and `FB_PumpResetPulse` (MONETcommon#227, one suite `FB_HalfBROTHelpers_Tests`; CI:
  35 suites, 306 tests, 0 failed). What is left for this step are the blocks with `AT` (hydraulics, axes,
  cover, pendant, focus) and `FB_Hydraulics`' state.
- Limit found 2026-11-09: `MONETcommonTests` loads on the user-mode runtime with at most 35 suites (36, 37 and 38
  gave `PLC: Error (hr=0x98410033) loading boot project`); the build shows ~70 MB global data for 35 suites,
  about 2 MB per suite instance, probably the limit (not confirmed). The TcUnit parameters say 41 suites. So
  tests for the blocks do not fit as new suites in `MONETcommonTests` (34 + 1 = 35 used): they go into a
  `HalfBROTTests` project, or `MAXNUMBEROFASSERTSFOREACHTESTSUITE` (256) is lowered first to make room.
  Also: do not run a manual XAE activation while CI runs, both use the same runtime and boot folder.

- Started 2026-11-09 (PR "feat/halfbrot-tests-32"): `HalfBROTTests` exists with `FB_HydraulicsControl_Tests` (start-up, the brake, the
  step 1 re-arm, the latched errors up to 13 s) and `FB_CoverControl_Tests` (sequence, last call wins, stop, limit switch fault,
  timeout, reversal dead time); CI builds and runs it through `tcbuild.yml`. Pendant added 2026-11-09 (`FB_PendantControl_Tests`: selector,
  error lamp, unset references #16, covers, hydraulics start/brake; focus, derotator, elevation, azimuth and telescope selections only as "not
  connected"). Left: axes and focus: they run MC blocks on an NC axis (`AXIS_REF`) and `HalfBROTTests` has no NC axis, so they need a
  simulated NC axis first (not tried: unknown whether the CI runtime supports it). The 15 s suction and 30 s pressure watchdogs need a time
  seam (step 4: per-state timeouts).
- `HalfBROTTests` project like `MONETcommonTests`, run by Tim in XAE and by CI through `tcbuild.yml` like `MONETcommonTests`. Possible for every
  block once step 2 is done (I/O is just struct members): a test sets the `In` struct and reads the `Out`
  struct.
- Characterisation tests first: pin down what the blocks do today, including the known bugs (#4, #5) as
  expected-failure or documented cases, so step 4 shows exactly what changes.
- First targets: hydraulics (start-up, pressure loss closes the brake, watchdog timeouts, reset from error),
  cover sequencing 1→3→2 / 2→3→1, and the step 1 brake re-arm: switch rising edge opens once; after pressure
  loss, pump stop or `CloseBrake()` the brake stays closed while the switch stays on; off/on opens it again;
  switch off closes it.

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

## Decisions

Accepted by Tim, 2026-11-09.

1. **Manual brake: request with re-arm.** The cabinet switch level is passed each cycle through a method of
   `FB_HydraulicsControl`; the rising edge opens the brake once, and after any close it stays closed until the
   switch goes off and on. Reason: today the brake probably re-opens by itself after a pressure loss while the
   switch stays on (read from code, not tested). Changes: step 1 (method, `MAIN` call, VISU re-binding,
   hardware-test issue, Tim's confirmation before release), step 3 (TcUnit test of the re-arm).
2. **FB_BaseAxis inputs: HalfBROT only.** The wrappers keep their clamped target in `VAR` and write the base
   axis only through its properties/methods. Reason: moving the `FB_BaseAxis` inputs into `VAR` would be
   breaking for IAG50cm and for every VISU binding to `bEnable`/`fPosition`/`bReset`; it becomes a separate
   BROTLib issue for later. Changes: step 1 (axis wrappers), VISU axis bindings stay.
3. **Struct layout: in/out split.** `ST_<Block>In` as `VAR_INPUT`, `ST_<Block>Out` as `VAR_IN_OUT` for
   hydraulics, pendant, cover and focus; one shared `ST_AxisIn` for the three axes. Reason: the block cannot
   change its inputs, and tests set `In` and read `Out` (the application can still write an `Out` struct, so
   this is convention, not enforcement). Changes: step 2 (structs, I/O GVL per application, link counts),
   step 3 (test pattern).
