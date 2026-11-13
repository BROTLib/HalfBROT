# HalfBROTTests

TcUnit tests for the function blocks of HalfBROT (HalfBROT#32, step 3 of
[the plan](../specs/plans/2026-10-11-commands-state-machines-io.md)). A separate PLC project, so no test code ends up in the
shipped library. It references the *installed* HalfBROT (`HalfBROT, * (BROT)`), exactly like MONETN or MONETS do. Structure
and tooling are copied from `MONETcommonTests`; see its README and `BROTLibTests/README.md` for the one-time setup and the
Windows 11 user-mode runtime notes, they apply unchanged here.

Since 0.14.0 the blocks take their I/O as structs (`ST_*In` as `VAR_INPUT`, `ST_*Out` as `VAR_IN_OUT`) and their commands only
through methods, so a test is a rig around one block: it sets the `In` struct, calls the methods, runs the block and reads the
`Out` struct. The rigs (`FB_HydraulicsRig`, `FB_CoverRig`) hold the block, the structs and a small plant behind them.

## What is covered

| Suite | What it checks |
|---|---|
| `FB_HydraulicsControl_Tests` | `FB_HydraulicsControl` (characterisation, they pin what the block does today): no request means everything off; `Enable` starts the pump; the brake opens about 3 s after the pressure and is reported 1 s later; no brake without `bBrakeClearing`; pressure loss closes the brake after the 2 s debounce and the automatic request opens it again; `CloseBrake()`, a pump stop and the azimuth limit close it; the manual brake of #31 (the switch opens it once, it stays closed after a pressure loss, `CloseBrake()` or a pump restart while the switch stays on, switch off and on opens it again, switch off closes it); oil low, the hydraulics watchdog (#10) and a missing pump feedback are latched errors cleared by `Reset()` |
| `FB_CoverControl_Tests` | `FB_CoverControl` (characterisation): open 1->3->2 and close 2->3->1, each cover only after the previous one reached its switch; the last call of `Open()`/`Close()` wins; `Stop()` drops the outputs and the command; a limit switch fault stops all covers and is latched until `Reset()` (#28); the move timeout; the reversal dead time (#27) |
| `FB_BrakeCloseHold_Tests` | `FB_BrakeCloseHold` (MONETcommon#277): the enable of a moving, enabled axis is kept when the brake gate closes under it; never a new enable against a closed brake, never for a disabled axis or at standstill; the hold ends at standstill, when the gate opens, on a disable, and after `tMaxHold`; `tMaxHold = T#0S` is the old behaviour |
| `FB_FocusUnlock_Tests` | `FB_FocusUnlock`, the unlock sequence of `FB_FocusControl` (rig with a drive model and the real `FB_MoveFollower`): the drive is enabled with the lock closed, the lock opens only after the power status and the settle time; no move before `ptFocusDelay` after the unlock; no power status within the timeout keeps the lock closed and drops the drive until `Reset`; a lost power status closes the lock; an error, and a dropped request, close the lock and drop the drive in one cycle |

Not covered yet: the 15 s suction and 30 s pressure watchdogs of the hydraulics (they need a time seam, the real time is too long
for CI), the pendant, the axes and the focus block (only its unlock sequence, `FB_FocusUnlock`). The pure-logic helpers of HalfBROT (`FB_FocusJogLatch`, `FB_FocusManualHome`,
`F_*`, ...) are tested in `MONETcommonTests` (suite `FB_HalfBROTHelpers_Tests` and others).

## Running the tests

```powershell
tctool HalfBROT.sln library --xae HalfBROT --plc HalfBROT --out HalfBROT.library --install
cd C:\TwinCAT\3.1\Runtimes\UmRT_Default; .\Start.bat
cd HalfBROTTests; tctool HalfBROTTests.sln build run --require-valid-license
```

CI does the same through `.github/workflows/tcbuild.yml`: it installs the checked-out HalfBROT into the machine's library
repository (that is the compile), then builds and runs the tests on the user-mode runtime. Do not run a manual XAE activation while
CI runs: both use the same runtime and boot folder.

## Things to know

- TcUnit parameters: 41 suites, 32 tests and 256 asserts per suite. A multi-cycle test must not assert in every cycle: the
  asserts of a suite are limited, so the tests collect a "violated" flag over a time and assert it once.
- Every test method runs in every PLC cycle, all tests of a suite side by side; a test that waits owns its own state (`VAR_INST`).
- The block timers run on the system time, so the tests that wait take real time (the longest, about 15 s, are the manual brake
  tests); the suites run in parallel, so the whole run takes about as long as the longest test.
- TcUnit writes the text of a failed assertion only to the ADS log, which CI cannot see. `FB_TestBase.Check()` therefore also
  appends `test: message;` to `GVL_Diag.sFailures`; `tools/Run-Tests.ps1` prints it as `Failed checks: ...`.
- `.tmc`, `_Boot`, `_CompileInfo` and `_Libraries` are generated and git-ignored.
