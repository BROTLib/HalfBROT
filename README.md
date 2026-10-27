# HalfBROT

HalfBROT is the **"BROTLib library for Halfmann telescopes"** — the hardware
abstraction layer of the BROT project for
[Halfmann](https://www.halfmann.com/) telescope mounts (TwinCAT 3, IEC 61131-3
Structured Text). It provides the function blocks that control
the mechanical subsystems of a Halfmann-mounted Alt-Az telescope: the azimuth,
elevation and derotator axes, focus, the three mirror covers, the hydraulic
pump/brake system, the manual hand pendant and auxiliary telemetry. The
MONET-class telescopes (MONETN / MONETS via MONETcommon) are built on this
layer, on top of the core BROTLib library.

> **Status:** the library is under active development. It is a standalone PLC
> project (`HalfBROT.tspproj`, no TwinCAT system part): no I/O, NC axes, tasks or
> safety project. The `AT %I*` / `AT %Q*` variables in the blocks are mapped by the
> consuming applications (MONETN, MONETS) in their own system projects.
>
> **Planned changes:** the blocks will stop using their inputs as internal state
> (commands only through methods), the `AT` I/O moves out of the library into
> structs, and each block gets an explicit state machine with TcUnit tests. See
> [the plan](specs/plans/2026-10-11-commands-state-machines-io.md) (#31, #32).

---

## Repository layout

```
HalfBROT/
├── HalfBROT.sln                  # TwinCAT solution
├── HalfBROT/
│   ├── HalfBROT.tspproj          # standalone PLC project (no system part)
│   ├── HalfBROT/
│   │   ├── HalfBROT.plcproj      # PLC library project (Company BROT)
│   │   ├── POUs/                 # Function blocks (see below)
│   │   ├── GVLs/                 # Global_Version
│   │   ├── VISUs/                # TwinCAT visualizations
│   │   │   ├── Azimuth.TcVIS, Elevation.TcVIS, Derotator.TcVIS
│   │   │   ├── Focus.TcVIS, Cover.TcVIS, Hydraulics.TcVIS
│   │   │   └── VisualizationManager.TcVMO
│   │   └── GlobalTextList.TcGTLO # Global text list (visu texts, format strings)
└── README.md
```

---

## Function blocks

| Function block | Description |
|---|---|
| `FB_AxisControl` | Abstract axis control — extends `FB_BaseAxis` (BROTLib), implements `I_Axis` |
| `FB_AzimuthControl` | Azimuth axis: brake interlock, rest detection, homing, limit switches |
| `FB_ElevationControl` | Elevation axis: cover/brake coordination on enable, 0–90° limits |
| `FB_DerotatorControl` | Field de-rotation axis with auto-homing |
| `FB_FocusControl` | Focus unit (Faulhaber motor): auto-positioning, brake lock, homing |
| `FB_CoverControl` | Three mirror covers, sequenced open 1-3-2 / close 2-3-1, timeout, reversal dead time |
| `FB_HydraulicsControl` | Hydraulic pump/brake system, oil monitoring, watchdog timers |
| `FB_PendantControl` | Manual hand pendant (BCD selector, 15 functions) |

### Axis control

The axis blocks (`FB_AzimuthControl`, `FB_ElevationControl`,
`FB_DerotatorControl`) extend `FB_AxisControl`, which extends BROTLib's
`FB_BaseAxis` (NC wrapper `FB_Axis2` over `MC_Power`/`MC_MoveAbsolute`/...) and
implements the `I_Axis` interface. Common features:

- **Brake interlock** — an axis may only be enabled when at rest or calibrated;
  the azimuth additionally requires the hydraulic brake open and standstill
  (|velocity| < 0.01 for 2 s); the elevation opens the mirror covers and the
  hydraulic brake on first enable and closes the brake on the falling edge of
  enable.
- **Homing** — azimuth/elevation home onto a calibration cam
  (`MC_SetAcceptBlockedDriveSignal` during homing, `fCalibPosition` per telescope);
  the derotator has an absolute encoder and only gets the calibrated flag
  (`MC_ForceCalibration`, no motion, position unchanged, `fCalibPosition` ignored);
  the focus (incremental encoder) takes the stored last position with `MC_Direct`
  (no motion) or, without one, homes onto the cam at `fHomingPosition`. The stored position is also written to
  disk (`FB_WritePersistentData`, port `nPersistentAdsPort` = 851) after it changed, at rest, at most once a
  minute, so an unclean power-off restores the last position written, not the one of the last clean shutdown (#8). After an SoE reset the axis
  must fully re-home.
- **Manual focus jog** — the stored position is tracked in every mode (calibrated, brake locked, not homing), so a focus
  jogged with the pendant in manual mode is restored at its new position after a restart (#66). Outside automatic
  mode and while a jog runs, the target `fPosition` follows the actual position, so the focus does not drive back to
  its old target when the telescope returns to automatic. A jog of an uncalibrated focus drops the stored position
  (cam homing next time).
- **Manual focus re-home** — `FB_FocusControl.bManualHome` (set TRUE online in XAE, by ADS or from a program; the block
  clears it) homes a calibrated focus again on its cam at `fHomingPosition`, e.g. when the stored position is in
  doubt. The stored position is dropped first, so the `MC_Direct` shortcut is not taken. It is accepted in automatic
  mode, calibrated, at rest, without `bManualUnlock` or error and with no homing running; otherwise a warning with the
  reason (`E_FocusManualHome`) is logged and nothing happens. Use it with the telescope idle: the re-home takes
  minutes, `Busy` holds back goto/slew/track/park, and if the NC drops `Homed` while homing, the telescope shows
  READY_STATE -2 until it is done (unverified on the telescope). Afterwards the focus returns to the target it had
  before. `Stop()` aborts a re-home; if that leaves the focus uncalibrated, the automatic homing stays off until the
  next `bManualHome` or another homing request.
- **No position restore** — azimuth and elevation home on their cam at every start. A restore with
  `MC_SetPosition` was removed (#7): it most likely never set `Calibrated`, and a mount moved while switched off
  must not trust an old position. Only the focus restores its last position (`MC_Direct`, see Homing).
- **Diagnostics and slew-time prediction** — the drive diagnostic word is
  translated via `NCError_TO_STRING` into `FB_EventLog` messages; the remaining
  slew time is read cyclically with `MC_ReadParameter`.

### Covers, hydraulics and pendant

- `FB_CoverControl` — three mirror covers, sequenced by limit switches: open 1->3->2, close 2->3->1, each cover
  starts when the previous one is at its end switch. All six outputs are recomputed every cycle; close has
  precedence over open; a cover in error (both switches active) or over `tMoveTimeout` (default 60 s, latched
  until `Reset()`) stops all covers and drops the command; a reversal waits `tReversalDeadTime` (1 s). Commands
  only through `Open()`, `Close()`, `Stop()`, `Reset()` (HalfBROT#4; the logic of MONETcommon's former
  `FB_MonetCoverControl`).
- `FB_HydraulicsControl` — oil pump + suction pump + the hydraulic brake for
  azimuth and elevation, with oil pressure/level/filter monitoring and four
  watchdog timers (suction 15 s, pressure 30 s, main pump 10 s, hydraulics
  140 s); 16-bit `statusWord` telemetry. The pump starters get a reset pulse of `tPumpResetPulse` (default 1 s, `T#0S` =
  old behaviour) on every reset, before the first pump start after a PLC start and before the first start after a main
  pump failure (`FB_PumpResetPulse`, HalfBROT#120); the pump enable is held off during the pulse. ADS writes of the reset
  outputs have no effect (written every cycle), only an XAE force does. A raw oil level outside `nMinValidPanLevel`..`nMaxValidPanLevel` (inputs, default:
  no limit) is a sensor fault (wire break, over-range; MONETS#32): `bOilLevelInvalid`, `statusWord` bit 14, one WARN
  log entry, `OilLevel` is not published, and the level neither starts nor stops the suction pump (only the float
  switch starts it; a running suction is released when the main pump is switched off). No error.
- `FB_PendantControl` — BCD-selector manual hand pendant with 15 functions
  (covers, focus, derotator, elevation, azimuth, dome, filter wheel, torque
  display, manual telescope control, horn (not wired), hydraulics, ...).
  Selection 0 (pendant disconnected) lights the error lamp, nothing else; the
  lamp otherwise shows the selected function's error. Selection 13 steps the
  pointing offsets by 10 arcsec per press in automatic mode and restarts
  TwinCAT after reset is held 3 s in manual mode only. Selection 1-3 with the key on: hold-to-run open/close of
  all three covers (single covers can no longer be moved). MONETS/MONETN use
  MONETcommon's `FB_MonetPendantControl` instead.
  *Warning: some safety routines are disabled when the telescope is operated
  manually.*

---

## Removed system part (safety, NC axes, I/O)

Until the commit tagged `pre-standalone-plc-project` this repository also held
a TwinCAT system project (`HalfBROT.tsproj`: EtherCAT I/O, NC axes, the
`PlcTask`, symbolic I/O mappings), a TwinSAFE project (`MONETNTwinSAFE`) and an
empty `MAIN`. It was removed because a library cannot carry a machine
configuration: the mappings of the `AT` variables, the NC axes and the safety
project live in the application repositories (MONETN, MONETS). The old system
part is still in git history under that tag.

## Dependencies

- **BROTLib** — `FB_BaseAxis`, `FB_Axis2`, `FB_AltAzTelescopeControl`,
  interfaces (`I_Axis`, `I_Focus`, `I_Brake`, `I_Hydraulics`,
  `I_MirrorCovers`), `FB_EventLog`, `E_TelescopeMode`, communication.
- Beckhoff system libraries: `Tc2_MC2`, `Tc2_MC2_Drive` (motion control),
  `Tc2_Standard`, `Tc2_System`, `Tc2_Utilities`, `Tc3_Module`, plus the TwinCAT
  visualization libraries (`VisuSymbols`, `VisuElems`, ...).

Library references resolve to the newest installed version (`*`); the
application that consumes HalfBROT decides which versions are actually built.

HalfBROT is consumed by **MONETcommon** (and through it MONETN / MONETS).

## Building and deployment

The library is built with TwinCAT 3.1 Build 4024.66 in TwinCAT XAE (PLC task
10 ms, priority 20; NC-Task 1 SAF 2 ms / SVB 10 ms). Solution platforms cover
TwinCAT RT (x64/x86), TwinCAT CE7 (ARMV7) and TwinCAT OS (ARM/x64). Releases are
cut by `.github/workflows/release.yml` (version bump on `develop`, release PR to
`main`, tag `vX.Y.Z`); each GitHub release carries the compiled
`HalfBROT_vX.Y.Z.library`.

**CI.** `.github/workflows/tcbuild.yml` builds `HalfBROT.sln` with [`tctool`](https://github.com/BROTLib/TcTool) on every push (self-hosted runner, never on pull requests). A green run means the project compiles; `tctool` exit code 1 (built with warnings) counts as success. It only compiles; no tests run in CI.
