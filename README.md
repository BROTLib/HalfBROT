# HalfBROT

HalfBROT is the **"BROTLib library for Halfmann telescopes"** — the hardware
abstraction layer of the BROT project for
[Halfmann](https://www.halfmann.com/) telescope mounts (TwinCAT 3, IEC 61131-3
Structured Text, version 0.4.0). It provides the function blocks that control
the mechanical subsystems of a Halfmann-mounted Alt-Az telescope: the azimuth,
elevation and derotator axes, focus, the three mirror covers, the hydraulic
pump/brake system, the manual hand pendant and auxiliary telemetry. The
MONET-class telescopes (MONETN / MONETS via MONETcommon) are built on this
layer, on top of the core BROTLib library.

> **Status:** the library is under active development — `MAIN` is an empty
> skeleton and the compiled module description contains no HalfBROT function
> blocks yet (the `.tsproj` still records unrestored symbolic I/O links
> leftover from the fuller application this library was extracted from).

---

## Repository layout

```
HalfBROT/
├── HalfBROT.sln                  # TwinCAT solution
├── HalfBROT/
│   ├── HalfBROT.tsproj           # TwinCAT system project (I/O, NC, tasks, mappings)
│   ├── HalfBROT/
│   │   ├── HalfBROT.plcproj      # PLC library project (Company BROT, v0.4.0)
│   │   ├── PlcTask.TcTTO         # PLC task (10 ms, priority 20, calls MAIN)
│   │   ├── POUs/                 # Function blocks (see below)
│   │   ├── VISUs/                # TwinCAT visualizations
│   │   │   ├── Azimuth.TcVIS, Elevation.TcVIS, Derotator.TcVIS
│   │   │   ├── Focus.TcVIS, Cover.TcVIS, Hydraulics.TcVIS
│   │   │   └── Visualization Manager.TcVMO
│   │   └── GlobalTextList.TcGTLO # Global text list (visu texts, format strings)
│   ├── MONETNTwinSAFE/           # TwinSAFE safety project (FSoE over EtherCAT)
│   └── _Boot/                    # Boot project for TwinCAT RT
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
| `FB_CoverControl` | Three mirror covers, sequenced open/close |
| `FB_HydraulicsControl` | Hydraulic pump/brake system, oil monitoring, watchdog timers |
| `FB_PendantControl` | Manual hand pendant (BCD selector, 15 functions) |
| `FB_TelescopeAuxiliary` | Mirror temperature sensors (BT9/BT10) |

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
  (`MC_SetAcceptBlockedDriveSignal` during homing); the derotator homes into
  its hardware end position (`MC_ForceCalibration`); the focus homes to the
  stored last position or a fixed homing position. After an SoE reset the axis
  must fully re-home.
- **Calibration restore** — `MC_SetPosition` re-applies the persistent last
  position after a restart, so the mount resumes without re-homing.
- **Diagnostics and slew-time prediction** — the drive diagnostic word is
  translated via `NCError_TO_STRING` into `FB_EventLog` messages; the remaining
  slew time is read cyclically with `MC_ReadParameter`.

### Covers, hydraulics and pendant

- `FB_CoverControl` — three mirror covers, sequenced by limit switches:
  opening order **1 → 3 → 2**, closing order **2 → 3 → 1** (interlocked, close
  takes precedence); a cover is in error when both open and closed signals are
  active.
- `FB_HydraulicsControl` — oil pump + suction pump + the hydraulic brake for
  azimuth and elevation, with oil pressure/level/filter monitoring and four
  watchdog timers (suction 15 s, pressure 30 s, main pump 10 s, hydraulics
  140 s); 16-bit `statusWord` telemetry.
- `FB_PendantControl` — BCD-selector manual hand pendant with 15 functions
  (covers, focus, derotator, elevation, azimuth, dome, filter wheel, torque
  display, manual telescope control, horn, hydraulics, ...). Selection 0 means
  control disconnect and is treated as an error. *Warning: some safety routines
  are disabled when the telescope is operated manually.*

---

## Safety (TwinSAFE)

A TwinSAFE safety application (`TwinSafeGroup1` on the **EL6910** safety PLC,
FSoE over EtherCAT) provides the safety chain:

- **`FBEstop1`** (safeEstop) monitors the emergency-stop chain; its output
  drives **Safe Torque Off (STO) on all three servo axes** (azimuth, elevation,
  derotator) through the **AX5805** TwinSAFE option cards of the AX5000 drives.
- **`FBEdm2`** (safeEdm) monitors the contactor/feedback contacts with
  switch-on/off monitoring times.
- Per-axis STO activation/reset and STO-state/error feedback are implemented
  with safeAnd (`FBAnd1/2/3`) and safeDecouple (`FBDecouple1/2`) blocks and
  exposed as alias devices (`Azimuth/Elevation/Derotator_STOState/_STOReset`,
  `Restart`, `Run`, `ErrorAcknowledgement`, `Safety IN (EL1904)`, `Safety OUT
  (EL2904)`).

## NC axes and I/O

The system project defines six NC axes: `Focus` (Id 1, EL7342 DCM channel 1 +
EL5101 incremental encoder) and `Axis 2`…`Axis 6`. Three of the axes are driven
by the AX5000 servo drives with AX5805 STO option cards (AX5125 ×2, AX5206 ×1);
the exact axis→FB assignment beyond Focus is not recorded in the project
files. The EtherCAT I/O (Device 1) includes the EL6910 safety PLC, EL1904 /
EL2904 TwinSAFE terminals, EL2008 / EL1008 digital I/O and the EL9410 power
supply.

## Dependencies

- **BROTLib** — `FB_BaseAxis`, `FB_Axis2`, `FB_AltAzTelescopeControl`,
  interfaces (`I_Axis`, `I_Focus`, `I_Brake`, `I_Hydraulics`,
  `I_MirrorCovers`), `FB_EventLog`, `E_TelescopeMode`, communication.
- **AstroBROT** — astronomical calculations used by the telescope control.
- Beckhoff system libraries: `Tc2_MC2`, `Tc2_MC2_Drive` (motion control),
  `Tc2_EtherCAT`, `Tc3_IotCommunicator`, `Tc3_Module`, `Tc2_Utilities`, plus
  the TwinCAT visualization libraries (`VisuSymbols`, `VisuElems`, ...).

HalfBROT is consumed by **MONETcommon** (and through it MONETN / MONETS).

## Building and deployment

The library is built with TwinCAT 3.1 Build 4024.66 in TwinCAT XAE (PLC task
10 ms, priority 20; NC-Task 1 SAF 2 ms / SVB 10 ms). Solution platforms cover
TwinCAT RT (x64/x86), TwinCAT CE7 (ARMV7) and TwinCAT OS (ARM/x64). Versioned
with git tags `v0.2.0`, `v0.4.0`; the library release (`Released=false`) has
not been performed yet.
