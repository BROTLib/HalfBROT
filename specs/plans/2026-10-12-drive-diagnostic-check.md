# Drive diagnostic check: one method, errors and warnings, SoE reset on reset once per error (#2)

**Status: implemented in the PR linked from #2 (CI build only), not merged; the release waits for MONETS #19 and the checks below.** Read from HalfBROT `develop` at `86bfee6` (BROTLib `6d6035d`, MONETcommon `origin/develop`
`38b5896`, MONETS `origin/develop` `67e683d`, MONETN `origin/develop` `5f94f89`). Nothing implemented, nothing
run on a PLC or a telescope. Q1 and Q2 below are recommendations for Tim to decide.

## Problem

`FB_AzimuthControl`, `FB_ElevationControl` and `FB_DerotatorControl` each carry:

```
bDiagnosticError := FALSE;
IF inDiagnostic < 16#D012 AND inDiagnostic > 16#D014 THEN
	bDiagnosticError := TRUE;
	nErrorID := inDiagnostic;
	bSoeReset := bReset;          // derotator only
END_IF
```

(azimuth lines 87-91, elevation 112-116, derotator 91-96 of the `.TcPOU` files). No value is both below D012 and
above D014, so `bDiagnosticError` is always FALSE, `diagnosticEvent` (`'<Axis> Axis Diagnostic Error: %s'`,
azimuth 112-117, elevation 143-148, derotator 126-131) never logs, `nErrorID := inDiagnostic` never runs, and the
derotator's `bSoeReset := bReset` is dead. The code came in as is with the first import (`e40e3db`/`787cb4b`,
2024-11-12); the history does not say what was meant or why only the derotator has the SoE line.

### What `InDiagnostic` is

`InDiagnostic AT%I* : UDINT` (`FB_AxisControl`) is linked in both telescopes to the AX5000 PDO entry
"Diagnostic number" (index `#x0186` = S-0-0390, 32 bit, default 0, cyclic in the AT):

| | Azimuth (AX5125) | Elevation (AX5125) | Derotator (AX5206, 2 channels) | Derotator NC axis uses |
|---|---|---|---|---|
| MONETS.tsproj | `AT^Diagnostic number` | `AT^Diagnostic number` | `AT 1^Diagnostic number` | `AT 2` / `MDT 2` |
| MONETN.tsproj | `AT^Diagnostic number` | `AT^Diagnostic number` | `AT 1^Diagnostic number` | `AT 1` / `MDT 1` |

**Mismatch on MONET/S (new finding):** the derotator's NC axis drives channel B (`AT 2`/`MDT 2`, PDO index
`#x1010`/`#x1018`), but `DerotatorControl.InDiagnostic`, `InTorque` and `InDigitalInputs` are linked to channel A
(`AT 1`, `#x0010`). So on MONET/S the derotator diagnostic (and the derotator torque warning) read the other
channel, which as far as the links show drives nothing. What channel A shows is unknown: if it sits in a permanent
F error (no motor configured), the fixed check would report a permanent derotator drive error on MONET/S. This is a
MONETS link problem, not a HalfBROT one; see Checks step 0 and Risks.

### AX5000 diagnostic numbers (Beckhoff Infosys, checked 2026-10-12)

[Basic knowledge of the diagnostic system](https://infosys.beckhoff.com/content/1033/ax5000_diag_messages/1788397067.html):
`0xF...` error, `0xE...` warning, `0xD...` information. An error "always causes the axis to be stopped" and needs
S-0-0099 (reset) after the cause is removed; with a warning or information the AX5000 does not react, and most
warnings need no reset. From the [message list](https://infosys.beckhoff.com/content/1033/ax5000_diag_messages/8875191051.html)
and the linked pages:

| Number | Beckhoff text | Class / reaction (Infosys) | Seen on MONET? |
|---|---|---|---|
| D000..D006 | ESC state Init/PreOp/Boot/SafeOp/Op/invalid | information | start-up |
| D010 / D011 | Axis state machine: Control section not ready / ready | information | start-up (likely) |
| D012 / D013 / D014 | Control and power section ready / Axis in operation / Axis halt | information | normal (D012, D013 in `NCError_TO_STRING`) |
| D015 | Axis state machine: Axis error | information, "no reset required" | not known |
| D101 / D102 | Hardware enable not active / Safety card active, AX5000 in the safe condition | information | likely with STO (AX5805), not verified |
| ECD0 / ECD1 | Positive / Negative limit switch warning | warning, no reset; cause "axis moved to the limit switch" | ECD0 in `NCError_TO_STRING` |
| ECD2 | Amplifier overload utilisation warning | warning | not known |
| ED00 / ED01 | Motor over-temperature / Overload - motor | warning | not known |
| ED42 | Over-temperature - Amplifier | warning | not known |
| ED43 | Under-voltage - DC link ("has not yet reached the default value") | warning, "power stage is already switched off, no reset" | in `NCError_TO_STRING` |
| F324 | Excessive position command value deviation | error, reaction reset, NC handling "Execute Reset-Command (S-0-0099)" | in `NCError_TO_STRING` |
| F601 | Calibrate commutation offset error | error | in `NCError_TO_STRING` |

The "Seen on MONET?" column only means someone added the number to BROTLib's `NCError_TO_STRING` (D012, D013,
ECD0, ED43, F324, F601), which suggests they were observed; there is no log or report in any repo that says so.

### Who uses the result

- `nErrorID` is also the `ErrorID` output of `fbAxis` (`ErrorID => nErrorID`, every cycle, before the diagnostic
  check). BROTLib `FB_Axis.ErrorID` is only ever set, never cleared (`FB_Axis.TcPOU` lines 269-302), so it holds
  the last NC error (or 0) forever. A diagnostic override therefore lasts exactly as long as the drive error.
- `axisEvent` (`'<Axis> Axis Error: %s'`, `NCError_TO_STRING(nErrorID)`) is called after the check, so with an NC
  error and a drive error at once it shows the drive code.
- MONETcommon `FB_MonetTelescopeControl` copies `fbAzimuth/fbElevation/fbDerotator.nErrorID` into its `nErrorID`
  only in the `IF <axis>.bError` chain (verified, lines 120-141), and publishes
  `POSITION.INSTRUMENTAL.ALT/AZ.ERROR_STATE` = `ErrorID` only while `Error` (else `'0'`).
- MONETcommon `FB_MonetDiagnostics` publishes `diagnostics/base/<Axis>ErrorID` (`UDINT_TO_STRING`, i.e. decimal)
  when `<axis>.bError` changes **and on every 30 s heartbeat**, so `nErrorID` is published even without `bError`
  (verified). With a drive error and no NC error, `<Axis>ErrorID` = e.g. 62244 (16#F324) while `<Axis>Error` =
  false.
- Nothing outside the three blocks reads `bDiagnosticError` or `diagnosticEvent` (grep over HalfBROT, MONETcommon,
  MONETS, MONETN incl. visualisations).
- `bDiagnosticError` does not feed `bError`. A drive error stops the drive (Beckhoff), and the NC normally reports
  it as 16#4650 "Drive hardware not ready to operate" ("the drive is in error state (hardware error)", TwinCAT NC
  Error Codes 2.5.1, section 2.7) when it next needs the drive. Whether the NC raises an axis error for a drive
  error while the axis is **not** enabled is not verified; in that case the diagnostic number is the only signal.

### `FB_EventLog` (BROTLib)

Logs (ADS log + `Comm.PublishLog` to MQTT) on the **rising** edge of `Trigger` with `OnMessage`; on the falling
edge **only if `OffMessage` is non-empty**. The current `diagnosticEvent` has no `OffMessage`, so it would log once
per error, nothing when it clears. A change of the code while `Trigger` stays TRUE (F324 -> F601) does not log
again.

## Decisions (recommendations for Tim)

### Q1: errors only, or warnings too? -> errors as ERROR + `nErrorID`; warnings as WARN only, without ECD0/ECD1

Recommendation: **middle way.**

- **F (`16#F000..16#FFFF`)**: `bDiagnosticError`, `nErrorID := code`, ERROR event, HINT `cleared` when it goes.
  An F error stops the drive and needs a reset; this is what the original check was evidently for.
- **E (`16#E000..16#EFFF`)**: `bDiagnosticWarning`, WARN event after 2 s, **`nErrorID` untouched** (the
  telescope's error ID and the `ErrorID` telemetry must not show a warning as an error). Reasons: the AX5000 does
  not react to warnings and most clear themselves, so as ERROR they would be misleading; but some say something
  useful that nothing else shows (ED43 DC link not charged = drive has no mains power; ED00/ED01/ED42 over-temperature
  /overload before the drive trips with an F error; ECD2 amplifier overload).
- **Excluded warnings: ECD0, ECD1** (limit switch). Azimuth and elevation home onto their limit/cam input
  (`bCalibrationCam := NOT inDigitalInputs.x`, `Enable_Positive := inDigitalInputs.x OR bHomeAxis`), so if the
  drive evaluates that input as a limit switch (ECD0 in `NCError_TO_STRING` suggests it does), every homing
  would produce a warning. The pendant already shows the limits (lamps).
- **2 s delay for warnings** (`TON`): overload/utilisation warnings can flicker during slews; a warning must
  stand 2 s to be logged. One WARN line per occurrence, no "cleared" line (`OffMessage` empty), so the noise is
  bounded by how often a warning really appears. ED43 at every mains-off (if MONET switches drive mains, not
  verified) would be one WARN per power-off: acceptable and informative.
- **D015** ("Axis error", information class): not flagged. With an F error pending the drive is expected to show the
  F number (the number shown is the current diagnostic); D015 alone is not documented as needing anything. **Not
  verified** which number S-0-0390 shows while an F error is pending; Checks step 6 records it.
- **0** (default before the drive is up, or PDO not valid at start-up): below `16#E000`, so neither condition
  fires. A lost drive is not detected here; the NC covers that (4650/4655, invalid IO data 4466).
- **Upper word**: S-0-0390 is 32 bit; the Infosys page does not give its bit layout, and SERCOS defines class bits
  in the upper half. The old code and `NCError_TO_STRING` assume plain `16#Dxxx`. Masking with `16#FFFF` makes the
  check independent of that (Checks step 0 reads the live value).

Errors only (`>= 16#F000`, the original proposal) is also fine and smaller; it loses the over-temperature/overload
early warnings. Warnings as ERROR (`>= 16#E000` into `bDiagnosticError`) is not recommended: ED43 or ECD0 would
put a non-error into `nErrorID` and the error telemetry.

### Q2: derotator `bSoeReset := bReset` -> keep the idea, but once per drive error, on the rising edge of a reset; derotator only for now

What the line does once the condition is fixed (traced, derotator order per cycle: `fbAxis(Reset := bReset)` ->
`IF fbAxis.ResetDone THEN bReset := FALSE` -> diagnostic check -> `_SoEReset`):

1. Operator resets (pendant selection 6 `fbResetTrigger.Q` -> `fbDerotator.Reset()`, or the TCS sets the telescope
   `bReset`, which calls `fbDerotator.Reset()` **every cycle while `fbDerotator.bError`** and stays TRUE until
   `NOT bError`, MONETcommon `FB_MonetTelescopeControl` lines 120-160, 502-504).
2. Same cycle: `MC_Reset` starts; the check sets `bSoEReset := TRUE`; `_SoEReset` goes 0 -> 1.
3. `MC_Reset` and the SoE reset run in parallel (Beckhoff's order is SoE reset first, then NC reset; step 4 below
   gives the NC reset afterwards anyway).
4. `_SoEReset` step 10 (success): `bSoEReset := FALSE`, **`bReset := TRUE`**.
5. Next cycle: `fbAxis` sees `bReset` TRUE (new `MC_Reset` edge if it had dropped); `MC_Reset` is not done in that
   cycle, so `bReset` is still TRUE at the check. **If the drive still shows an F error** (cause not removed, or not
   yet updated after the reset), `bSoEReset := bReset` = TRUE again -> another SoE reset -> step 4 -> 5 ...

So with a persisting drive error **one reset press starts an endless series of drive resets**, re-armed by the
`bReset := TRUE` that `_SoEReset` itself sets, and with the TCS reset held TRUE by MONETcommon it never stops
either. A successful SoE reset logs nothing, so the loop would be silent apart from the drive's diagnosis
history; each round also resets the NC axis and drops the derotator's calibration (`MC_ForceCalibration` re-home,
no motion, absolute encoder). A failed SoE reset ends with `bSoEResetFailed` and no `bReset`, so after a failure
the loop stops unless the TCS holds `bReset`. A further side effect: the plain assignment also writes
`bSoEReset := FALSE` while a drive error is shown and no reset is pending, which cancels an ADS request
`bSoEReset := TRUE` written between cycles (only when `nSoEResetStep = 0`).

Is the SoE reset needed at all? Beckhoff: `MC_Reset` "resets the NC axis; in many cases this also resets a
connected drive; depending on the bus system or drive type a separate drive reset may be necessary"
([MC_Reset](https://infosys.beckhoff.com/content/1031/tcplclib_tc2_mc2/70050955.html)); `FB_SoEReset` "executes a
drive reset (S-0-0099); no NC reset is executed", timeout "must be 10 s, the reset can take up to 10 s", and "with
multi-channel devices both channels may have to execute a reset"
([FB_SoEReset](https://infosys.beckhoff.com/content/1031/tcplclib_tc2_ncdrive/2304974347.html)). The AX5000
system manual's recovery sequence is: remove the cause, S-0-0099, wait for Op ("Wait for WcState is Ok"), then NC
reset ([AX5000 reset](https://infosys.beckhoff.com/content/1031/ax5000_system_doku_hw2/723731595.html)); F324's NC
handling is "Execute Reset-Command (S-0-0099)". So the SoE reset on a drive error follows Beckhoff's own sequence;
whether `MC_Reset` alone clears an AX5000 F error on this TwinCAT/NC version is **not verified** (#55 item 2 tests
`bSoEReset`, not `MC_Reset` alone; see Checks step 7).

Why only the derotator: not documented. Plausible: it is the two-channel AX5206 (Beckhoff's multi-channel note),
and its re-home after a reset is a motionless `MC_ForceCalibration` on the absolute encoder, while azimuth and
elevation re-home on their cam (motion; elevation has a brake). That makes an automatic drive reset cheap on the
derotator and less so on the other two.

Recommendation:

- **Keep it for the derotator, but edge-triggered and guarded**: an automatic SoE reset only on the rising edge of
  a reset request, only while the drive shows an F error, only when `_SoEReset` is idle, and **at most once per
  drive error** (re-armed only after the diagnostic has been free of errors for 10 s, so the short gap while the
  drive restarts does not re-arm it). A drive error that survives one SoE reset needs a person anyway (Beckhoff:
  fix the cause, power-cycle); `bSoEReset` over ADS stays available for a deliberate retry.
- **Do not extend to azimuth/elevation now.** Decide after #55 item 2 and Checks step 7 show whether `MC_Reset`
  alone clears an AX5000 F error; if it does not, the same flag can be set for them in one line each.
- **Not on MONET/S until the channel-A link is resolved** (Problem): on MONET/S the check reads channel A while
  `FB_SoEReset` resets the NC axis's channel B, so a channel-A error would cost one useless drive reset and re-home
  per error. The guard keeps it to one; fixing the MONETS link is the real fix.
- Dropping the line entirely is the safe alternative if Tim prefers no automatic drive reset at all; then the
  method's `bSoEResetOnReset` input is left out.

## Change

### `FB_AxisControl`: new declarations (in the existing `VAR`, after `bDiagnosticError`)

`bDiagnosticError` and `diagnosticEvent` stay where they are.

```
	// drive diagnostic (AX5000 S-0-0390 "Diagnostic number" in InDiagnostic), see _CheckDiagnostic (#2)
	nDiagnosticCode			: UDINT;		// InDiagnostic AND 16#FFFF: 16#Fxxx error, 16#Exxx warning, 16#Dxxx information
	bDiagnosticWarning		: BOOL;			// drive warning (16#Exxx), except the limit switch warnings ECD0/ECD1
	sDiagnosticText			: STRING(255);	// code and text for the events, e.g. 'F324: Excessive position command value deviation'
	nDiagnosticTextCode		: UDINT;		// code that sDiagnosticText was built for
	tonDiagnosticWarning	: TON;			// a warning must stand 2 s before it is logged
	diagnosticWarningEvent	: FB_EventLog;
	resetTrigger			: R_TRIG;		// rising edge of a reset request (automatic SoE reset)
	tonDiagnosticClear		: TON;			// drive free of errors for 10 s: re-arms the automatic SoE reset
	bAutoSoEResetArmed		: BOOL := TRUE;	// one automatic SoE reset per drive error
```

### `FB_AxisControl._CheckDiagnostic` (new method)

New `<Method Name="_CheckDiagnostic" Id="{...}">` element after `_SoEReset`. It needs a **new GUID** for `Id`:
add the method in XAE (Add > Method, it creates the GUID), or generate one (`[guid]::NewGuid()` in PowerShell,
`python -c "import uuid; print(uuid.uuid4())"`) and paste it in braces; never copy the `Id` of another method.
Add a matching `<LineIds Name="FB_AxisControl._CheckDiagnostic">` entry like the other members (XAE regenerates it
on save if missing; `_SoEReset` currently has none).

```
// Drive diagnostic (AX5000 S-0-0390 in InDiagnostic), call every cycle after fbAxis(...) and before axisEvent (#2).
// F (error): bDiagnosticError, nErrorID := code, ERROR event; the code overrides fbAxis's ErrorID while it lasts.
// E (warning, except the limit switch warnings ECD0/ECD1): WARN event after 2 s, nErrorID untouched.
// D (information, incl. D012..D015) and 0 (drive not up yet): nothing.
// bSoEResetOnReset: a reset request (rising bReset) while the drive shows an error also resets the drive (SoE,
// via bSoEReset/_SoEReset), at most once per drive error; re-armed after 10 s without a drive error, so the
// bReset that _SoEReset sets after the drive reset cannot start another one.
METHOD PROTECTED _CheckDiagnostic
VAR_INPUT
	sAxis				: STRING;	// axis name for the event log
	bSoEResetOnReset	: BOOL;		// derotator: SoE reset on a reset while the drive shows an error
END_VAR
```

```
nDiagnosticCode := InDiagnostic AND 16#FFFF;
bDiagnosticError := nDiagnosticCode >= 16#F000;
bDiagnosticWarning := nDiagnosticCode >= 16#E000 AND nDiagnosticCode < 16#F000
	AND nDiagnosticCode <> 16#ECD0 AND nDiagnosticCode <> 16#ECD1;
tonDiagnosticWarning(IN := bDiagnosticWarning, PT := T#2S);

IF bDiagnosticError THEN
	nErrorID := nDiagnosticCode;
END_IF

// event text, rebuilt only when the code changes: 'F324: Excessive position command value deviation', or only
// 'F415' when NCError_TO_STRING does not know the number (it returns the hex code then)
IF (bDiagnosticError OR bDiagnosticWarning) AND nDiagnosticCode <> nDiagnosticTextCode THEN
	sDiagnosticText := DWORD_TO_HEXSTR(nDiagnosticCode, 4, FALSE);
	IF NCError_TO_STRING(nDiagnosticCode) <> sDiagnosticText THEN
		sDiagnosticText := CONCAT(CONCAT(sDiagnosticText, ': '), NCError_TO_STRING(nDiagnosticCode));
	END_IF
	nDiagnosticTextCode := nDiagnosticCode;
END_IF

// automatic SoE reset. ResetDone covers an MC_Reset that finished in the cycle bReset rose (bReset is already
// FALSE again here then).
resetTrigger(CLK := bReset OR fbAxis.ResetDone);
tonDiagnosticClear(IN := NOT bDiagnosticError, PT := T#10S);
IF tonDiagnosticClear.Q THEN
	bAutoSoEResetArmed := TRUE;
END_IF
IF bSoEResetOnReset AND bDiagnosticError AND resetTrigger.Q AND bAutoSoEResetArmed AND nSoEResetStep = 0 THEN
	bSoEReset := TRUE;
	bAutoSoEResetArmed := FALSE;
END_IF

diagnosticEvent(
	Trigger 		:= bDiagnosticError,
	Level 			:= ADSLOG_MSGTYPE_ERROR,
	FormatString 	:= CONCAT(sAxis, ' drive error: %s'),
	OnMessage 		:= sDiagnosticText,
	OffMessage 		:= 'cleared',
	OffLevel 		:= ADSLOG_MSGTYPE_HINT,
	Comm 			:= fbComm);

diagnosticWarningEvent(
	Trigger 		:= tonDiagnosticWarning.Q,
	Level 			:= ADSLOG_MSGTYPE_WARN,
	FormatString 	:= CONCAT(sAxis, ' drive warning: %s'),
	OnMessage 		:= sDiagnosticText,
	Comm 			:= fbComm);
```

Notes on the code:

- `DWORD_TO_HEXSTR` (Tc2_Utilities, referenced by HalfBROT) with `bLoCase := FALSE` gives upper case, the same
  format `NCError_TO_STRING` uses for unknown numbers, so the comparison detects "unknown".
- `bSoEReset := TRUE` (not `:= bReset`): never clears an external request.
- `bDiagnosticError` keeps its name (VAR, nobody outside reads it); `bDiagnosticWarning` is new.

### Messages (ADS log and MQTT `PublishLog`)

| Situation | Level | Text |
|---|---|---|
| F error appears | ERROR | `Elevation drive error: F324: Excessive position command value deviation` |
| unknown F error | ERROR | `Azimuth drive error: F415` |
| F error gone (after a reset) | INFO (HINT) | `Elevation drive error: cleared` |
| E warning stands 2 s | WARNING | `Derotator drive warning: ED43: Under-voltage - DC link.` |
| ECD0/ECD1, D..., 0 | none | |

Old text was `'<Axis> Axis Diagnostic Error: %s'` with `NCError_TO_STRING(inDiagnostic)`; it never appeared, so
no log consumer depends on it.

### Replacement in the three blocks

Azimuth (`FB_AzimuthControl`, lines 87-91), replace

```
bDiagnosticError := FALSE;
IF inDiagnostic < 16#D012 AND inDiagnostic > 16#D014 THEN
	bDiagnosticError :=TRUE;
	nErrorID := inDiagnostic;
END_IF
```

by

```
// drive diagnostic: error -> nErrorID and ERROR event, warning -> WARN event (#2)
_CheckDiagnostic(sAxis := 'Azimuth', bSoEResetOnReset := FALSE);
```

and delete the `diagnosticEvent(...)` call (lines 112-117).

Elevation (`FB_ElevationControl`, lines 112-116, same five lines): `_CheckDiagnostic(sAxis := 'Elevation',
bSoEResetOnReset := FALSE);`, delete `diagnosticEvent(...)` (lines 143-148).

Derotator (`FB_DerotatorControl`, lines 91-96):

```
bDiagnosticError := FALSE;
IF inDiagnostic < 16#D012 AND inDiagnostic > 16#D014 THEN
	bDiagnosticError := TRUE;
	nErrorID := inDiagnostic;
	bSoeReset := bReset;
END_IF
```

by

```
// drive diagnostic (#2); a reset while the drive shows an error also resets the drive (SoE), once per error
_CheckDiagnostic(sAxis := 'Derotator', bSoEResetOnReset := TRUE);
```

and delete `diagnosticEvent(...)` (lines 126-131). The call stays where the old block was: after `fbAxis(...)` and
the `ResetDone` handling, before `axisEvent` (all three) and, in elevation/derotator, before `_SoEReset`, so an
automatic request starts in the same cycle. Line numbers shift by the deletions; LineIds of the three blocks are
regenerated by XAE.

### Effect

| Block / case | Before | After |
|---|---|---|
| any axis, no drive error (D0xx, 0) | nothing | nothing |
| az/el, drive F error, no NC error | nothing; `nErrorID` = old NC ErrorID | ERROR `<Axis> drive error: <code>: <text>`; `nErrorID` = code; `diagnostics/base/<Axis>ErrorID` shows it on the next 30 s heartbeat (decimal), `<Axis>Error` stays false; telescope `nErrorID`/`ERROR_STATE` unchanged (they follow `bError`); no motion change |
| az/el, drive F error + NC error (e.g. 4650) | `axisEvent` with the NC text, `nErrorID` = NC code | `axisEvent` text and `nErrorID` show the drive code (more specific); telescope `nErrorID`, `ERROR_STATE` and `<Axis>ErrorID` show the drive code; plus the drive error event |
| derotator, drive F error | as az/el; SoE line dead | as az/el; **one** SoE reset on the first reset (pendant or TCS) after the error appeared; then `bReset`, re-home with `MC_ForceCalibration` (no motion); no further automatic SoE reset until the drive has been free of errors for 10 s |
| any axis, drive error cleared | - | HINT `<Axis> drive error: cleared`; `nErrorID` back to `fbAxis`'s ErrorID (sticky, last NC error or 0) |
| any axis, E warning (not ECD0/1) >= 2 s | nothing | one WARNING line; `nErrorID`, `bError`, telemetry unchanged |
| az/el homing onto the limit input (ECD0/ECD1) | nothing | nothing |
| MONET/S derotator | reads channel A (unused?) | same reading; **if channel A shows an F error, a permanent derotator drive error is logged once and `nErrorID` shows it** (Checks step 0) |

## Not changed

- `bDiagnosticError` does not feed `bError`: a drive error alone does not stop the telescope in software (the drive
  has stopped itself; the NC reports 4650 when it needs the drive). Changing that is a separate decision.
- `_SoEReset` itself (#14). Side finding: it passes `Timeout := DEFAULT_ADS_TIMEOUT` (5 s in Tc2_System, from
  memory, not checked) while Beckhoff says the `FB_SoEReset` timeout "must be 10 s"; worth its own small issue.
  Also Beckhoff's note that on multi-channel devices both channels may need a reset (AX5206) is not handled.
- MONETS link of the derotator's `InDiagnostic`/`InTorque`/`InDigitalInputs` to `AT 1` (MONETS repo, separate issue
  if step 0 confirms). MONET/N links `DerotatorControl.InTorque` to `AT 1^Actual torque/force generating current`,
  MONET/S to `Torque/force feedback value` (different quantities, not in scope).
- IAG50cm (own blocks, no diagnostic check).
- Event on a code change while the error persists (F324 -> F601): `FB_EventLog` fires on the edge only, so the
  second code is not logged; `nErrorID` shows it. Acceptable; noted in the method comment if wanted.

### Optional: `NCError_TO_STRING` (BROTLib, separate PR)

Only numbers that are normal states on MONET or the counterpart of one already there, with Beckhoff's text:

| Number | Text (Infosys) | Why |
|---|---|---|
| 16#D010 | Axis state machine: Control section not ready | start-up state |
| 16#D011 | Axis state machine: Control section ready | start-up state |
| 16#D014 | Axis state machine: Axis halt | normal state, between D012/D013 already listed |
| 16#D015 | Axis state machine: Axis error | state while in error |
| 16#ECD1 | Negative limit switch warning. | counterpart of ECD0 (listed) |

Not needed for this change (unknown numbers print as hex, and the event shows the hex code anyway). No other
number has evidence of occurring on MONET; add them when a log shows them.

## Risks

- **MONET/S derotator reads channel A.** Without step 0, a permanent channel-A error would appear as a permanent
  derotator drive error, `DerotatorErrorID` in telemetry, and one SoE reset + re-home of the wrong channel per
  error. Mitigation: step 0 before the release reaches MONET/S; if channel A is in error, fix the MONETS links
  first (or pass `bSoEResetOnReset := FALSE` until then, which needs a telescope-specific input).
- **A real drive error that was invisible now shows up** in logs and `nErrorID`. That is the point, but the first
  days after the release may show errors that have been there all along (e.g. on an idle axis).
- **Automatic SoE reset on the derotator** is new behaviour (dead until now): one drive reset per drive error on
  the first reset press. Guarded against loops; worst case one extra re-home without motion.
- **Upper 16 bits of S-0-0390** unknown; masked, so only the event text could miss information carried there.
- Force test: a forced `InDiagnostic` is cleared by logging out of XAE or "Unforce values"; forgetting it leaves a
  fake drive error (only logs/telemetry, no motion effect on azimuth/elevation).

## Checks

0. **Before the merge, read-only, both telescopes** (ADS or XAE online, no writes): `MAIN.AzimuthControl.InDiagnostic`,
   `MAIN.ElevationControl.InDiagnostic`, `MAIN.DerotatorControl.InDiagnostic` with the telescope idle and, if
   possible, enabled. Expected 16#D012/D013/D014 (53266/53267/53268), upper word 0. On MONET/S also read the
   derotator drive's `AT 2` "Diagnostic number" in the XAE I/O tree and compare with `AT 1`. Report the values.
1. CI: TcBuild of HalfBROT on the PR branch; the log must end with `Build complete -- 0 errors, 0 warnings`
   (the workflow accepts exit code 1 = warnings, so read the log, do not rely on the green check).
2. XAE (Tim): MONETS and MONETN build against the HalfBROT build with the change, 0 errors, 0 warnings.
3. Hardware: item 6 of #59 (text below), remote, elevation, telescope parked, no motion.
4. Optional, on MONET/N (not MONET/S before step 0 is resolved), after #55 passed: derotator, force
   `MAIN.DerotatorControl.InDiagnostic := 16#F324`, press reset once (pendant selection 6 or a TCS reset):
   exactly one SoE reset (one `bSoEReset` pulse, one entry in the drive's diagnosis history), `bReset`, derotator
   `Calibrated` again without motion; press reset again while still forced: **no** second SoE reset. Release the
   force, wait 10 s, force again, reset: one SoE reset again.
5. Watch the logs of both telescopes for a week after the release for `drive error`/`drive warning` lines.
6. Record which number S-0-0390 shows during a real F error (F code or D015) if one happens.
7. Open question for #55 item 2: with a real drive F error, does `MC_Reset` alone (reset without `bSoEReset`)
   clear it? Decides whether azimuth/elevation get `bSoEResetOnReset := TRUE`.

Text for item 6 of #59:

> **6. Drive diagnostic check (#2, remote, elevation, telescope parked)**
> In XAE online on the PLC project, **force** (PLC > Force values) `MAIN.ElevationControl.InDiagnostic` to
> `16#F324` (62244). Forcing holds against the input update: the forced value is applied after the inputs are read,
> each cycle ([Force values](https://infosys.beckhoff.com/content/1033/tc3_userinterface/2531616395.html)). Do not
> use the derotator (its reset would start a drive reset).
> - Expected: one ERROR line `Elevation drive error: F324: Excessive position command value deviation` (ADS log
>   and MQTT log), `ElevationControl.bDiagnosticError = TRUE`, `ElevationControl.nErrorID = 62244` while forced,
>   `ElevationControl.bError` unchanged (FALSE), no motion, within 30 s `diagnostics/base/ElevationErrorID = 62244`.
> - Force `16#ED43`: after 2 s one WARNING `Elevation drive warning: ED43: Under-voltage - DC link.`, `nErrorID`
>   back to its previous value. The F324 -> ED43 switch also logs `Elevation drive error: cleared`.
> - Force `16#ECD0`, then `0`: no log line.
> - Release with PLC > **Unforce values** (check that no forced value is left before logging out). Expected:
>   `nErrorID` back to its previous value, no further lines.
> - Fail if: no line, more than one ERROR line for one force, `nErrorID` not 62244, `bError` TRUE, or any line for
>   ECD0/0.
> Optional (MONET/N, after #55): derotator automatic SoE reset, see check 4 in
> `specs/plans/2026-10-12-drive-diagnostic-check.md`.

If forcing an `AT%I*` input does not hold on this TwinCAT version (value jumps back), force the input at the
source instead: the drive's `AT^Diagnostic number` in the XAE I/O tree (online tab, Force); this does not affect
the NC, which does not link the diagnostic number.
