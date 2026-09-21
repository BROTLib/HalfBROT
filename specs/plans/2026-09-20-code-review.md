# Code review of HalfBROT (develop @ 0566865)

**Status: draft. Review finished; the body below describes `develop` at the review commit. Since then the `FB_CoverControl` reset (#3, `91a0694`) was fixed on `develop`, and generated build outputs and the stale TwinSAFE `bak/` stopped being tracked (`d29b2b6`, partly addressing #25 and #26). Nothing here has been run on a PLC or a telescope. Open items: GitHub issues.**

Reviewed at `develop` 0566865. `origin/main` is 2 commits ahead (a TcBuild test workflow and its runner
labels). They matter only for the CI and release sections.

Scope: all 9 function blocks plus `MAIN` (~2300 lines of `.TcPOU` XML), `GVLs/Global_Version.TcGVL`,
`HalfBROT.plcproj`, `PlcTask.TcTTO`, `.github/workflows/release.yml`, the committed `.tsproj`, `.tmc`,
`_Boot/` and `MONETNTwinSAFE/` files, and `README.md`. **Not reviewed:** the six `.TcVIS` files (generated
XML, ~24k lines) and the TwinSAFE logic itself (see the Safety section). Consumers (MONETN, MONETS,
MONETcommon) and BROTLib were read only where a finding depended on them.

## How this was checked (and what that is worth)

- **Static read** of every POU, line by line.
- **Cross-repo checks** with `grep`, `diff` and `git rev-list` against BROTLib (`FB_Axis2`, `FB_BaseAxis`,
  `FB_TONTP`, the interfaces), MONETcommon, MONETN and MONETS.
- **Limit:** this is control code that talks to drives, valves and limit switches. There is no
  numerical reference to compare against (unlike AstroBROT), so nothing could be executed. A finding
  marked *read from code* is what the source says, not what a telescope did. TwinCAT-specific behavior
  (FB timing, `PERSISTENT` in function blocks, interface null calls) is marked **unsure** where I am not
  certain. Findings are marked **verified (code)** (the statement can be checked by reading or a grep/diff
  and I did so), **read from code**, or **unsure**.

## Where the blocks are used (matters for severity)

Read from `MONETN/.../MAIN.TcPOU` and `MONETS/.../MAIN.TcPOU`:

| HalfBROT block | Used by MONETN / MONETS |
|---|---|
| `FB_HydraulicsControl`, `FB_FocusControl`, `FB_DerotatorControl`, `FB_ElevationControl`, `FB_AzimuthControl` | yes, directly |
| `FB_TelescopeAuxiliary` | MONETS only |
| `FB_CoverControl` | **no**, both use `FB_MonetCoverControl` from MONETcommon |
| `FB_PendantControl` | **no**, both use `FB_MonetPendantControl` from MONETcommon |

Findings on the last two apply to the HalfBROT copy. Where the MONETcommon copy has the same code I say so.
IAG50cm has its own cover, focus and pendant blocks and was not looked at.

## Findings, worst first

### High

**H1. The drive-diagnostic test can never be true (azimuth, elevation, derotator).**
`IF inDiagnostic < 16#D012 AND inDiagnostic > 16#D014 THEN`: no value is both below D012 and above D014.
*Verified (code).* So `bDiagnosticError` is always FALSE, `diagnosticEvent` never logs, `nErrorID :=
inDiagnostic` never runs, and in the derotator the `bSoeReset := bReset` line inside the branch is dead too.
The axis error path through `fbAxis.Error` still works, so this loses a diagnostics channel silently, not the
error handling. I do not know which range was meant. `OR` would flag 0 (no diagnostic) as an error, so the
intent is probably "non-zero and outside D012..D014". **Unsure** whether `NCError_TO_STRING` knows AX5000
diagnostic numbers (they are a different number space from NC error codes). In production use.

**H2. Cover errors can never be reset, in HalfBROT and in the MONETcommon copy.**
`Reset()` does `Reset := TRUE`, which sets the method's own return value. `bReset` (the only input of the
`SR` error latch) is never assigned anywhere. *Verified (code)* in `FB_CoverControl` and in
`FB_MonetCoverControl` (`bReset` appears only at its declaration and in the `SR`). After one moment where a
cover's open and closed switch both read active (glitch, wiring fault), `Error` stays TRUE until the PLC
restarts, and the error event keeps firing. I did not check what the consumers do with `I_MirrorCovers.Error`.
Fix: `bReset := TRUE;` in `Reset()`, and a way to clear it again. MONETN and MONETS are affected through
MONETcommon.

**H3. `FB_CoverControl` outputs are only written inside the active branch, so they go stale.** *Read from code.*
The command block is `IF bClose THEN <close outputs> ELSIF bOpen THEN <open outputs> END_IF`, so nothing
resets the other direction. Scenario: closing is under way (`OabCloseCover[2]` TRUE), then `Open()` is called
(weather alarm, operator). `Open()` clears `bClose`, the `ELSIF` branch sets the open outputs, and the close
outputs stay TRUE for every cover that has not reached its closed switch. Both directions are then commanded.
What the DC driver does with that is **unsure**. Related in the same block:
- Nothing switches the outputs off on `bError`.
- No travel timeout: the three `afbTimeoutEvent` triggers are the constant `FALSE`, `bWarning` is never set.
  The outputs follow the raw limit-switch inputs (NC style), so a broken switch wire reads "not at limit" and
  keeps driving.
- The README says opening order 1, 3, 2 is "interlocked". In the code, on open covers 1 and 3 start
  together and only cover 2 waits (1 s after 3 leaves closed). On close, cover 3 waits a fixed 3 s after
  cover 2 leaves open, not for cover 2 to be closed.

MONETcommon's `FB_MonetCoverControl` computes every output every cycle from position conditions, so this
finding is HalfBROT-only.

**H4. `FB_PendantControl` leaves motion commands set when the selector or key changes.** *Read from code.*
`MovePos`/`MoveNeg` (elevation, azimuth, derotator, focus), `bStop`, `bPark`, `bGoto` and `bGoHome` are only
assigned inside the selected `CASE` branch, most of them also inside `IF key_switch`. The callee FBs keep the
last value when the pendant stops writing. Hold a direction button, turn the selector to another position
(or the key to automatic), and the axis keeps jogging with no operator input, as long as it is enabled.
The BCD selector is also not debounced, so a turn passes through intermediate codes. `FB_MonetPendantControl`
(the production one, sampled only) has the same shape and returns early when the key is off
(`IF NOT bEnable THEN RETURN`), so a key turned off while a direction button is held leaves the last
`MovePos`/`MoveNeg` in place. **Unsure** what else stops the axis (TwinSAFE E-stop, pendant hardware). Fix:
write every output on every call (default FALSE, then the selected branch), or reset all of them when the
selection or key changes.

### Medium

**M1. The azimuth can be enabled against a closed brake once it is calibrated.** *Read from code.*
`IF tonRestTimer.Q OR bCalibrated THEN bAtRest := TRUE ELSIF NOT bBrakeOpen THEN bAtRest := FALSE`, then
`Enable := bEnable AND bAtRest`. With `bCalibrated` TRUE, `bAtRest` never returns to FALSE, so a closed brake
does not block the enable. The comment says this is intended, the README says the brake must be open.
Elevation is different (`Enable := bEnable AND fbBrake.BrakeOpen`). In manual mode the pendant enables the
azimuth directly. The visible symptom would be the torque warning after 3 s. I did not check whether the
automatic sequence in BROTLib always opens the brake first.

**M2. The position-restore path may never set `Calibrated`. Unsure, test on the telescope.**
Azimuth and elevation restore the persisted position with `MC_SetPosition` (`fbAxisCalibration`). In BROTLib
`FB_Axis2`, `Calibrated := TRUE` is set in one place only, `IF HomeDone`, and `MoveAxis`, `Tracking` and the
setpoint enable are all gated on `Calibrated`. `MC_SetPosition` does not go through `Axis_Home`. So after a
restore I cannot find what makes `bCalibrated` TRUE, which the README relies on ("resumes without re-homing").
Also: the sentinel is `-1.0` and the tests are `fLastPosition >= 0.0`, but MONETN's azimuth range is -72 to
495, so a parked azimuth below 0 is never restored (safe direction, forces a re-home). Focus has the same
sentinel with `> 0.0`, so 0.0 is not restorable either.

**M3. Persistence assumptions. Unsure.** `fLastPosition` and `fLastPposition` are `VAR PERSISTENT` inside
function blocks. I am not certain TwinCAT 3 keeps FB-local persistent variables per instance (check the
Beckhoff Infosys page on the `PERSISTENT` keyword). By default persistent data is written on a controlled
shutdown, so a power cut may lose it (also check). And a restored position assumes the mount did not move
while unpowered: the azimuth comment says it can be moved "by the power chain", and there is no plausibility
check against a switch or the cam.

**M4. Brake state is a timer, not a measurement, and elevation disable is abrupt.** *Read from code.*
`bBrakeOpen` is the brake output plus 1 s, and `bBrakeClosed := NOT bBrakeOpen`, so "closed" is reported the
moment the output drops. There is no brake feedback input. `rsBrakeState` resets on pump-not-running, not on
loss of oil pressure, so with the pump running and pressure gone the brake is still reported open until the
30 s pressure watchdog trips. In `FB_ElevationControl` the falling edge of `bEnable` disables the drive
(`Enable := bEnable AND BrakeOpen`) and calls `CloseBrake()` in the same cycle, while the hydraulics
comment says the elevation is unbalanced outside the control loop. Whether a caller always stops the axis
first (BROTLib) I did not check.

**M5. Hydraulics: a watchdog that may trip in normal operation, and comments that disagree with the code.**
*Unsure.* `tonHydraulicsWatchdog` raises `bHydraulicsFailure` (latched error, pump stops) when the main pump
runs 140 s with the suction pump not running. Suction starts only when the pan is full
(`srSuctionTrigger`) or manually. Either the suction pump runs by another path, or this trips in normal use.
Also: the header says the suction pump runs 60 s and the main pump stops after 10 s of pressure failure. The
code has no 60 s timer and the pressure watchdog is 30 s. The suction latch resets at
`fOilLevel < fMinPanPercent`, default 0 %, so it only clears below the sensor minimum (MONETN sets
`nMinPanLevel := 3000`, so it means "below calibrated empty"). The FB defaults (0 / 16383 / 0 % / 90 %) are
not usable values, consumers override them.

**M6. Hydraulics sensor polarity is mixed.** *Read from code, wiring not available.* Inverted:
`bOilLow`, `bOilHigh`, `bOilFilterDirty`, `bOilCold`. Not inverted: `bOilHot`, `bOilWarning`,
`bOilPressureOK`, `bOilPanMaximum`. Cold inverted and hot not is suspicious for two temperature switches.
A broken wire on `IbOilpanMaximum` reads "not full", so the pump keeps running and suction never starts.
`bOilLow` and pressure fail in the safe direction. `IbOilHigh` is commented `?` and `bOilHigh` "unused?"; if
the input is not wired, `bOilHigh` is permanently TRUE and bit 1 of `statusWord` is stuck.

**M7. Pendant: safety-relevant behavior that is easy to trigger.** *Read from code, HalfBROT copy only.*
- Selection 13 restarts TwinCAT (`TC_Restart`, `NETID := ''`) after the reset button is held 3 s and the
  suction pump is stopped. It sits outside `IF key_switch`, so it also works in automatic mode.
- Selection 13 with the key off changes `fElevationOffset`/`fAzimuthOffset` by 10" on every call while a
  direction is held (the `up_trigger`/`down_trigger` edges are declared and unused). At the 10 ms task cycle
  that is about 1000"/s (0.28 deg/s). **Unsure** whether hold-to-nudge at that rate is intended.
- `Error := (Selection = 0)` (pendant disconnected) is computed and never used, the README says it is
  "treated as an error". `horn` has no `AT %Q` mapping, so the horn never sounds. `lamp_error := lamp_error`
  is a no-op and the lamp is never cleared, so it stays lit after leaving a selection with an error.
  MONETcommon's rewrite keeps the same `bError` and `ObLampError := ObLampError` lines (sampled).

**M8. The next release would fail half way, because `main` is ahead of `develop`. Resolved 2026-09-20: `origin/main` was merged into `develop` (71dab61) and pushed.**
*Verified (code).* `git rev-list --left-right --count origin/main...origin/develop` gives `2 0`.
`release.yml` pushes the version bump to `develop`, then runs `git merge --ff-only develop` on `main`.
`main` has 2 commits `develop` lacks, so the merge fails, the job stops with the bump on `develop`, `main`
not updated and no tag. This is the same situation AstroBROT had (its M8). Merging `main` into `develop` fixed the
divergence (`rev-list` now gives `0 2`).
Nothing stops it from recurring when `main` gets commits directly. I did not run the workflow.

### Low

- **L1. Copy-paste across the three axes.** SoE reset, torque monitoring, `MC_ReadParameter`, diagnostics and
  calibration restore exist three times with small differences (which is how H1 spread). The azimuth SoE
  block calls `fbSoEReset(Execute := FALSE)` while `Busy`, elevation and derotator do not, so the azimuth
  one would re-trigger the reset each cycle if `Busy` stays TRUE. In all three, `Done`/`Error` are never read and an error is
  treated as success (`bReset := TRUE`). **Unsure** how `FB_SoEReset` behaves on the first call. Open PR #1
  (`feature/fb-baseaxis-unification`) moves `bSoEReset`/`fbSoEReset`/`axisRef` into the base class, so
  re-check after it merges.
- **L2. Dead code.** `homeDelay` (`FB_TONTP`) in `FB_AzimuthControl` is never called, so the block that sets
  `fPosition := fCalibPosition; bMoveAxis := TRUE` never runs (*verified (code)*, grep). Also: `fbOffTrigger`
  in hydraulics, `IbSTO` in `FB_AxisControl` (the STO state is never consulted), `bWarning` in covers,
  `GVL_Telescope` in the pendant, and `bCalibrated := FALSE` right before `bCalibrated := fbAxis.calibrated`
  in focus.
- **L3. Unchecked references.** Elevation takes `fbCovers`/`fbBrake` as `VAR_INPUT` interfaces and the pendant
  takes `REFERENCE TO` inputs. Only the focus case in the pendant tests `__ISVALIDREF`. A null call ends in a
  task exception (**unsure** of the exact Beckhoff behavior). MONETN/MONETS pass everything every cycle, so
  this bites a new telescope (no derotator, no focus). `FB_TelescopeAuxiliary` calls `comm.Publish` after 5 s,
  so a missing `comm` fails late, not at start.
- **L4. Defaults that can be forgotten.** `fCalibPosition := 45.0` on all axes. MONETN's derotator call does
  not set it, so 45.0 applies (check it is intended). `FB_ElevationControl` inherits `fMinPosition`/`fMaxPosition`
  (default 0/450) and ignores them (hard-coded 0 and 90).
- **L5. Focus.** While homing with a saved position two calibrations run at once (`MC_ForceCalibration` through
  `fbAxis` and `MC_SetPosition`), and `fbSetPosition.Done/Error` are not read. `fPosition` is overwritten
  during homing, so a target set by the caller is lost. The limit switches are active-high (`IbLimitFar` TRUE =
  at limit), the opposite of azimuth and elevation, so a broken wire reads "no limit" (**unsure** of the sensor
  type). Magic number `92.02` (`104.02 - 12.00`), also passed by MONETN.
- **L6. Telemetry.** `FB_HydraulicsControl` publishes 14 messages in one cycle at every `statusWord` change
  (contact bounce gives bursts), unmeasured cost on the 10 ms task. `tonComm : TON := (PT := fTelemetryInterval)`
  initializes from the input's default, so setting the input probably has no effect (**unsure**, evaluation
  order). Comments say "every second", the interval is 5 s. Focus `POWER_STATE` is a constant `'1.0'`.
- **L7. Derotator limits.** The hardware limit inputs (`inDigitalInputs.0/.1`, active-high here, active-low
  on the other axes) are used only by the pendant. In automatic operation the derotator relies on software
  limits from `ActualPosition`, and `bStopAxis := TRUE` is re-asserted every cycle while outside the range.
  **Unsure** whether that blocks homing or jogging back in.
- **L8. Analog scaling** in `FB_TelescopeAuxiliary`: `raw / 32767 * 150 - 50` maps 0..32767 to -50..100 C with
  no wire-break or range check. **Unsure** of the terminal type.
- **L9. Naming and comments.** `FB_Eventlog`/`FB_EventLog`, `fLastPposition`, `bAzimutLimitSwitch`, the
  `InTorque` comment says "derotator" but all three axes use it, the elevation comment says `ActVelo>1` and the
  code uses 2.0, `bCloseBrake` is commented "open the brake". Mixed `snake_case` and Hungarian in the pendant.

## Security and repository hygiene

Nothing here is exploitable remotely (no network code beyond publishing through `I_Comm`). Items are hygiene.

- **S1 (Low). Script injection in `release.yml`.** The "Validate version format" step interpolates
  `${{ inputs.version }}` into the shell command, so a `$(...)` payload runs before the regex. Only users who
  can already dispatch the workflow (write access, `contents: write`) can use it. Same as AstroBROT S1. Fix:
  pass it through `env:` and read `$VERSION`.
- **S2 (Low-Medium). The release job pushes to `develop` and `main` and tags with no build or test gate**,
  and the release runs on `ubuntu-latest` where TwinCAT cannot build. `actions/checkout@v4` is pinned by tag,
  not SHA. With M8 this is how a broken release gets half published.
- **S3 (Low). Network identifiers in a public repo** (checked with `gh`, the repo is public). `.tsproj`,
  `.tsproj.bak` and the safety `TargetSystemConfig.xml` contain ADS/AMS NetIds (IP-based addresses, not
  repeated here), the EL6910 serial number and FSoE address. *Not verified* whether the address is routable.
  Text files show no credentials (grep for `password`, `secret`, `token`, `api key` across tracked files:
  only Beckhoff visualization dialog names and a `.gitignore` comment). They stay in git history.
- **S4 (Low). Stale duplicates are tracked:** `HalfBROT.tsproj.bak` (343 KB), `MONETNTwinSAFE/.../bak/`, and
  `Visualization Manager.TcVMO`, which the `.plcproj` does not reference (it references
  `VisualizationManager.TcVMO`). Some files were saved by TwinCAT 3.1.4026.8 (`FB_TelescopeAuxiliary`, the
  orphan `.TcVMO`), the rest by 4024.15, and the project targets 4024.66.

### Safety project (`MONETNTwinSAFE/`), not assessed, two observations

I am not qualified to sign off the TwinSAFE logic and did not try. What the files show:

- **This is a second, older copy of MONETN's safety project.** `MONETN/MONETN/MONETNTwinSAFE` has the same name
  but different content: `ProjectCRC` 9657 here vs 32006 there, application size 76 KB vs 94 KB, and the MONETN
  copy has the roof safety alias devices and (per its git log) "EDM for roof and mirror covers", which this copy
  lacks. If someone loads this copy onto an EL6910, the safety function set is wrong. *Verified (code)* for the
  differences, not for which one is deployed. A library repo should not carry it, or it should be marked as
  a reference snapshot.
- **`Restart`, `Run` and `ErrorAcknowledgement` are standard alias devices** (non-safe bits, `Type 1`). The
  E-stop restart input is therefore a PLC bit. Whether it comes from a physical reset button or from software
  is not visible here. The safety officer should check this against the manual-reset requirement of the
  applicable standard (I have not looked up the clause). Passivation and deactivation flags are all off,
  which is the conservative setting. All ports have `maxDeviation="0"`; **unsure** whether that means no
  discrepancy monitoring.

## Design assessment

1. **Copies instead of one library.** `FB_HydraulicsControl` exists three times (HalfBROT, MONETcommon, and
   MONETN/`Components`, all with the same POU GUID, so they are copies of one file). Cover, pendant and focus
   have MONETcommon versions too. They have already drifted: the `bCloseBrake` latch exists only in HalfBROT
   (MONETcommon closes when `bOpenBrake` drops), the covers were rewritten in MONETcommon, the pendant was
   rewritten. A bug fixed in one copy stays in the others (H2 is in two). Pick the canonical blocks and delete
   the rest, or the review effort is spent three times.
2. **The library carries a system.** The repo has a `.tsproj` with I/O, NC axes and six unrestored symbolic
   links, a TwinSAFE project, a boot folder and a `.tmc` with no HalfBROT symbols. The README says so
   ("under active development"). Blocks with `AT %I*`/`AT %Q*` inside a reusable library force the consumer to
   map every instance and make unit tests impossible. Pass I/O in through inputs, and keep hardware mapping
   in the machine projects (MONETN already owns its own).
3. **Inputs used as state.** `bEnable`, `bOpenBrake`, `bHomeAxis`, `bMoveAxis`, `fPosition` are `VAR_INPUT`
   and are written inside the blocks. A caller that assigns them every cycle (MONETN does with
   `HydraulicsControl.bOpenBrake := brakeClearing`) fights the block. Same class as AstroBROT M1.
4. **State in booleans.** Behavior is spread over latched flags cleared by completion flags (H3, H4, M2 come
   from this). IAG50cm already has explicit state FBs (`FB_CoverState`, `FB_PendantState`). An enum state and a
   single place that writes each output would remove the stale-output class of bug.
5. **No tests.** Neither TcUnit tests nor a simulation. The cover sequencing and the hydraulics state machine
   are pure logic and would be testable once I/O comes in through inputs (see 2).
6. **README vs code.** Claims that do not match: brake required for azimuth enable (M1), interlocked cover order
   (H3), pendant disconnect is an error (M7), resume without re-homing (M2), dependencies `Tc2_EtherCAT` and
   `Tc3_IotCommunicator` (neither is referenced in `HalfBROT.plcproj`), and "AstroBROT, used by the telescope
   control" (no POU in this repo uses it, *verified (code)* by grep). The AstroBROT placeholder can be dropped.
   BROTLib and AstroBROT are floating (`*`), so builds are not reproducible.
7. **The hydraulics block** is the most consistent one: RS/SR/TON logic with one event per watchdog. Its
   problems are semantics (M4, M5, M6), not structure.

## CI and release

Verified from the repo and `gh`:
- `tcbuild-test.yml` (`workflow_dispatch` only, self-hosted `[self-hosted, twincat, windows]`, `TcBuild build
  HalfBROT.sln`) exists on `main`, not on `develop`. Its one run succeeded on 2026-09-16 (on `main`). So
  compile checks work, but CI has never built `develop`, and it runs no tests.
- Unlike AstroBROT there is **no committed `.library`**, so the stale-binary problem (AstroBROT H4) does not
  exist here. `<Released>false` and `release.yml` only bumps `ProjectVersion` and `Global_Version` and tags.
- Runner setup, TcBuild exit codes, and why numeric or runtime tests are an open question are covered in the
  CI section of `../AstroBROT/specs/plans/2026-09-20-code-review.md` and are not repeated.

Recommendation: add the TcBuild job to `develop` (the merge for M8 brought the workflow file over), make the release job
depend on a successful build, and fix S1 in the same change.

## Suggested order of work

1. Small, certain fixes: H1 (diagnostic condition), H2 (`bReset := TRUE`), S1, L2 dead code (M8 is resolved).
2. Decide the canonical copies (design point 1) before fixing H3, H4 or M7 anywhere, otherwise they get fixed
   in a block nobody runs. H4 also applies to the MONETcommon pendant that MONETN/MONETS do run.
3. Test M2 on the telescope (power-cycle with a saved position, watch `Calibrated`), and check M1 and M4
   against the BROTLib automatic sequence.
4. Ask whoever knows the hardware about M5 and M6 (suction pump behavior, sensor wiring).
5. Move I/O mapping out of the blocks, add tests for cover sequencing and the hydraulics state machine.
6. Have the safety copy (S3 and the Safety section) removed or clearly labelled, and the restart source checked
   by the person responsible for the safety function.
