# Plans

Dated `YYYY-MM-DD-<slug>.md` investigation and work logs, one per unit of work.

- [2026-09-20-code-review.md](2026-09-20-code-review.md): full code review of HalfBROT
  (bugs, security, design, CI and release). **draft**, review finished; #3 fixed on `develop`. Static read plus cross-repo checks
  against BROTLib, MONETcommon, MONETN and MONETS, not run on a PLC or a telescope.
- [2026-10-11-commands-state-machines-io.md](2026-10-11-commands-state-machines-io.md): plan for #31/#32:
  commands only through methods, I/O through structs (no `AT` in the library), TcUnit
  tests, then one explicit state machine per block. **draft**, nothing implemented; three open questions (manual brake semantics,
  `FB_BaseAxis` inputs, struct layout).
- [2026-10-12-soe-reset-and-restore-in-base.md](2026-10-12-soe-reset-and-restore-in-base.md): #14, SoE reset
  and position restore once in `FB_AxisControl` (`_SoEReset`, `_RestoreLastPosition`); the SoE reset now waits
  for `Busy` to rise and drop, logs `Error` and no longer resets the axis after a failed reset.
  **implemented**, CI build only, hardware check pending.
- [2026-10-12-telemetry-interval-and-bursts.md](2026-10-12-telemetry-interval-and-bursts.md): #19, hydraulics
  telemetry on change and every `fTelemetryInterval`, at most once per second (no lost updates, no bursts);
  `fTelemetryInterval` effective in covers/focus. **implemented**, CI build only.
- [2026-10-12-derotator-range-stop-once.md](2026-10-12-derotator-range-stop-once.md): #20, the derotator
  stops once when leaving its software range instead of every cycle (which cancelled every move/homing back in).
  **implemented**, CI build only, hardware check in #59.
- [2026-10-12-remove-axis-position-restore.md](2026-10-12-remove-axis-position-restore.md): #7, remove the
  `MC_SetPosition` restore from azimuth/elevation (most likely never set `Calibrated`; the axes home on their cam at
  every start anyway). **implemented**, CI build only.
- [2026-10-12-persistent-write-focus-position.md](2026-10-12-persistent-write-focus-position.md): #8, write the
  focus's persistent `fLastPosition` to disk with `FB_WritePersistentData` (`SPDM_2PASS`, port 851) after it changed,
  at rest, at most once a minute, so an unclean power-off does not restore a stale position with `MC_Direct`.
  **implemented**, CI build only, hardware check in #59.
- [2026-10-12-pendant-fixes.md](2026-10-12-pendant-fixes.md): #12, pendant: TC_Restart only in manual mode, pointing
  offsets one 10 arcsec step per press, error lamp cleared each cycle and lit on selection 0 (no safety function),
  horn left unwired. Part A HalfBROT `FB_PendantControl` (CI build only), Part B MONETcommon `FB_MonetPendantControl`
  held on a branch until an on-site test on MONETS and MONETN. Part A **implemented** (CI build only), Part B **proposed**.
- [2026-10-12-drive-diagnostic-check.md](2026-10-12-drive-diagnostic-check.md): #2, drive diagnostic check once in
  `FB_AxisControl._CheckDiagnostic`: AX5000 F errors -> ERROR event and `nErrorID`, E warnings (not ECD0/ECD1) -> WARN
  event only; derotator SoE reset on a reset at most once per drive error (the old line would loop). MONET/S derotator
  diagnostic is linked to the other (apparently unused) drive channel. **implemented** (PR open, not merged; Q1/Q2
  as recommended, decided 2026-10-12), CI build only, release waits for MONETS #19.
- [2026-10-12-cover-control-single-copy.md](2026-10-12-cover-control-single-copy.md): #4/#29, one cover block:
  `FB_CoverControl` keeps its name and gets MONETcommon's `FB_MonetCoverControl` logic 1:1 (sequence by limit switches,
  timeout as input `tMoveTimeout`, 1 s reversal dead time, error stops all) plus `Stop()`; MONETcommon drops its copy,
  MONETS/MONETN `MAIN` use `HalfBROT.FB_CoverControl` (I/O links unchanged); pendants hold-to-run of the whole sequence
  (option B); new `HalfBROTTests`. **proposed**, nine questions for Tim, on-site check T15.
- [2026-10-20-oil-level-guard-monets-32.md](2026-10-20-oil-level-guard-monets-32.md): MONETS#32, a raw oil level outside a configurable range (wire break, over-range) is not published and does not start/stop the suction by level (`F_OilLevelValid`, `F_OilSuctionLatch`, `statusWord` bit 14). **Implemented in HalfBROT, not compiled or hardware-tested;** tests and the MONETS/MONETN `MAIN` settings follow.
- [2026-10-22-focus-manual-jog-target-and-stored-position-66.md](2026-10-22-focus-manual-jog-target-and-stored-position-66.md):
  focus after a pendant jog in manual mode: (A, no issue yet) the target follows the actual position outside automatic
  and during a jog, so the follower does not drive back to the pre-jog target on the switch to automatic; (B, #66)
  `fLastPosition` is tracked in every mode, so `MC_Direct` does not restore the pre-jog position after a restart.
  One plan for both (same block, release, tests and on-site session). Pure-logic POUs tested in MONETcommonTests,
  HalfBROT 0.7.2 plus MONETcommon/MONETS/MONETN pins. **Implemented in HalfBROT, not compiled or hardware-tested;** tests in MONETcommonTests and the pins follow.
- [2026-10-25-focus-enable-handling-108-110-111-114.md](2026-10-25-focus-enable-handling-108-110-111-114.md):
  focus drive enable (#108, #110, #111, #114): one owner of `bEnable` (new pure-logic `FB_FocusDriveEnable`, decided
  before `ObFocusUnlock`), no lock while a move or homing is wanted or the axis is not at rest, follower `bReady`
  from the enable actually applied, stale `MovePending` cancelled, explicit target after a homing; BROTLib
  `FB_MoveFollower` raises `bNeedEnable` for a pending move. #114 root cause partly a hypothesis (trace needed).
  #103, #85, #102, MONETcommon#122, BROTLib#74 out of scope. Sim tests in MONETcommonTests, BROTLib then HalfBROT
  0.7.3 then pins. **proposed**, nothing implemented, compiled or run on a PLC; eight open questions.
- [2026-10-25-focus-target-after-restart-115.md](2026-10-25-focus-target-after-restart-115.md):
  #115, after a PLC restart the calibrated focus has `fPosition` 0 (no initial value), the clamp makes it 0.5 and the
  follower drives there (reproduced on MONET/S, D1). Fix: new pure-logic `FB_FocusTargetInit`, the first calibration
  outside a homing takes the actual position (before the clamp and the follower). Power-on case (H-205, `:205` with
  `fLastPosition` -1) not explained, decision after run D6. Sim tests in MONETcommonTests. **proposed**, nothing
  implemented, compiled or run for the fix; four questions.
