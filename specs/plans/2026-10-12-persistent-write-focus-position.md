# Write the stored focus position to disk while running (#8)

**Status: proposed.** Nothing implemented. Facts below were read from HalfBROT `develop` @ `68faea3` and the
consumers' working trees on 2026-10-12 (MONETS `develop` @ `5bebeb3`, MONETN `develop` @ `5f94f89`, MONETcommon
`develop` @ `38b5896`, BROTLib `develop` @ `6d6035d`), plus the Beckhoff Infosys pages and the installed
`Tc2_Utilities` 3.9.2.0 named below.

Decided 2026-10-12 (Tim): the telescope PCs have a UPS, so a clean TwinCAT shutdown on a power cut is likely; the
focus position is still written with `FB_WritePersistentData` as a second line of defence (runtime crash, UPS
failure, forced reset). `fLastPposition` and `_MigrateLastPosition` stay for one more release (#49), untouched here.

## Problem

After #7 the only persistent variable in HalfBROT is the focus position in `FB_FocusControl`
(`VAR PERSISTENT fLastPosition : LREAL := -1.0`, plus the deprecated `fLastPposition`). The focus has an
incremental encoder with a reference mark. With `fLastPosition > 0.0` it homes with `MC_Direct` (#18): `_StartHoming`
calls `fbAxis.Home(fReference := fLastPosition)`, the body sets `eHomingMode := MC_Direct` and
`fPosition := fLastPosition`. The axis becomes `Calibrated` at that value without moving.

`fLastPosition` is written in the body:

- every cycle while automatic mode, `NOT bManualUnlock`, `fbAxis.calibrated` and `NOT ObFocusUnlock` (brake
  locked, drive off): `fLastPosition := fActualPosition`;
- on `fbAxis.HomeDone`: `fLastPosition := fPosition`;
- while `bError`: `fLastPosition := -1.0` (forces a cam homing next time).

Nothing in HalfBROT, BROTLib, MONETcommon, MONETS or MONETN calls `FB_WritePersistentData`. Beckhoff
([Remanent variables](https://infosys.beckhoff.com/content/1033/tc3_plc_intro/2528803467.html),
[Settings tab](https://infosys.beckhoff.com/content/1033/tc3_userinterface/3434442123.html)): persistent data are
written to `Port_<port>.bootdata` in the boot folder only on a TwinCAT stop/shutdown; at the next start the file is
read and renamed to `.bootdata-old`; if `.bootdata` is missing (power cut without clean shutdown), `.bootdata-old` is
read instead. So after an unclean power-off the focus gets the value from the **last start** (that is, the last clean
shutdown before it), possibly many focus moves old. `MC_Direct` calibrates the incremental encoder to that stale
value, and every focus position is silently off by the distance moved since, while the axis reports `Calibrated`.
MONETRoof's `FB_Roof` documents the same effect for its own persistent data ("TwinCAT does not flush persistent
data every cycle").

## Change (one PR to `develop`)

Only `FB_FocusControl` changes. No change in any consumer.

### FB and parameters

`FB_WritePersistentData` from `Tc2_Utilities`. Signature from the
[Infosys page](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_utilities/35030155.html) (the parameter
comments, the name `MODE` and the enum names also appear in the installed library,
`C:\TwinCAT\3.1\Components\Plc\Managed Libraries\Beckhoff Automation GmbH\Tc2_Utilities\3.9.2.0`, string table):

```
VAR_INPUT
    NETID : T_AmsNetId;                 // '' = local computer
    PORT  : UINT;                       // ADS port of the PLC runtime whose persistent data are written (851, 852, ...)
    START : BOOL;                       // rising edge starts the write
    TMOUT : TIME := DEFAULT_ADS_TIMEOUT;
    MODE  : E_PersistentMode;
END_VAR
VAR_OUTPUT
    BUSY  : BOOL;
    ERR   : BOOL;
    ERRID : UDINT;                      // ADS error code
END_VAR
```

The input is `MODE`, not `WRITEMODE`. `E_PersistentMode`
([Infosys](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_utilities/35342347.html)) has two values only:
`SPDM_2PASS := 0` (all data from the same PLC cycle) and `SPDM_VAR_BOOST := 1` (consistent per variable only).
[System behaviour](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_utilities/35401483.html): `SPDM_2PASS` can
overrun the PLC cycle if copying **all** persistent data takes longer than one cycle; `SPDM_VAR_BOOST` only if the
largest single variable does.

Values used:

- `NETID := ''` (local).
- `PORT := nPersistentAdsPort`, a new `VAR_INPUT nPersistentAdsPort : UINT := 851;`. MONETS and MONETN both run their
  only PLC project on AMS port 851 (`AmsPort="851"` on the `<Project>` element in `MONETS.tsproj` and
  `MONETN.tsproj`), so neither has to set it. Not used: `TwinCAT_SystemInfoVarList._AppInfo.AdsPort` (exists in both
  `.tmc` files) would avoid the input, but whether library code may reference it was **not verified**.
- `MODE := E_PersistentMode.SPDM_2PASS`. The persistent area of these PLCs is tiny (below), so a one-cycle copy is
  fine and the file stays consistent.
- `TMOUT := DEFAULT_ADS_TIMEOUT` (5 s, from the `.tmc`).

What gets written: `FB_WritePersistentData` writes the **whole** persistent area of the PLC runtime, not one
variable. In MONETS/MONETN that is the focus's `fLastPosition` and `fLastPposition` and MONETcommon
`FB_MonetTelescopeControl.nInterruptedAutoClearCount` (a counter, `UDINT`). BROTLib's `FB_RaDecTelescopeControl`
(pointing terms) is not instantiated in either `MAIN`, and MONETRoof is a separate PLC that MONETS/MONETN do not
reference; no `PERSISTENT` in the MONETS/MONETN POUs or GVLs. Writing the counter as well is harmless (it only
becomes more up to date). If a consumer later adds persistent data that must only be stored at shutdown, this has to
be revisited; none exists today.

How long it takes and what it blocks: **not verified**. It is an ADS request to the PLC runtime; the block returns
`BUSY` until the answer. With `SPDM_2PASS` the data are copied in one PLC cycle (risk of a cycle overrun only if the
copy is longer than the 10 ms `PlcTask`, not credible for a few bytes); the file write itself is done by the runtime
outside the PLC code. Measure `BUSY` duration in the hardware test (below).

### Library reference

None needed. HalfBROT already references `Tc2_Utilities, * (Beckhoff Automation GmbH)` (placeholder in
`HalfBROT/HalfBROT/HalfBROT.plcproj`, listed in the README's dependencies). MONETS, MONETN, MONETcommon and BROTLib
reference it with the same placeholder too, and IAG50cm does not use HalfBROT. So no build or placeholder change
anywhere.

### Trigger rule

Write only when the stored value changed and the focus is at rest, rate-limited:

1. **Changed:** `ABS(fLastPosition - fSavedPosition) > fTolerance` (0.005 mm by default; a change below the
   positioning tolerance does not matter), **or** validity changed (`(fLastPosition > 0.0) <> (fSavedPosition > 0.0)`,
   covers the `-1.0` written on an error), **or** `bSaveRequested` (set on every `fbAxis.HomeDone`, so a fresh
   `.bootdata` exists soon after each start and homing; one write per start).
2. **At rest:** `NOT ObFocusUnlock AND NOT bHomeAxis AND NOT fbAxis.Busy`. `ObFocusUnlock` is FALSE when the brake
   is locked and the drive is off, and also while `bError`, so the `-1.0` case is covered. The tracked value only
   changes while locked anyway; this guarantees no write while the drive moves.
3. **Rate limit:** at least `tPersistMinInterval := T#60S` since the last write started (after a failed write
   `tPersistRetryInterval := T#10M`). The timer runs from the first cycle, so there is no write in the first 60 s
   after a start (the persistent data are restored before the first cycle; nothing to save before that).
4. `fSavedPosition` is set to `fLastPosition` in the **first cycle** (`bPersistInit`), so the restored value does
   not trigger a write by itself. (`_MigrateLastPosition` runs in the same first cycle before it; a migrated value is
   then picked up by the `HomeDone` request.)

Expected frequency: the brake locks `ptBrakeDelay` after the last move (30 s on MONETS), so an autofocus series is
one write after it ends, not one per step. Typically a handful per night, worst case 60 per hour. The `.bootdata`
file is small (exact size **not verified**). On an SSD/CFast with wear levelling this is negligible; the storage
type of the telescope PCs was **not verified**.

### Where in `FB_FocusControl`

New member variables in `VAR` (not in the method: a method's `VAR` is re-created on every call, same reason as in
`FB_AxisControl._SoEReset`):

```
// persistent write of fLastPosition, see _WritePersistentPosition (#8)
fbWritePersistent		: FB_WritePersistentData;
nPersistStep			: INT;		// 0 idle, 1 started, 2 running, 10 finished
bPersistInit			: BOOL;		// fSavedPosition taken in the first cycle
fSavedPosition			: LREAL;	// fLastPosition at the last successful write (or as restored)
fWritePosition			: LREAL;	// fLastPosition when the running write started
bSaveRequested			: BOOL;		// write once after HomeDone
tonPersistInterval		: TON;		// rate limit since the last write start
tonPersistWatchdog		: TON;		// ends the sequence if BUSY never drops
bPersistWriteFailed		: BOOL;		// last write failed (cleared by the next successful one)
sPersistWriteMessage	: STRING(255);
persistWriteEvent		: FB_EventLog;
```

plus `VAR_INPUT nPersistentAdsPort : UINT := 851;` and two constants (`VAR CONSTANT` or plain `VAR` with initial
values) `tPersistMinInterval := T#60S`, `tPersistRetryInterval := T#10M`.

Body: in `IF fbAxis.HomeDone THEN`, add `bSaveRequested := TRUE;` after `fLastPosition := fPosition;`. Call
`_WritePersistentPosition();` once per cycle after the `IF bError THEN fLastPosition := -1.0` block and the
`bCalibrated`/`bReady` lines (all writes to `fLastPosition` of the cycle are done), before the telemetry.

New method `METHOD PRIVATE _WritePersistentPosition`, a step sequence modelled on `_SoEReset`:

| Step | Action | Next |
|---|---|---|
| first call | `fSavedPosition := fLastPosition`, `bPersistInit := TRUE` | 0 |
| 0 idle | `tonPersistInterval(IN := TRUE, PT := SEL(bPersistWriteFailed, tPersistMinInterval, tPersistRetryInterval))`; if changed (rule 1) AND at rest (rule 2) AND `tonPersistInterval.Q`: `fWritePosition := fLastPosition`, call `fbWritePersistent(START := FALSE)` (guarantees the edge), restart `tonPersistInterval` (`IN := FALSE`) | 1 |
| 1, 2 | `fbWritePersistent(NETID := '', PORT := nPersistentAdsPort, START := TRUE, TMOUT := DEFAULT_ADS_TIMEOUT, MODE := E_PersistentMode.SPDM_2PASS)`; `ERR` → failed; watchdog → failed; `BUSY` → 2; step 2 and `NOT BUSY` → done | 10 |
| 10 | `fbWritePersistent(START := FALSE)`; on success: `fSavedPosition := fWritePosition`, `bSaveRequested := FALSE`, `bPersistWriteFailed := FALSE` | 0 |

Failed: `bPersistWriteFailed := TRUE`, `sPersistWriteMessage := 'Focus: writing persistent data failed (ADS error <ERRID>)'`
or `'Focus: writing persistent data timed out'`. `fSavedPosition` and `bSaveRequested` stay, so the write is
retried, but only after `tPersistRetryInterval` (rule 3), never in a loop. Watchdog: `tonPersistWatchdog(IN :=
nPersistStep <> 0, PT := T#15S)` (three times `TMOUT`), as in `_SoEReset`.

Event, called every cycle in the method (or in `HandleEvents`):

```
persistWriteEvent(
	Trigger 		:= bPersistWriteFailed,
	Level 			:= ADSLOG_MSGTYPE_ERROR,
	FormatString 	:= '%s',
	OnMessage 		:= sPersistWriteMessage,
	OffMessage 		:= 'Focus: persistent data written again',
	OffLevel 		:= ADSLOG_MSGTYPE_HINT,
	Comm 			:= fbComm);
```

`FB_EventLog` logs on the edges only, so a write that keeps failing logs once, not every 10 minutes.

### Failure mode

A failed or never-finishing write changes nothing in the focus control: positioning, homing and telemetry work as
before; only the protection against an unclean power-off is missing until a write succeeds. The clean shutdown
still writes the data as today. The error goes to the TwinCAT ADS log and, through `fbComm`, to MQTT (`PublishLog`), like the other events.

### Docs

README: one sentence in the focus/calibration part: the focus position is also written to disk with
`FB_WritePersistentData` after it changed (at rest, at most once per minute), so an unclean power-off restores the
last position written. Plans index entry.

## Not changed

- `fLastPposition` and `_MigrateLastPosition` (removal in the next minor after that, #49, decided 2026-10-12).
- Whether `VAR PERSISTENT` inside an FB is stored per instance: open (#8), irrelevant here since MONETS and MONETN
  each have exactly one `FB_FocusControl` (`MAIN.FocusControl`); the by-name restore after a download is item 4 of
  #59.
- **Stale-value detection, considered, not done.** `TwinCAT_SystemInfoVarList._AppInfo.OldBootData` (in both
  `.tmc` files, with `BootDataLoaded`) tells that the backup file was loaded, i.e. no clean shutdown and no write
  since the last start. With the write after every `HomeDone` this only remains true if the PLC dies within about a
  minute of a start, when the focus cannot have moved much; checking it (and forcing a cam homing) would also need
  `_AppInfo` from library code (not verified). Comparing against the cam after `MC_Direct` would need a move, which
  is what `MC_Direct` is meant to avoid. The remaining window (a move, then a crash before the next write: brake
  delay plus up to 60 s) is accepted.
- **Found, separate decision:** `fLastPosition` is only tracked in automatic mode (`IF eTelescopeMode =
  E_TelescopeMode.automatic AND NOT bManualUnlock`). A focus jogged in manual mode and then shut down (even cleanly)
  keeps the value from before the jog. Moving the tracking out of the automatic branch is a one-line change but
  changes behaviour; not part of this plan, proposed as its own issue.
- The UPS-triggered shutdown itself (no code or config handles it, #8 comment): out of scope.

## Risks

- A community source ([industrialmonitordirect.com](https://industrialmonitordirect.com/blogs/knowledgebase/beckhoff-twincat-3-writepersistentdata-not-working-fix))
  claims `FB_WritePersistentData` "does not respond" for persistent variables declared locally in a POU and that an
  empty `NETID` can give `ERR`. **Not verified**, contradicts Infosys (`''` = local). The hardware test checks
  exactly this (file timestamp and the restore after a hard reset). If `NETID := ''` fails, use the PLC's own
  AmsNetId (would need another input).
- Whether `FB_WritePersistentData` writes `Port_851.bootdata` (and not some other file) is inferred from the
  Settings-tab text, **not verified**; the hardware test checks the timestamp.
- First download after the update: adding non-persistent variables does not change the persistent data; no effect
  on the restore expected (#59 item 4 covers the download anyway).

## Release

Behaviour addition, no interface break (one new `VAR_INPUT` with a default). Goes into the next HalfBROT minor
together with the unreleased #53..#65 on `develop`.

## Checks

- CI: TcBuild of HalfBROT on the PR branch; the run log must end with `Build complete -- 0 errors, 0 warnings`
  (`gh run view <id> --log -R BROTLib/HalfBROT`).
- XAE (Tim): MONETS and MONETN build against the new HalfBROT release (no consumer change expected).
- Hardware, new item 5 in #59 (MONETS or MONETN, focus calibrated, automatic mode, telescope parked):
  1. Note the timestamp of `Port_851.bootdata` in the `Plc` subfolder of the TwinCAT boot folder (`TwinCAT\Boot`
     per Infosys; the exact path depends on the TwinCAT version on the telescope PC, **not verified**; the repos'
     `_Boot` copies hold the other `Port_851.*` files). After a start it should be updated about 60 s after the focus
     got `Calibrated`.
  2. Move the focus by about 0.5 mm. After `ptBrakeDelay` (30 s on MONETS) plus at most 60 s the file timestamp
     changes, `FocusControl.fSavedPosition = fLastPosition`, `FocusControl.fbWritePersistent.ERR = FALSE`, no error
     event. Note how long `BUSY` was TRUE (trace or ADS watch) and whether the PLC reported a cycle overrun.
  3. Move again, wait for the write, then a hard reset without TwinCAT shutdown (power cut behind the UPS or the
     IPC reset). After the restart: `_AppInfo.BootDataLoaded = TRUE`, `_AppInfo.OldBootData = FALSE`, and the focus
     becomes `Calibrated` **without moving** at the position of step 3 (compare `fActualPosition`).
  4. Optional, error path: set `FocusControl.nPersistentAdsPort := 999` over ADS and move the focus: one error event
     `Focus: writing persistent data failed (ADS error ...)`, focus keeps working; set 851 again: hint event
     `Focus: persistent data written again` within 10 minutes.
  Fail if: no timestamp change, an error event in step 2, or the focus restores an older position in step 3.
