# Covers: one FB_CoverControl in HalfBROT, MONETcommon's logic, used by MONETS and MONETN (#4, #29)

**Status: proposed.** Decided 2026-10-12 (Tim): "drop the HalfBROT version and move the MONETcommon version into
HalfBROT". Nothing implemented, nothing run on a PLC or a telescope. Facts below were read on 2026-10-12 from
HalfBROT `origin/develop` @ `2cfaaf5`, MONETcommon `origin/develop` @ `38b5896` (= v0.8.0), MONETS `origin/develop`
@ `67e683d` (v1.4.0), MONETN `origin/develop` @ `5f94f89`, BROTLib `origin/develop` @ `6d6035d`, IAG @ `05dda82`.
Line numbers refer to those commits. Items marked **unverified** could not be checked from the code.

**Refreshed 2026-10-27 against the local checkouts** (HalfBROT `develop` @ `07d0c49` = 0.8.0, MONETcommon `develop`
= 0.9.10, MONETS/MONETN `MAIN` as local). Still true: the issue #4 findings in `FB_CoverControl.TcPOU`; MONETcommon's
`FB_MonetCoverControl.TcPOU` and `FB_MonetPendantControl.TcPOU` unchanged since v0.8.0 (the port source is the same
as described); `MAIN` l. 22 in MONETS and MONETN; both pendants as described; no test project or `tests.yml` in
HalfBROT. Changed since the first version of this plan:

- Versions: HalfBROT is at 0.8.0 (not 0.5.x/0.6.0), MONETcommon at 0.9.10. Release numbers below are HalfBROT
  **v0.9.0** and MONETcommon **v0.10.0** (next free minors; check again when merging). Whether MONETcommon #68 is
  still open or released was **not rechecked**.
- Paths: the POUs are one folder deeper than written first (`HalfBROT/HalfBROT/HalfBROT/POUs/`,
  `MONETcommon/MONETcommon/MONETcommon/`); fixed below. The test project paths (`MONETcommonTests/...`) were not
  rechecked.
- HalfBROT's body already has the #19 telemetry fix `tonComm(IN:=TRUE, PT:=fTelemetryInterval);`; MONETcommon's
  copy still has `tonComm.PT := ...; tonComm(IN:=TRUE);`. The port keeps HalfBROT's form (see difference 5).
- Line numbers into `MAIN` and the pendants moved (MONETS `MAIN`: `CoverControl.Reset()` l. 92, `CoverControl();`
  l. 126, `Error` l. 213); re-read them when implementing.

Repos: HalfBROT, MONETcommon, MONETS, MONETN (and the IAG fleet list). Issues: HalfBROT #4, #5 (cover part), #29,
#32; MONETcommon #3 (cover part), #10, #27, #28, #36, #62; MONETN #15 check 5.

## Problem

Two copies of the same cover block exist, and the one the telescopes run is in the wrong library:

| | HalfBROT `FB_CoverControl` | MONETcommon `FB_MonetCoverControl` |
|---|---|---|
| File | `HalfBROT/HalfBROT/HalfBROT/POUs/FB_CoverControl.TcPOU` | `MONETcommon/MONETcommon/MONETcommon/FB_MonetCoverControl.TcPOU` |
| POU Id | `{febe04fb-b0fe-4b52-a725-1c7655537933}` | **the same Id**, and the same method/property Ids (copies of one file) |
| Used by | no IAG telescope; Potsdam (HalfBROT #29 comments) | MONETS and MONETN `MAIN` l. 22: `CoverControl : FB_MonetCoverControl(comm := fbComm);` |
| Interface | `IMPLEMENTS I_MirrorCovers` | `IMPLEMENTS I_MirrorCovers` |

`I_MirrorCovers` is defined in BROTLib (`BROTLib/BROTLib/Interfaces/I_MirrorCovers.TcIO`): methods `Open()`,
`Close()`, `Reset()` (all `: BOOL`), properties `Opened`, `Closed`, `Error` (all `BOOL`, get only). No `Stop`.

**Old HalfBROT logic (verified, matches Tim's derivation and #4):**

- Open: `OabOpenCover[1] := IabCoverOpen[1]` and `OabOpenCover[3] := IabCoverOpen[3]` start covers 1 and 3
  together; cover 2 starts 1 s (`tonCover2OpenDelay`, `IN := bOpen AND NOT abCoverClosed[3]`) after cover 3
  left its closed switch.
- Close: cover 2 at once; cover 3 3 s (`tonCover3CloseDelay`, `IN := bClose AND NOT abCoverOpen[2]`) after cover
  2 left its open switch; cover 1 once cover 3 is closed.
- Outputs are written only in the branch of the active command (`IF bClose ... ELSIF bOpen ...`): after a
  reversal the outputs of the old direction stay TRUE (stale, both directions driven).
- `bError` (SR latch on "both switches active") stops nothing; no timeout (`afbTimeoutEvent` triggers are
  constant `FALSE`); outputs follow the raw NC inputs.

**MONETcommon logic (verified, v0.8.0):** all six outputs recomputed every cycle; open 1->3->2 and close 2->3->1,
each cover waiting for the previous one to be at its end switch; per-cover travel timeout (`atonMoveTimeout`,
`PT := T#60S` hard-coded, latched in `asrCoverTimeout` until `bReset`, since v0.4.1); `bError` drops `bOpen` and
`bClose` and gates every output (#28, v0.8.0); 1 s reversal dead time (`atofOpen`/`atofClose`,
`tReversalDeadTime`, #27, v0.8.0); close has precedence over open. **The #27/#28 parts and the 60 s value have not
run on a telescope yet** (MONETcommon #62 open; IAG test plan
`IAG/specs/plans/2026-10-12-monet-telescope-test-plan.md` T8 steps 5-6 and T11 row 5 are still unticked; what
runs on the two controllers today is unknown, test plan "Scope").

**Declaration differences (verified):**

| Variable | HalfBROT | MONETcommon |
|---|---|---|
| `bReset`, `bOpen`, `bClose` | `VAR` (set by the methods) | `VAR_INPUT` (comment: "meant to be driven only via the methods") |
| `OabOpenCover`, `OabCloseCover` `AT%Q*` | `VAR_INPUT` | `VAR` |
| `IabCoverOpen`, `IabCoverClosed` `AT%I*` | `VAR` | `VAR` |
| `tReversalDeadTime` | - | `VAR_INPUT := T#1S` |
| timeout | - | `PT := T#60S` literal in the body |
| `tonCover2OpenDelay`, `tonCover3CloseDelay` | yes | - |
| `_SendTelemetry` | `IF fbComm = 0 THEN RETURN` guard (#16) | no guard |

**Who touches the cover block from outside (grep over HalfBROT, MONETcommon, MONETS, MONETN, BROTLib,
IAG50cm `origin/develop`):**

- MONETS/MONETN `MAIN`: declaration l. 22, `CoverControl.Reset()` (MONETS l. 89, MONETN l. 87),
  `fbCovers := CoverControl` into `TelescopeControl` (l. 103/101), `ElevationControl` (l. 129/127) and
  `PendantControl` (l. 159/157), `CoverControl();` without parameters (l. 119/117), `CoverControl.Error`
  (l. 201/185). `TelescopeControl` and `ElevationControl` take `I_MirrorCovers`; only the pendants take the
  concrete type.
- MONETcommon `FB_MonetPendantControl` l. 7 `fbCovers : REFERENCE TO FB_MonetCoverControl;`, l. 97-110:
  writes `fbCovers.bOpen := IbDirectionUp AND fbCovers.Opened; fbCovers.bClose := IbDirectionDown AND
  fbCovers.Closed;` and reads `fbCovers.bError`. **The only external writer of `bOpen`/`bClose`.**
- HalfBROT `FB_PendantControl` l. 6 `fbCovers : REFERENCE TO FB_CoverControl;`, l. 131-144: key on writes
  `fbCovers.OabOpenCover[Selection]`/`OabCloseCover[Selection]` directly (single-cover jog, no interlock), key
  off calls `Open()`/`Close()` while a direction is held; reads `IabCoverOpen/Closed[Selection]`,
  `abCoverError[Selection]`.
- MONETcommonTests `FB_MonetCoverControl_Tests` (one test, unlinked hardware -> `bError`), instance in the test
  `MAIN`, row in `MONETcommonTests/README.md`.
- MONETcommon `FB_MonetTelescopeControl`, `FB_MonetDiagnostics`, BROTLib `FB_Comm_MQTT*` (`MirrorCovers :
  I_MirrorCovers`, not assigned in MONETS/MONETN `MAIN`): interface only, not affected.
- IAG50cm has its own, unrelated `FB_CoverControl` (state machine, `POUs/CoverControl/`) and does **not**
  reference HalfBROT (its `.plcproj` references BROTLib only): no clash.
- pyobs-monet: not on this machine, **unverified**. MONET commands covers over MQTT through `TelescopeControl`;
  an ADS client would see unchanged symbol paths (see below).

## Decision

One implementation, in HalfBROT, used by MONETS, MONETN and anyone else. Tim's explicit decision overrides the
standing rule "never delete HalfBROT blocks" for this block only, and the 2026-10-11 decision in HalfBROT #29
("covers kept in both libraries"). The HalfBROT block **stays** (same name, same file, same POU Id); only its
declarations and body are replaced.

Recommended design choices (each one is also a question for Tim at the end):

1. **Name: keep `FB_CoverControl`**, file and POU Id unchanged. A rename buys nothing (no clash: MONETcommon's
   copy goes away, IAG50cm does not reference HalfBROT) and would break every external user (Potsdam) twice.
2. **Interface:** `I_MirrorCovers` unchanged (no BROTLib change). Commands only through methods (`Open()`,
   `Close()`, `Reset()`, plus a new `Stop()` that is not part of the interface), as in step 1 of the #31 plan
   (`2026-10-11-commands-state-machines-io.md`): `bOpen`/`bClose`/`bReset` and `OabOpenCover`/`OabCloseCover`
   in `VAR`. `tReversalDeadTime` (1 s) and a new `tMoveTimeout` (60 s, replaces the literal) as `VAR_INPUT`.
   `AT%I*`/`AT%Q*` stay inside the block (step 2 of the #31 plan moves all I/O later).
3. **Pendant: option B** (no single-cover jog; selection 1-3 with the key on runs the whole sequence
   hold-to-run, `Stop()` on release, key off or selector change). Option A (per-cover manual requests in the
   block) is written out below as the alternative.
4. **Tests: a new `HalfBROTTests` project** (copy of `MONETcommonTests`, with its own `tests.yml`); the cover
   suite moves there and is extended with sequencing, timeout, dead time and error tests via a probe subclass.
5. **No deprecated `FB_MonetCoverControl` subclass** in MONETcommon: MONETS and MONETN are its only users and
   change in the same release train; the pendant has to change anyway (it writes `bOpen`/`bClose`).

## Change

### HalfBROT step 0: `HalfBROTTests` project and CI (own PR, no library change)

Copy `MONETcommonTests/` from MONETcommon `origin/develop` to `HalfBROTTests/` in HalfBROT and rename
(`MONETcommonTests` -> `HalfBROTTests` in folder, `.sln`, `.tsproj`, `.plcproj`, README, tools). Files:
`HalfBROTTests.sln`, `HalfBROTTests/HalfBROTTests.tsproj`, `HalfBROTTests/HalfBROTTests/HalfBROTTests.plcproj`,
`PlcTask.TcTTO`, `POUs/MAIN.TcPOU`, `POUs/FB_NullComm.TcPOU`, `README.md`, `tools/Install-TcUnit.ps1`,
`tools/Run-Tests.ps1`, `vendor/tcunit.library`, `vendor/README.md`.

- **Fresh GUIDs** for every POU/method/project (Python `uuid.uuid4()`), never copied ones. Fresh `ProjectGuid`
  in the `.plcproj` and `.tsproj`.
- Library references in the `.plcproj`: `HalfBROT, * (BROT)` (namespace `HalfBROT`), `BROTLib, * (BROT)`,
  `Tc2_Standard`, `Tc2_System`, `Tc2_Utilities`, `Tc2_MC2`, `Tc3_Module`, `TcUnit, * (www.tcunit.org)`, plus
  whatever HalfBROT itself needs to resolve (take the list from `MONETcommonTests.plcproj` l. 54-96, drop
  `MONETcommon`; keep `AstroBROT` only if the HalfBROT library needs it at link time).
- Drop the MONETcommon suites (`F_AzimuthOffsetOnAxis_Tests`, `F_HomeBelowHorizon_Tests`,
  `FB_MethodVarProbe*`); `MAIN` starts with one trivial suite (or the moved unlinked cover test, see step 1)
  so the CI wiring can be debugged without the safety-relevant change.
- `.github/workflows/tests.yml`: copy of MONETcommon's, with the install step
  `TcBuild.exe install HalfBROT.sln -x HalfBROT -p HalfBROT -l $file` (same as HalfBROT's `release-library.yml`
  l. 44), `working-directory: HalfBROTTests`, `TcBuild.exe build HalfBROTTests.sln`,
  `.\HalfBROTTests\tools\Run-Tests.ps1`. Same `concurrency: group: twincat-tests` (one runtime on the shared
  runner), `push` only, never `pull_request`.
- `.gitignore`: the generated `.tmc`/`_Boot`/`_CompileInfo`/`_Libraries` of the test project, as in MONETcommon.

Breaks nothing alone. Note: like MONETcommon's `tests.yml`, every run installs the checked-out HalfBROT into the
runner's library repository, so the runner's `HalfBROT` is the last-pushed branch afterwards (MONETcommon CI
builds resolve `*` to it).

### HalfBROT step 1: new `FB_CoverControl` (own PR, closes #4)

All edits in `HalfBROT/HalfBROT/HalfBROT/POUs/FB_CoverControl.TcPOU` (UTF-8 with BOM, keep it). Keep the POU Id and the
Ids of `_SendTelemetry`, `Close`, `Closed`, `Error`, `FB_Init`, `Open`, `Opened`, `Reset` (they are already
the same as MONETcommon's). New method `Stop` with a fresh GUID. The `<LineIds>` blocks are not edited by hand
(XAE rewrites them on the next save, as in #57).

**Declaration (target; `[NEW]`/`[CHANGED]` mark every difference from MONETcommon v0.8.0, the markers are not
part of the code):**

```
// Three mirror covers, sequenced by their limit switches: open 1->3->2, close 2->3->1, each cover starts only when
// the previous one is at its end switch. Commands only through Open(), Close(), Stop() and Reset().
// Ported 1:1 from MONETcommon FB_MonetCoverControl v0.8.0 (cover plan 2026-10-12); the only copy since then.   [NEW]
FUNCTION_BLOCK FB_CoverControl IMPLEMENTS I_MirrorCovers                                          [CHANGED: name]
VAR_INPUT
	// telemetry interval
	fTelemetryInterval		: TIME := T#5S;	
	// #27: a cover may only be driven in one direction once its opposite output has been off this long. Only delays a
	// reversal (a stop is immediate); 1 s is a safe default for any drive, shorten it once the drives are known.
	tReversalDeadTime		: TIME := T#1S;
	// a cover whose output has been on this long without reaching its target switch stops and latches an error   [NEW]
	// until Reset(). 60 s is an untested placeholder: set it per telescope in MAIN once the travel time is known.
	tMoveTimeout			: TIME := T#60S;
END_VAR
VAR_OUTPUT
	bAllOpen			: BOOL;		// all covers are open
	bAllClosed			: BOOL;		// all covers are closed
	bError				: BOOL;		// any cover-driver reports an error
END_VAR
VAR
	fbComm				: I_Comm;
	tonComm 			: TON := (PT:=fTelemetryInterval);	

	// commands, set only by Reset(), Open(), Close() and Stop(); the block clears them itself          [CHANGED: were VAR_INPUT]
	bReset				: BOOL;		// resets all drives after an error has occurred
	bOpen				: BOOL;		// open the cover in proper order
	bClose				: BOOL;		// close the cover in proper order

	// AT%I*/AT%Q* below are mapped directly on this FB: only one instance per PLC is
	// supported, a second instance would collide on the same physical hardware.
	// output signals for opening/closing covers
	OabOpenCover  AT%Q* : ARRAY[1..3] OF BOOL;	// open cover1-3 command
	OabCloseCover AT%Q* : ARRAY[1..3] OF BOOL;   // close cover1-3 command
	
	// limit switches
	IabCoverOpen	AT%I* : ARRAY[1..3] OF BOOL;	// limit coper 1 open switch
	IabCoverClosed	AT%I* : ARRAY[1..3] OF BOOL;	// limit coper 1 closed switch
	
	// status
	abCoverOpen			: ARRAY[1..3] OF BOOL;	// cover is open
	abCoverClosed		: ARRAY[1..3] OF BOOL;	// cover is closed
	abCoverError			: ARRAY[1..3] OF BOOL;	// cover has error
	afbCoverEvent			: ARRAY[1..3] OF FB_EventLog;	// cover event
	afbTimeoutEvent : ARRAY[1..3] OF FB_EventLog;	// cover has timeout

	// M3: drive timeout if a cover does not reach its target limit switch within tMoveTimeout          [CHANGED: comment]
	atonMoveTimeout		: ARRAY[1..3] OF TON;
	asrCoverTimeout		: ARRAY[1..3] OF SR;	// latched until bReset, see the TON note above
	abCoverTimeout		: ARRAY[1..3] OF BOOL;
	atofOpen			: ARRAY[1..3] OF TOF;	// #27: Q stays TRUE for tReversalDeadTime after the open output dropped
	atofClose			: ARRAY[1..3] OF TOF;	// #27: same for the close output
	
	srErrorTrigger			: SR;	// trigger for error
	
	fbCoversOpenedEvent		: FB_EventLog; // log if covers are open
	fbCoversClosedEvent		: FB_EventLog; // log if covers are closed
	fbCoversErrorEvent		: FB_EventLog; // Event log for a generic drive error
	
	i	: UINT;
END_VAR
```

The variable names `OabOpenCover`, `OabCloseCover`, `IabCoverOpen`, `IabCoverClosed` and their comments are kept
exactly (the `.tsproj` `<Var>` entries carry the comments; keeping them avoids a cosmetic `.tsproj` diff on save).
`tonCover2OpenDelay`/`tonCover3CloseDelay` are gone.

**Body (target): MONETcommon v0.8.0's body verbatim, with exactly two changes marked, plus the telemetry timer lines (HalfBROT keeps
its #19 form, see difference 5):**

```
(*
This is the control for the two (three) telescope covers. 
The correct order for opening is 1->3->2 and for closing 2->3->1.
GVL Open and close signals are inverted
*)

// close has precedence over open
IF bClose THEN
	bOpen := FALSE;
END_IF

// do stuff for all covers
FOR i := 1 TO 3 BY 1 DO
	// limit switches are inverted
    abCoverOpen[i]   := NOT IabCoverOpen[i];
	abCoverClosed[i] := NOT IabCoverClosed[i];

	// errors occur if covers are both opened and closed
	abCoverError[i] := abCoverOpen[i] AND abCoverClosed[i];

	// M3: a cover being driven but not reaching its target switch is also an error,
	// and must stop driving the output (gated below), not just log and keep pushing.
	atonMoveTimeout[i](IN := OabOpenCover[i] OR OabCloseCover[i], PT := tMoveTimeout);      // [CHANGED: was PT := T#60S]
	asrCoverTimeout[i](SET1 := atonMoveTimeout[i].Q, RESET := bReset, Q1 => abCoverTimeout[i]);
	abCoverError[i] := abCoverError[i] OR abCoverTimeout[i];

	// #27: evaluated with last scan's outputs, before they are recomputed below, so a reversal in the same scan is caught
	atofOpen[i](IN := OabOpenCover[i], PT := tReversalDeadTime);
	atofClose[i](IN := OabCloseCover[i], PT := tReversalDeadTime);
END_FOR;

// error indicates an axis error on any axis; latched until bReset
srErrorTrigger(
	SET1 := abCoverError[1] OR abCoverError[2] OR abCoverError[3],
	RESET := bReset,
	Q1 => bError);

// #28: an error stops all covers, not just the faulty one. A cover with a limit-switch error reads both open and
// closed, so without this the next cover in the sequence would start while the faulty one may still be in its way.
// The command is dropped too, so nothing moves by itself after Reset(): it needs a new Open()/Close().
IF bError THEN
	bOpen := FALSE;
	bClose := FALSE;
END_IF

// close 2->3->1
OabCloseCover[2] := bClose AND NOT bError AND NOT atofOpen[2].Q AND NOT abCoverClosed[2] AND NOT abCoverTimeout[2];
OabCloseCover[3] := bClose AND NOT bError AND NOT atofOpen[3].Q AND NOT abCoverClosed[3] AND abCoverClosed[2] AND NOT abCoverTimeout[3];
OabCloseCover[1] := bClose AND NOT bError AND NOT atofOpen[1].Q AND NOT abCoverClosed[1] AND abCoverClosed[2] AND abCoverClosed[3] AND NOT abCoverTimeout[1];

// open 1->3->2
OabOpenCover[1] := bOpen AND NOT bError AND NOT atofClose[1].Q AND NOT abCoverOpen[1] AND NOT abCoverTimeout[1];
OabOpenCover[3] := bOpen AND NOT bError AND NOT atofClose[3].Q AND NOT abCoverOpen[3] AND abCoverOpen[1] AND NOT abCoverTimeout[3];
OabOpenCover[2] := bOpen AND NOT bError AND NOT atofClose[2].Q AND NOT abCoverOpen[2] AND abCoverOpen[1] AND abCoverOpen[3] AND NOT abCoverTimeout[2];

// we are open, if all covers are open
bAllOpen := abCoverOpen[1] AND abCoverOpen[2] AND abCoverOpen[3];
// we are closed, if all covers are closed
bAllClosed := abCoverClosed[1] AND abCoverClosed[2] AND abCoverClosed[3];

// clear reset, if no error present
IF NOT bError THEN
	bReset := FALSE;
END_IF

IF bAllClosed THEN
	bClose := FALSE;
END_IF

IF bAllOpen THEN
	bOpen := FALSE;
END_IF

fbCoversErrorEvent.OnMessage := 'Cover drive error';
IF bError THEN
	IF abCoverError[3] THEN 
		fbCoversErrorEvent.OnMessage := 'Cover 3 has a drive error.';
	END_IF
	IF abCoverError[2] THEN 
		fbCoversErrorEvent.OnMessage := 'Cover 2 has a drive error.';
	END_IF
	IF abCoverError[1] THEN 
		fbCoversErrorEvent.OnMessage := 'Cover 1 has a drive error.';
	END_IF
END_IF

// send telemetry
// PT set in the call: the declaration's initial value is evaluated only once, at init (#19)      [CHANGED: HalfBROT's #19 form]
tonComm(IN:=TRUE, PT:=fTelemetryInterval);
IF tonComm.Q THEN // publish new payload every fTelemetryInterval                                  [CHANGED: comment, said "every second"]
	tonComm(IN:=FALSE);
	THIS^._SendTelemetry();	
END_IF

... (fbCoversErrorEvent, afbCoverEvent[1..3], afbTimeoutEvent[1..3] with Trigger := abCoverTimeout[i],
     fbCoversOpenedEvent, fbCoversClosedEvent: verbatim from MONETcommon v0.8.0, l. "fbCoversErrorEvent(" to the end)
```

The elided tail is copied unchanged (it is event logging only, about 50 lines); the implementer copies the whole
MONETcommon body and then makes the marked edits (and keeps the #19 telemetry timer lines), rather than retyping it.

**Methods and properties:**

- `Open`, `Close`, `Reset`, `Opened`, `Closed`, `Error`, `FB_Init`: unchanged (identical in both files today).
- `_SendTelemetry`: keep HalfBROT's version with the `IF fbComm = 0 THEN RETURN; END_IF` guard (#16). On MONET
  `comm` is always passed (`MAIN` l. 22), so the guard never triggers there.
- `Stop` **[NEW]** (fresh GUID; not part of `I_MirrorCovers`):

```
// Drops a running open or close; the outputs go off in the next call of the block (no dead time on a stop).
// Not part of I_MirrorCovers: used by the pendant (hold-to-run), an application may use it too.
METHOD Stop : BOOL
VAR_INPUT
END_VAR

bOpen := FALSE;
bClose := FALSE;
Stop := TRUE;
```

**Complete list of intentional differences from MONETcommon v0.8.0 (nothing else may differ):**

1. Name `FB_CoverControl` (POU/method Ids are the same in both files already).
2. Header comment above `FUNCTION_BLOCK`.
3. `bReset`, `bOpen`, `bClose`: `VAR_INPUT` -> `VAR` (comment changed). Only external writer today:
   MONETcommon's pendant (changed below). The M3 comment on them is replaced.
4. New `VAR_INPUT tMoveTimeout : TIME := T#60S;` and `PT := tMoveTimeout` instead of `PT := T#60S` (same value
   by default; comment of `atonMoveTimeout` adjusted).
5. Telemetry timer: keep HalfBROT's `tonComm(IN:=TRUE, PT:=fTelemetryInterval);` (#19) instead of MONETcommon's
   `tonComm.PT := ...; tonComm(IN:=TRUE);` (same behaviour), and the comment "every second" -> "every fTelemetryInterval".
6. `_SendTelemetry` guard (#16).
7. New method `Stop()`.

With the defaults (`tMoveTimeout = 60 s`, `tReversalDeadTime = 1 s`) and no caller of `Stop()`, items 1-7 do not
change any output for a caller that only uses `Open()`/`Close()`/`Reset()` and the properties, i.e.
`TelescopeControl`, `ElevationControl` and `MAIN` on MONETS/MONETN. The pendant is the only caller that changes.

**HalfBROT `FB_PendantControl`** (same PR; it would no longer compile, because it writes `OabOpenCover`/
`OabCloseCover`, which are now `VAR`). Option B (recommended):

Declarations, after `right_trigger` (l. 35):

```
	// covers, key on: hold-to-run of the whole sequence (cover plan 2026-10-12)
	bCoverJogUp			: BOOL;
	bCoverJogDown		: BOOL;
	bCoverJogActive		: BOOL;		// the pendant started the running open/close and stops it on release
	coverUpTrigger		: R_TRIG;
	coverDownTrigger	: R_TRIG;
```

Body, after `fbStartTrigger(CLK := start_switch);` (l. 121), before the `CASE`, so it also runs when the
selector has left 1-3:

```
// covers, key on, selection 1-3: hold-to-run of the whole sequence (open 1->3->2, close 2->3->1); the cover block does
// the order, interlock, timeout and dead time. A run the pendant started stops when the button is released, the key
// goes off or the selector leaves 1-3 (#5, cover part). After an error a new press is needed (edge, not level).
bCoverJogUp := key_switch AND Selection >= 1 AND Selection <= 3 AND direction_up AND NOT direction_down;
bCoverJogDown := key_switch AND Selection >= 1 AND Selection <= 3 AND direction_down AND NOT direction_up;
coverUpTrigger(CLK := bCoverJogUp);
coverDownTrigger(CLK := bCoverJogDown);
IF __ISVALIDREF(fbCovers) THEN
	IF coverDownTrigger.Q THEN
		fbCovers.Close();
		bCoverJogActive := TRUE;
	ELSIF coverUpTrigger.Q THEN
		fbCovers.Open();
		bCoverJogActive := TRUE;
	ELSIF bCoverJogActive AND NOT bCoverJogUp AND NOT bCoverJogDown THEN
		fbCovers.Stop();
		bCoverJogActive := FALSE;
	END_IF
END_IF
```

`CASE` branch `1, 2, 3` (l. 131-144) becomes:

```
		1, 2, 3:	// covers: the whole sequence (hold-to-run above); single covers are not moved separately
			lamp_up := fbCovers.IabCoverOpen[Selection];		// lit while the selected cover is not open
			lamp_down := fbCovers.IabCoverClosed[Selection];	// lit while the selected cover is not closed
			IF key_switch THEN
				lamp_error := fbCovers.abCoverError[Selection];	
			ELSE
				IF direction_down THEN 
					fbCovers.Close();
				ELSIF direction_up THEN
					fbCovers.Open();
				END_IF		
			END_IF
```

The key-off branch (automatic mode, latched `Open()`/`Close()` while held) is unchanged. Reading
`IabCoverOpen`/`abCoverError` from outside stays allowed (read access to `VAR`).

**README** (`HalfBROT/README.md` l. 52-55 table row and l. 89-96): replace the `FB_CoverControl` bullet with:

> `FB_CoverControl` — three mirror covers, sequenced by limit switches: open 1->3->2, close 2->3->1, each cover
> starts when the previous one is at its end switch. All six outputs are recomputed every cycle; close has
> precedence over open; a cover in error (both switches active) or over `tMoveTimeout` (default 60 s, latched
> until `Reset()`) stops all covers and drops the command; a reversal waits `tReversalDeadTime` (1 s). Commands
> only through `Open()`, `Close()`, `Stop()`, `Reset()`. Used by MONETS and MONETN (since MONETcommon v0.10.0,
> which dropped its `FB_MonetCoverControl` copy).

and in the `FB_PendantControl` bullet: "selection 1-3 with the key on: hold-to-run open/close of all three
covers (single covers can no longer be moved)".

**Tests** (in `HalfBROTTests`, `POUs/FB_CoverControl_Tests.TcPOU`, fresh GUIDs). The limit switches are `AT%I*`
inside the block. A probe in the test project gets around that without moving the I/O:

```
// Test probe: sets the (unlinked) limit-switch inputs of FB_CoverControl from ST.
FUNCTION_BLOCK FB_CoverControlProbe EXTENDS HalfBROT.FB_CoverControl
VAR
END_VAR

METHOD SetSwitches
VAR_INPUT
	nCover	: UINT;
	bAtOpen		: BOOL;		// cover at its open switch (not bOpen: that name is the inherited command)
	bAtClosed	: BOOL;		// cover at its closed switch
END_VAR
IabCoverOpen[nCover] := NOT bAtOpen;		// switches are inverted (NC)
IabCoverClosed[nCover] := NOT bAtClosed;

METHOD FB_Init : BOOL		// same signature as the base; the base FB_Init runs implicitly
VAR_INPUT
	bInitRetains	: BOOL;
	bInCopyCode		: BOOL;
	comm			: I_Comm;
END_VAR
```

**Unverified** (first build/run decides): that a derived FB may write the inherited `AT%I*` variables, and that
an unlinked `%I` variable keeps a value written by the PLC. If not, keep only test 1 below and leave the rest to
step 2 of the #31 plan (I/O through structs). Multi-cycle tests keep their instance and state in the suite's
`VAR`, not in the method's `VAR` (a method `VAR` is re-initialised on every call: MONETcommonTests
`FB_MethodVarProbe_Tests`), and run with `tMoveTimeout := T#500MS`, `tReversalDeadTime := T#200MS`.

| Test | Asserts |
|---|---|
| 1 `Unlinked_Hardware_Reports_Error` (moved from MONETcommon) | `bError` TRUE with nothing linked |
| 2 `Closed_Idle_No_Output` | all closed, no command: six outputs FALSE, `Closed`, no `bError` |
| 3 `Open_Sequence_1_3_2` | from closed, `Open()`: only `OabOpenCover[1]`; cover 1 between: still only 1; cover 1 open: only 3; cover 3 open: only 2; all open: no output, `Opened`, `bOpen` FALSE |
| 4 `Close_Sequence_2_3_1` | mirror of 3 |
| 5 `Never_Both_Directions` | in every cycle of 3, 4, 6: `NOT (OabOpenCover[i] AND OabCloseCover[i])` |
| 6 `Reversal_Dead_Time` | opening cover 1, `Close()`: open output off in the next cycle, close output off for `tReversalDeadTime`, then on |
| 7 `Close_Wins_Over_Open` | `Open()` then `Close()` in one cycle: only close outputs |
| 8 `Timeout_Stops_And_Latches` | `Open()`, switches unchanged: after `tMoveTimeout` output off, `abCoverTimeout[1]`, `bError`; `Reset()`: error clears, nothing moves until a new `Open()` |
| 9 `Error_Stops_All_And_Drops_Command` | closing, cover 2 both switches active: all outputs off, cover 3 does not start, `bOpen`/`bClose` FALSE; after `Reset()` with switches fixed: nothing moves |
| 10 `Stop_Drops_Outputs` | opening, `Stop()`: all outputs off next cycle, no dead time involved, no error |

Tests 3-9 pin down exactly MONETcommon's behaviour, so they also prove the port; test 10 covers the new method.
TcUnit is sized to 32 suites / 32 tests.

**Release:** HalfBROT minor, **v0.9.0** (HalfBROT is at 0.8.0; breaking, so a minor bump like the earlier breaking changes),
Breaking for external users: `OabOpenCover`/`OabCloseCover` are no longer inputs, the delay
timers are gone, the sequence and pendant behaviour change (see Risks, Potsdam). Breaks nothing on an IAG
telescope alone: no IAG application or MONETcommon uses HalfBROT's `FB_CoverControl`/`FB_PendantControl`
(MONETS/MONETN keep running `FB_MonetCoverControl` until step 2/3).

### MONETcommon step 2: drop `FB_MonetCoverControl`, pendant on HalfBROT's block (one PR)

Branch from `fix/pendant-error-lamp` (draft PR #68, `9527380`) while #68 is open, because both edit
`FB_MonetPendantControl`; retarget/rebase onto `develop` once #68 is merged. Bump `ProjectVersion` to the next minor (**0.10.0**; develop is at 0.9.10)
on the branch before any on-site install (MONETcommon #69 comment: a branch build with the released version
number overwrites the release in the library repository).

1. Delete `MONETcommon/MONETcommon/MONETcommon/FB_MonetCoverControl.TcPOU` and its `<Compile Include="FB_MonetCoverControl.TcPOU">`
   (`MONETcommon.plcproj` l. 31-33).
2. `FB_MonetPendantControl.TcPOU` l. 7: `fbCovers : REFERENCE TO HalfBROT.FB_CoverControl;`
3. Pendant option B. Declarations, after `fbResetTrigger : R_TRIG;` (l. 26):

```
	// covers, key on: hold-to-run of the whole sequence (HalfBROT cover plan 2026-10-12)
	bCoverJogUp			: BOOL;
	bCoverJogDown		: BOOL;
	bCoverJogActive		: BOOL;		// the pendant started the running open/close and stops it on release
	fbCoverUpTrigger	: R_TRIG;
	fbCoverDownTrigger	: R_TRIG;
```

   Body, directly before `// enabled?` / `bEnable := IbEnableKey;` (l. 80-84), i.e. before the early `RETURN`,
   so key-off stops a run:

```
// covers, key on, selection 1-3: hold-to-run of the whole sequence (open 1->3->2, close 2->3->1); the cover block does
// the order, interlock, timeout and dead time. A run the pendant started stops when the button is released, the key
// goes off or the selector leaves 1-3 (#10, #3 for the covers). After an error a new press is needed (edge, not level).
bCoverJogUp := IbEnableKey AND nSelection >= 1 AND nSelection <= 3 AND IbDirectionUp AND NOT IbDirectionDown;
bCoverJogDown := IbEnableKey AND nSelection >= 1 AND nSelection <= 3 AND IbDirectionDown AND NOT IbDirectionUp;
fbCoverUpTrigger(CLK := bCoverJogUp);
fbCoverDownTrigger(CLK := bCoverJogDown);
IF __ISVALIDREF(fbCovers) THEN
	IF fbCoverDownTrigger.Q THEN
		fbCovers.Close();
		bCoverJogActive := TRUE;
	ELSIF fbCoverUpTrigger.Q THEN
		fbCovers.Open();
		bCoverJogActive := TRUE;
	ELSIF bCoverJogActive AND NOT bCoverJogUp AND NOT bCoverJogDown THEN
		fbCovers.Stop();
		bCoverJogActive := FALSE;
	END_IF
END_IF
```

   `CASE` branch `1, 2, 3` (l. 97-110) becomes (the `ELSE` was unreachable, #10):

```
	1, 2, 3:	// covers: lamps only; the hold-to-run is above the RETURN
		ObLampUp := fbCovers.Opened;
		ObLampDown := fbCovers.Closed;
		ObLampError := fbCovers.bError;
```

   The existing `TcUnit` pendant test (`Unlinked_Pendant_Lights_Error_Lamp`, PR #68) passes no references and
   all inputs read FALSE: the new code does nothing then (no edge, `__ISVALIDREF` FALSE), so it still passes.
4. Delete `MONETcommonTests/.../POUs/FB_MonetCoverControl_Tests.TcPOU`, its `<Compile>` entry
   (`MONETcommonTests.plcproj` l. 31-33), the instance `fbCoverControlTests` in the test `MAIN` (l. 6) and the
   README table row and the "Why `FB_MonetCoverControl`'s new movement timeout (#9) isn't tested yet" section
   (`MONETcommonTests/README.md` l. 13, l. 22-31); point to `HalfBROTTests` instead.
5. `README.md` l. 53: replace the `FB_MonetCoverControl` row with a sentence under the table: "The mirror covers
   use HalfBROT's `FB_CoverControl` (one copy since v0.10.0; needs HalfBROT >= the release from step 1)". Same in
   the pendant row: "selection 1-3 with the key on: hold-to-run open/close of all covers".
6. Release: MONETcommon **v0.10.0** (minor, breaking: `FB_MonetCoverControl` removed). If #68 is still unreleased,
   it ships in the same release (number then follows the next free minor).

**Alone (released and installed on `Becky`) this breaks the MONETS/MONETN build** (`FB_MonetCoverControl`
unknown), and it needs the step 1 HalfBROT installed (`Stop()`). So it is merged and released only together
with step 3.

**Neutral fallback** (if Tim wants the library swap without any pendant behaviour change): today's two pendant
lines only ever clear the commands. `bOpen := IbDirectionUp AND Opened` is TRUE only when all covers are open, and
the block then clears `bOpen` itself without driving anything (same for close); in every other case they write
FALSE, i.e. they cancel a running command each cycle while selection 1-3 is active with the key on. The exactly
equivalent code is `fbCovers.Stop();` in place of l. 101-102 (inside `IF IbEnableKey`), keeping #10 open. Not
recommended (it keeps a bug on purpose), but it lets steps 2-3 go out without a pendant test.

### MONETS and MONETN step 3: `MAIN` (one PR each)

MONETS `MONETS/MONETSRuntime/POUs/MAIN.TcPOU` l. 22 and MONETN `MONETN/MONETNRuntime/POUs/MAIN.TcPOU` l. 22, today:

```
	CoverControl		: FB_MonetCoverControl(comm := fbComm);
```

new:

```
	CoverControl		: HalfBROT.FB_CoverControl(comm := fbComm);	// mirror covers (HalfBROT; was MONETcommon FB_MonetCoverControl)
```

Qualified with the namespace on purpose (both `HalfBROT` and `MONETcommon` are referenced). Nothing else in
`MAIN` changes: `CoverControl();` (l. 119/117) keeps the defaults, so `tMoveTimeout` stays 60 s until the travel
times are measured (see Checks). README: MONETS l. 98, MONETN l. 16, l. 114, l. 166-167 (`FB_MonetCoverControl`
-> `HalfBROT.FB_CoverControl`).

**I/O links stay valid (verified from the files):** the links are by symbol path, not by type. MONETS.tsproj
has the four `<Var>` entries `MAIN.CoverControl.IabCoverOpen`/`IabCoverClosed` (l. 3011-3019, PlcTask inputs)
and `MAIN.CoverControl.OabOpenCover`/`OabCloseCover` (l. 3340-3348, outputs), typed `ARRAY [1..3] OF BOOL`; the
PDO entries `MAIN.CoverControl.*[n]` on "Mirror Cover IN (EL1008)" and "Mirror Cover OUT1/OUT2 (EL2004)"
(l. 5462-5551) and twelve `<Link VarA="PlcTask Inputs^MAIN.CoverControl.IabCoverClosed[1]" ...>` entries
(l. 11797-11813). MONETN.tsproj the same at l. 3457-3782, 5931-6020, 12580-12596 (Term 21). The instance name
(`CoverControl`), the variable names, their array type and their `AT%I*`/`AT%Q*` direction are unchanged, and
neither `.tsproj` contains the FB type name (`FB_MonetCoverControl` does not occur in them; no `.tmc` is tracked).
So the build regenerates the same symbols and the links resolve. The process-image offsets may move (TwinCAT
re-assigns them on build); that is why the links are checked in XAE before activation anyway.

**VISUs:** `MONETS/MONETSRuntime/VISUs/Cover.TcVIS` and `MONETN/MONETNRuntime/VISUs/Cover.TcVIS` reference
`MAIN.CoverControl.OabOpenCover[1..3]`, `OabCloseCover[1..3]`, `abCoverOpen/Closed/Error[1..3]`, `bAllOpen`,
`bAllClosed`, `bOpen`, `bClose`, `bReset`, `bError` and `bWarning`; HalfBROT's library `Cover.TcVIS` the same
minus the last six. All names exist in the new block except `bWarning`, which is **already** missing today
(removed in MONETcommon #9, v0.4.1), so nothing new breaks. A VISU can read and write `VAR` variables, so
`bOpen`/`bClose`/`bReset` moving from `VAR_INPUT` to `VAR` does not affect it. Whether the VISU elements write
those three or only show them was not decoded from the XML (**unverified**); if they write `bOpen`/`bClose`
directly, they bypass the methods exactly as today. No other VISU, GVL or DUT references the cover block.

**VISU, checked 2026-10-12 (Tim's question):** in the MONETS and MONETN `Cover.TcVIS` three `VisuFbElemDipSwitch`
elements are bound to `MAIN.CoverControl.bOpen`, `bClose` and `bReset`, so the VISU **writes these three flags
directly** (a VISU can only write variables, not call methods). They must therefore stay ordinary writable
symbols in the block: `VAR`, never `PRIVATE`/`PROTECTED` or properties, and under the same names. The VISU path is
a deliberate bypass of `Open()`/`Close()`/`Reset()`: switch on `bOpen` starts the sequence, switch off drops the
command and the outputs, `bClose` has precedence, completion clears the flag, `bError` clears both: all handled
inside the block, so it behaves as today. The #31 plan's "commands only through methods" cannot cover the VISU
until it has method-like buttons (e.g. `VisuFbElemButton` with an action calling a PROGRAM that calls the
method); not part of this plan.

`MAIN.CoverControl.bWarning` (read by the same VISU page) has not existed since MONETcommon v0.4.1 (and HalfBROT
removed its own in #56): the element shows nothing and the VISU build reports an unresolved variable, probably
the cause of one of the two known build warnings of MONETS/MONETN (**unverified**). Proposal: remove that element
in the two VISUs (nothing ever sets it again); XAE edit by Tim, a separate PR, part of step 3.

## Migration order and rollback

This changes safety-relevant motion (mirror covers) on two telescopes, so every step is built and checked
before the next, and the libraries are installed on each `Becky` in dependency order.

| Step | PR / release | Breaks if done alone |
|---|---|---|
| 0 | HalfBROT: `HalfBROTTests` + `tests.yml` | nothing |
| 1 | HalfBROT: new `FB_CoverControl` + pendant B + cover suite; release v0.9.0 | nothing on IAG telescopes; Potsdam behaviour (sequence, timeout, pendant) |
| 2 | MONETcommon: drop `FB_MonetCoverControl`, pendant B, tests moved; branch, ProjectVersion 0.10.0, **not merged** | released alone: MONETS/MONETN build fails (type unknown); needs step 1 installed |
| 3 | MONETS, MONETN: `MAIN` l. 22, README; branches, **not merged** | needs step 2 installed |
| 4 | Tim, XAE: build MONETS and MONETN against steps 1-3 (0 errors, warnings no more than the 2 known), check the twelve cover links per telescope | - |
| 5 | On site: install, activate, checks below; passed -> merge 2 (after #68), release MONETcommon v0.10.0, merge 3, release MONETS (v1.5.0) and MONETN | - |

Install order on `Becky`: BROTLib, AstroBROT (unchanged), **HalfBROT (step 1) before MONETcommon (step 2)**, then
build the application. The applications reference everything as `*` (IAG #9), so the newest installed version
wins: a MONETcommon 0.10.0 installed without the step 1 HalfBROT fails to compile (`Stop()` unknown), and any
application built on that `Becky` afterwards picks up 0.10.0.

**Rollback** (cover or pendant misbehaves on site): in XAE Library Repository **uninstall MONETcommon 0.10.0**
(with `*` it would otherwise keep winning), reinstall the MONETcommon release that ran before the swap (write the Library Manager versions down before
activation; v0.9.10 is the current `develop` version, which release is installed on the controllers is unknown), check out MONETS `v1.4.0` (`67e683d`) / MONETN `5f94f89`, build, check the
cover links, activate. The step 1 HalfBROT can stay installed: MONETcommon v0.8.0 and the old `MAIN`s do not use
HalfBROT's `FB_CoverControl` or `FB_PendantControl`. Only if something else in that HalfBROT release is the
problem: uninstall it and reinstall HalfBROT **v0.8.0**. Record the library versions that ran (IAG #9).

**Timing (recommendation):** implement steps 0-3 now. Put the swap into the coming visit only as a separate,
last activation ("build C"), after T8 and T9 have passed on the planned build, so T8's results stay attributable
to MONETcommon v0.8.0; then run the swap checks below (about 30 min per telescope). If steps 1-3 are not green in
CI and XAE before the visit, do the swap at the next visit; nothing is lost, because the telescopes already run
this logic.

## Not changed

- The sequencing, timeout, dead-time and error logic MONETS/MONETN run today (byte-identical, see Checks).
- `I_MirrorCovers` (BROTLib): no `Stop()` added, so no BROTLib release and nothing for IAG50cm.
- `FB_ElevationControl` (`fbCovers : I_MirrorCovers`), `FB_MonetTelescopeControl`, the MQTT path, the telemetry
  topic `AUXILIARY.COVER.REALPOS` and its values.
- The `.tsproj` files and the VISUs (no edit needed; see step 3).
- IAG50cm's own `FB_CoverControl`.
- The HalfBROT pendant's key-off branch for the covers (`Open()`/`Close()` while held, latched); the other
  pendant selections (#5 for the axes stays open, MONETcommon #3 for the axes stays open).
- #31 plan steps 2-4. Its step 2 text says "cover/pendant are not linked [on MONETS/MONETN], so moving them costs
  nothing": **after this plan that is wrong for the covers**: HalfBROT's `FB_CoverControl` then carries twelve
  links per telescope. Update that plan when step 3 here is merged.

## Risks and open questions

**Risks**

1. **Potsdam** uses HalfBROT's `FB_CoverControl`/`FB_PendantControl` (HalfBROT #29 comments). For them: covers 1
   and 3 no longer start together and the close waits for full end positions (slower); a cover taking longer
   than 60 s now stops with an error; a reversal waits 1 s; any error stops all; the pendant can no longer move
   a single cover (option B); an application that writes `OabOpenCover`/`OabCloseCover` or passes them in the
   call no longer compiles. The fleet list says "Potsdam is not a constraint" (2026-10-11), the #29 addendum says
   "coordinate with Potsdam before a release". Release notes must say this.
2. **External writers of `bOpen`/`bClose`/`bReset`**: only MONETcommon's pendant in the code. pyobs-monet and the
   VISU element semantics are **unverified**; ADS/VISU writes keep working (same symbol paths), they just bypass
   the methods as today.
3. **Library install order and `*`**: a wrong order or a leftover newer version on `Becky` builds the wrong code.
   Mitigation: Library Manager versions written down before activation (test plan "XAE checks").
4. **Shared CI runner**: HalfBROT's and MONETcommon's `tests.yml` install branch builds into the runner's library
   repository; a MONETcommon CI run may resolve `HalfBROT, *` to a branch build. Same as today for MONETcommon;
   acceptable, but a red MONETcommon run right after a HalfBROT branch push should be re-run against `develop`.
5. **Probe subclass** (writing inherited `AT%I*`): unverified; fallback is the unlinked test only.
6. **Pendant B** is hold-to-run: the operator holds the button for the full travel (several tens of seconds per
   cover, real time unknown). In automatic mode with the key on, releasing the button calls `Stop()` and can
   cancel an `Open()`/`Close()` from `TelescopeControl`; `_PowerOn` stage 0 re-issues `Open()` every cycle, so the
   power-on recovers, a park's single `Close()` would not (**unverified** which park stages re-issue it). The
   key on in automatic mode is an operator error case already (MONETcommon #3).
7. Same POU Id in both libraries today: harmless (the hydraulics/focus copies coexisted the same way), and it
   disappears with step 2.

**Questions for Tim**

- Q1 Name: keep `FB_CoverControl` (recommended) or rename (e.g. `FB_MirrorCoverControl`)?
- Q2 Deprecated `FB_MonetCoverControl EXTENDS HalfBROT.FB_CoverControl` in MONETcommon for one release: no
  (recommended; only MONETS/MONETN use it, and it could not keep `bOpen`/`bClose` as inputs anyway)?
- Q3 Pendant: option B (recommended) or A (single-cover jog, code below)? For B: hold-to-run (recommended, like
  the axes) or latched (press once, the sequence runs to the end; drop the `Stop()` branch)?
- Q4 Tests: new `HalfBROTTests` with CI (recommended: tests where the code is, and a HalfBROT PR is tested before
  merge) or keep the cover suite in MONETcommonTests against whatever HalfBROT the runner has installed?
- Q5 Timing: as "build C" at the end of the coming visit (recommended if green in time) or the next visit?
- Q6 On-site test on both telescopes (recommended: wiring and travel times may differ) and by whom?
- Q7 Tell Potsdam before the HalfBROT release?
- Q8 `tMoveTimeout` rule once the times are known: 1.5 x the longest measured single-cover travel, rounded up to
  5 s (recommended), set per telescope in `MAIN`?
- Q9 HalfBROT release: v0.9.0 (recommended) or fold into a later minor?

**Option A (alternative to B), exact code.** Block, `VAR` additions:

```
	abManualOpen		: ARRAY[1..3] OF BOOL;	// ManualOpen() request, valid for one call of the block
	abManualClose		: ARRAY[1..3] OF BOOL;	// ManualClose() request, valid for one call of the block
	bManualAllowed		: BOOL;
```

Body, after the `// open 1->3->2` lines:

```
// manual single-cover jog (pendant), hold-to-run: the caller repeats ManualOpen()/ManualClose() every cycle, a request
// is valid for one call only. Never while a sequenced command runs or with an error; same interlock as the sequence,
// same dead time and timeout.
bManualAllowed := NOT bOpen AND NOT bClose AND NOT bError;
OabOpenCover[1] := OabOpenCover[1] OR (bManualAllowed AND abManualOpen[1] AND NOT abManualClose[1] AND NOT atofClose[1].Q AND NOT abCoverOpen[1] AND NOT abCoverTimeout[1]);
OabOpenCover[3] := OabOpenCover[3] OR (bManualAllowed AND abManualOpen[3] AND NOT abManualClose[3] AND NOT atofClose[3].Q AND NOT abCoverOpen[3] AND abCoverOpen[1] AND NOT abCoverTimeout[3]);
OabOpenCover[2] := OabOpenCover[2] OR (bManualAllowed AND abManualOpen[2] AND NOT abManualClose[2] AND NOT atofClose[2].Q AND NOT abCoverOpen[2] AND abCoverOpen[1] AND abCoverOpen[3] AND NOT abCoverTimeout[2]);
OabCloseCover[2] := OabCloseCover[2] OR (bManualAllowed AND abManualClose[2] AND NOT abManualOpen[2] AND NOT atofOpen[2].Q AND NOT abCoverClosed[2] AND NOT abCoverTimeout[2]);
OabCloseCover[3] := OabCloseCover[3] OR (bManualAllowed AND abManualClose[3] AND NOT abManualOpen[3] AND NOT atofOpen[3].Q AND NOT abCoverClosed[3] AND abCoverClosed[2] AND NOT abCoverTimeout[3]);
OabCloseCover[1] := OabCloseCover[1] OR (bManualAllowed AND abManualClose[1] AND NOT abManualOpen[1] AND NOT atofOpen[1].Q AND NOT abCoverClosed[1] AND abCoverClosed[2] AND abCoverClosed[3] AND NOT abCoverTimeout[1]);
FOR i := 1 TO 3 BY 1 DO
	abManualOpen[i] := FALSE;
	abManualClose[i] := FALSE;
END_FOR
```

Methods `ManualOpen(nCover : UINT) : BOOL` / `ManualClose(nCover : UINT) : BOOL` (fresh GUIDs):
`IF nCover >= 1 AND nCover <= 3 THEN abManualOpen[nCover] := TRUE; ManualOpen := TRUE; END_IF` (same for close).
Pendants: HalfBROT l. 135-136 and MONETcommon l. 101-102 become

```
				IF direction_up AND NOT direction_down THEN			// MONETcommon: IbDirectionUp/IbDirectionDown
					fbCovers.ManualOpen(nCover := Selection);		// MONETcommon: nSelection
				ELSIF direction_down AND NOT direction_up THEN
					fbCovers.ManualClose(nCover := Selection);
				END_IF
```

The request expires every call, so release, key off and selector change stop the cover without extra pendant
logic. Costs: 13 more lines in the safety-relevant block (the "byte-identical" proof then has one more listed
hunk), the selector 1-3 gets a new per-cover meaning on MONET, and a cover with a broken switch still cannot be
jogged (error). The interlock keeps the sequence's collision protection; dropping it (as in the old HalfBROT
pendant) is not proposed.

## Checks

**1. Port proof (before the HalfBROT PR is merged).** Extract the body and compare:

```
py=/c/Users/husser/AppData/Local/Programs/Python/Python313/python.exe
git -C MONETcommon show v0.8.0:MONETcommon/MONETcommon/MONETcommon/FB_MonetCoverControl.TcPOU > old.TcPOU
git -C HalfBROT show <branch>:HalfBROT/HalfBROT/HalfBROT/POUs/FB_CoverControl.TcPOU > new.TcPOU
for f in old new; do $py -I -c "import re,sys; s=open(sys.argv[1],encoding='utf-8-sig').read(); m=re.findall(r'<(Declaration|ST)><!\[CDATA\[(.*?)\]\]>',s,re.S); print(m[0][1]); print('=====BODY'); print(m[1][1])" $f.TcPOU > $f.st; done
diff -u --strip-trailing-cr old.st new.st
```

Expected: the body diff has exactly two hunks (`PT := tMoveTimeout` with its comment line; the telemetry block: HalfBROT's #19
timer lines and the changed comment); the declaration diff only items 1-4 of the list above. Any other hunk fails the check. Paste the
diff into the PR description. Methods: `diff` of `Open`/`Close`/`Reset`/properties empty, `_SendTelemetry` only
the guard, `Stop` new.

**2. CI.** HalfBROT `tcbuild.yml` (`Build complete -- 0 errors`), HalfBROT `tests.yml` (all cover tests pass);
MONETcommon `tcbuild.yml` and `tests.yml` on the step 2 branch (with the step 1 HalfBROT installed on the runner).

**3. XAE (Tim, before any activation).** MONETS and MONETN build against steps 1-3: 0 errors, no more than the 2
known warnings. Library Manager: HalfBROT = the step 1 version, MONETcommon = 0.10.0. I/O: twelve links per
telescope, `MAIN.CoverControl.IabCoverOpen[1..3]`/`IabCoverClosed[1..3]` on "Mirror Cover IN (EL1008)" channels
1-6 (open = odd, closed = even), `OabOpenCover[1..2]`/`OabCloseCover[1..2]` on "Mirror Cover OUT1 (EL2004)"
channels 1-4, `OabOpenCover[3]`/`OabCloseCover[3]` on "Mirror Cover OUT2 (EL2004)" channels 1-2 (MONETS Term 8,
MONETN Term 21); nothing unlinked.

**4. On site, per telescope** (one person watching the mirror; clear area; E-stop at hand). The cover checks
that already exist apply unchanged to the new block, because the logic is identical: test plan T8 step 5 (order
1->3->2 / 2->3->1, no false `bError`, reversal: outputs off at once, 1 s all off, other direction; travel times
against the 60 s timeout = MONETcommon #62 checks 1-3 and 7, MONETN #15 check 5), T8 step 6 (parked only:
cover 2's switch connector pulled while closing: all stop, `bError`, cover 3 does not start, nothing moves after
the reset = #62 checks 4-5), T8 step 9 (reset button clears covers). If T8 ran on v0.8.0 earlier in the same
visit, repeat only a-c below plus one open/close; otherwise run T8 steps 5-6 on build C. Add as **T15** to the
test plan:

- a. After activation, covers closed: online `MAIN.CoverControl.abCoverClosed[1..3]` TRUE, `bError` FALSE,
  `tMoveTimeout` = `T#60S`, `tReversalDeadTime` = `T#1S`; the forced/online values match the pre-update ones
  (same switch states read).
- b. Power-on (pyobs or MQTT) opens 1->3->2, park closes 2->3->1. **Write down the travel time of each cover in
  each direction** (from `OabOpenCover[i]` rising to `abCoverOpen[i]`); compare with T8 step 5 if it ran (same
  within about 10 %).
- c. One reversal while opening (`Close()` from MQTT/XAE): open output off at once, 1 s all off, then closing.
- d. Pendant, key on, selection 1: hold up: covers open 1->3->2 while held; release mid-travel: all outputs off
  in the same second; hold down: after 1 s dead time closing 2->3->1. Selection 2 and 3 behave the same. Lamps:
  up lit when all open, down lit when all closed, error lamp = `CoverControl.bError`.
- e. While a pendant run is going: key off -> stops; selector to 5 -> stops. Nothing restarts when the key or
  selector comes back without a new press.
- f. Key off (automatic): the direction buttons on selection 1-3 do nothing to the covers (unchanged
  `RETURN`).
- g. With a cover error present (only if T8 step 6 is done anyway, parked): holding a direction does not move
  anything; after reset, still nothing until the button is released and pressed again.
- h. **VISU Cover page** (MONETS and MONETN, from XAE or the target VISU): the `bOpen` switch starts the opening
  1->3->2, switching it off stops all outputs in the same second; the `bClose` switch closes 2->3->1 and wins over
  `bOpen` when both are on; the `bReset` switch clears a cover error; the output, limit and error indicators follow
  `OabOpenCover/OabCloseCover`, `abCoverOpen/Closed/Error`, `bAllOpen/bAllClosed/bError`. After a build: the number
  of VISU warnings (the missing `bWarning` element gone if it was removed).

Fail if: an order differs, a cover moves without a command, both outputs of one cover are on together, a
release/key-off/selector change does not stop a pendant run, or a travel time is longer than 2/3 of
`tMoveTimeout`.

**5. After the visit.** Set `tMoveTimeout` per telescope in `MAIN` (separate MONETS and MONETN PRs):
`CoverControl(tMoveTimeout := T#<n>S);` in place of `CoverControl();`, `n` = 1.5 x the longest measured
single-cover travel, rounded up to 5 s, with a comment giving the measured times and the date. Until then the 60 s
placeholder stays. `tReversalDeadTime` stays 1 s unless the drive documentation gives a different value.

**Issues and fleet list** (after the PRs, by whoever implements):

- HalfBROT #4: closed by the step 1 PR (`Closes #4`); comment that the hardware verification is MONETcommon #62 /
  test plan T15 on MONETS/MONETN.
- HalfBROT #5: comment, cover part fixed by step 1 (pendant hold-to-run stops on release/key/selector); stays
  open for the axes.
- HalfBROT #29 (closed): comment that the 2026-10-11 cover decision is superseded: one copy in HalfBROT.
- HalfBROT #32: comment, `HalfBROTTests` exists with the cover suite; stays open for hydraulics.
- MONETcommon #10: fixed by step 2 (option B); close after T15 d-g passed. Comment there with T15 d-g (it is the
  on-site issue for the pendant covers; no new issue).
- MONETcommon #3: comment, cover part done; stays open for the axes.
- MONETcommon #27, #28: the code now lives in HalfBROT; they close with #62 as planned. Comment on #62: the block
  is `HalfBROT.FB_CoverControl` from MONETcommon v0.10.0 on, checks unchanged, plus T15.
- MONETcommon #36: comment, the cover test moved to `HalfBROTTests`. MONETcommon #24 point 5 (forks with the
  same POU Id): comment, covers now one copy too.
- MONETcommon #69: coordinate, PR #68 goes first; this branch is based on it.
- MONETN #15 check 5: unchanged, applies to the new block; result into T15 b.
- IAG fleet list (`IAG/specs/steering/brotlib-fleet-open-items.md`): HalfBROT #4 row (fix planned, this plan),
  MONETcommon #10 row (option B), #62 row (block moves to HalfBROT), HalfBROT #32 row (`HalfBROTTests`), the
  2026-10-11 "covers kept in both" decision lines (superseded), and this plan under "Open plans / Sibling repos".
  Test plan: add T15 and the build C row.
