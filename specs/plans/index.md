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
