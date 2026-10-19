# Pendant: restart only in manual mode, offsets per press, error lamp, horn (#12)

**Status: Part A implemented on `develop` (PR linked from #12, CI build only), Part B proposed (branch, not
released).** Decided 2026-10-12 (Tim): B2 yes, horn option (a), on-site test on MONETS and MONETN. Nothing was run
on a PLC or a telescope. Facts below were read on 2026-10-12
from HalfBROT `develop` @ `6a065e0`, MONETcommon `develop` @ `38b5896`, MONETS `origin/develop` @ `67e683d` and
MONETN `develop` @ `5f94f89`. Line numbers refer to those commits.

Decided 2026-10-12 (Tim): fix all of #12 in one plan. HalfBROT's `FB_PendantControl` stays in the library (it is
not deleted); behaviour and interface changes to it are fine (Potsdam is not a constraint, the next HalfBROT
release is a minor anyway).

## Problem

### Who runs which pendant block

- HalfBROT `FB_PendantControl` (`HalfBROT/HalfBROT/POUs/FB_PendantControl.TcPOU`): **no IAG telescope
  instantiates it.** No POU/GVL/DUT/VIS in MONETS, MONETN or MONETcommon references it. MONETN.tsproj still
  has the old HalfBROT names on the terminal side of its links (`Channel 5^MAIN.PendantControl.lamp_error`
  etc., l. 12619-12623), but the PLC side is `MAIN.PendantControl.ObLamp*`, i.e. MONETcommon's block. IAG50cm
  has its own, unrelated `FB_PendantControl` (`IAG50cm/POUs/PendantControl/`).
- MONETcommon `FB_MonetPendantControl` (`MONETcommon/MONETcommon/FB_MonetPendantControl.TcPOU`): instantiated as
  `MAIN.PendantControl` in MONETS (`MAIN.TcPOU` l. 30, called l. 158) and MONETN (l. 29, called l. 156), I/O
  linked to the EL1008/EL1008/EL2008 terminals "Pendant Control IN/IN2/OUT" (MONETS.tsproj l. 11822-11841,
  MONETN.tsproj l. 12603-12623). MAIN passes the seven references but **not** `eTelescopeMode` (stays at its
  default; the block's own `IF eTelescopeMode = automatic THEN ; END_IF` is empty anyway). MAIN also reads
  `PendantControl.IbResetButton` (l. 88, l. 221) for the reset of covers/hydraulics/telescope/safety.

### HalfBROT `FB_PendantControl`

1. **Restart in automatic mode.** Selection 13, l. 254-256:
   `ResetDelay(IN := reset_switch, PT := T#3S);` and
   `TwinCATRestart(NETID := '', RESTART := ResetDelay.Q AND NOT fbHydraulics.SuctionRunning, TMOUT := T#60S);`
   sit after the `IF key_switch ... ELSE ... END_IF` (l. 227-253), so holding reset for 3 s restarts the local
   TwinCAT in automatic mode too. It is the only `TC_Restart` in HalfBROT, MONETcommon, MONETS, MONETN, BROTLib
   and IAG50cm (grep over all `.TcPOU`).
2. **Offsets per cycle.** Selection 13, key off (l. 243-252): `fbTelescope.fElevationOffset` /
   `fAzimuthOffset` (BROTLib `FB_AltAzTelescopeControl`, LREAL, degrees) change by `10.0/3600.0` on **every
   call** while a direction is held: about 1000 arcsec/s at a 10 ms task. `up_trigger`/`down_trigger`
   (`R_TRIG`, l. 31-32) are called every cycle (l. 101-102) but their `.Q` is never read, so one step per press
   was most likely intended. There are no left/right triggers.
3. **`Error` unused.** l. 92 `Error := (Selection = 0);` with the comment "should also trigger emergency stop";
   `Error` is a local `VAR` (l. 21) that nothing reads. The README (l. 101-105) already says "sets `Error`, but
   nothing acts on it yet (#12)" (corrected in PR #53).
4. **Error lamp never cleared.** l. 104-105: `// the error lamp is the same as the error lamp of the main panel`
   / `lamp_error := lamp_error;` is a no-op. `lamp_error` is assigned only in some branches (see the table in
   Part A), so it keeps the value of the last selection that wrote it. Since #16 the not-connected branch
   (l. 123-124) sets it TRUE, and nothing clears it again until a branch that assigns it runs.
5. **Horn.** `horn : BOOL` (l. 22) has no `AT %Q*`; it is set FALSE every cycle (l. 99) and `key_switch` in
   selection 14 (l. 258). Nothing sounds.

### MONETcommon `FB_MonetPendantControl`

- **Neither `TC_Restart` nor offset code** (confirmed: no `TC_Restart`, no `Offset`, no selection 11-15 in the
  block; the `CASE` covers 1-8 only). Points 1 and 2 do not exist on MONETS/MONETN.
- `bError := (nSelection = 0);` (l. 64-65), local, never read (same pattern as HalfBROT).
- `ObLampError := ObLampError;` (l. 77-78), same no-op and the same misleading comment.
- `bHorn : BOOL;` (l. 22) is a plain local, **no `AT %Q*`** (there is no `ObHorn`); it is only ever set FALSE
  (l. 72). No selection sets it TRUE.
- **Early return when the key is off** (l. 81-84): `bEnable := IbEnableKey; IF NOT bEnable THEN RETURN;` comes
  after the lamp lines and before the `CASE`. With the key off, the four direction lamps are FALSE (l. 68-71)
  and the error lamp keeps whatever it had. Consequence: the `ELSE` (automatic) part of the cover branch
  (l. 104-110) is unreachable (already MONETcommon #10/M4).
- Where `ObLampError` is written today (key on): 1-3 `fbCovers.bError` (l. 103); 5 `fbFocus.Error` if the
  reference is valid (l. 124); 6 `fbDerotator.Error` (l. 133); 7 `fbElevation.bError` (l. 145); **8 never**
  (azimuth errors are not shown); 4 and 9-15 have no branch.
- Nothing else reads `ObLampError` (grep over MONETS/MONETN/MONETcommon POUs, GVLs, VIS): the lamp is an output
  only, it cannot trigger a command.
- Hardware: `ObLampError` is EL2008 "Pendant Control OUT" channel 5 on both telescopes (MONETS Term 8 (EK1200),
  MONETN Term 21 (EK1200)). Channels 6, 7, 8 of that EL2008 are unlinked in both projects (entries named
  `Output`). Whether anything (a horn) is wired to them is **unknown**.

### Unknown (not answered by the code)

- Whether TwinSAFE or the enable key already stops anything when the pendant is unplugged. This plan adds no
  safety function and must not be described as one.
- Whether the error lamp sits on the pendant itself (the terminal is called "Pendant Control OUT", so most
  likely). If it does, an **unplugged** pendant cannot show its own error lamp; the change then only shows up
  as an online value (`MAIN.PendantControl.ObLampError`) and on a pendant whose selector is at position 0, if
  the selector has such a position (also unknown).
- Whether there is a physical horn and to which terminal/channel it is wired.

## Change

Two separately releasable parts. Part A changes nothing on any telescope. Part B runs on MONETS and MONETN and
is held back until Tim has tested it on site.

### Part A: HalfBROT `FB_PendantControl` (one PR to `develop`, closes #12 together with Part B)

All edits in `HalfBROT/HalfBROT/POUs/FB_PendantControl.TcPOU` (UTF-8 with BOM, keep it). No new method or
property, so no new GUID. The `<LineIds>` blocks are not edited by hand (#57 added `_ReferencesValid` without
touching them; XAE rewrites them on the next save).

**A1, declarations** (after `down_trigger`, l. 32):

```
	left_trigger:	R_TRIG;
	right_trigger:	R_TRIG;
```

and the comments of `Error`/`horn` (l. 20-22):

```
	// pendant disconnected or selector at 0 (Selection = 0); only lights the error lamp
	Error:		BOOL;
	// not mapped to an output (no AT %Q*): nothing sounds (#12)
	horn:			BOOL;
```

**A2, top of the body** (l. 91-92 and l. 101-105). Replace the comment of l. 91 and the two lamp lines:

```
// selector reads 0: pendant disconnected (or selector at 0). Only lights the error lamp, no stop (#12)
Error := (Selection = 0);
...
up_trigger(CLK := direction_up);
down_trigger(CLK := direction_down);
left_trigger(CLK := direction_left);
right_trigger(CLK := direction_right);
// error lamp: Error by default; the selected function overwrites it below with its own error
lamp_error := Error;
```

The triggers stay outside the `CASE` so they see every edge, also while another selection is active. Delete the
commented-out `//R_Enable(CLK := enable);` (l. 103) while there.

**A3, selection 13, key off** (l. 243-252): steps on the rising edge only:

```
			IF up_trigger.Q THEN
				fbTelescope.fElevationOffset := fbTelescope.fElevationOffset + 10.0/3600.0;
			ELSIF down_trigger.Q THEN
				fbTelescope.fElevationOffset := fbTelescope.fElevationOffset - 10.0/3600.0;
			END_IF
			IF left_trigger.Q THEN
				fbTelescope.fAzimuthOffset := fbTelescope.fAzimuthOffset - 10.0/3600.0;
			ELSIF right_trigger.Q THEN
				fbTelescope.fAzimuthOffset := fbTelescope.fAzimuthOffset + 10.0/3600.0;
			END_IF
```

One 10 arcsec step per press, signs unchanged. Holding a button does nothing more (no auto-repeat).

**A4, selection 13, restart only in manual mode** (l. 254-256):

```
			// restart TwinCAT: manual mode only, reset held 3 s, suction pump stopped (#12)
			ResetDelay(IN := key_switch AND reset_switch, PT := T#3S);
			TwinCATRestart(NETID := '', RESTART := key_switch AND ResetDelay.Q AND NOT fbHydraulics.SuctionRunning,
				TMOUT := T#60S);
```

Gating the inputs instead of moving the two calls into `IF key_switch`: `TON` and `TC_Restart` (rising edge on
`RESTART`) must keep being called so they see their input drop when the key goes off; inside the `IF` a timer
that had run out would keep `Q = TRUE` frozen and fire on the next manual cycle. `fbTelescope.bReset :=
reset_switch` (l. 226) stays as it is.

**A5, README** (`HalfBROT/README.md` l. 101-107, the `FB_PendantControl` bullet). Replace "Selection 0 means
the pendant is disconnected and sets `Error`, but nothing acts on it yet (#12)." with:

> Selection 0 (pendant disconnected) lights the error lamp, nothing else; the lamp otherwise shows the
> selected function's error. Selection 13 steps the pointing offsets by 10 arcsec per press in automatic mode
> and restarts TwinCAT after reset is held 3 s in manual mode only. `horn` is not mapped to an output.

and drop "horn" from the function list in l. 103 (or write "horn (not wired)").

**Error lamp per selection after Part A** (HalfBROT; "last" = keeps the previous value, today's behaviour):

| Selection | Key on: today -> new | Key off: today -> new |
|---|---|---|
| 0 | last -> **TRUE** | last -> **TRUE** |
| any, references not set (#16) | TRUE -> TRUE | TRUE -> TRUE |
| 1-3 covers | `abCoverError[sel]` (unchanged) | last -> FALSE |
| 4, 9, 10, 11, 14 | last -> FALSE | last -> FALSE |
| 5 focus | `fbFocus.Error` (unchanged) | last -> FALSE |
| 6 derotator | `fbDerotator.Error` (unchanged) | last -> FALSE |
| 7 elevation | `fbElevation.bError` (unchanged, also key off) | unchanged |
| 8 azimuth | `fbAzimuth.Error` (unchanged) | last -> FALSE |
| 12 telescope | `fbTelescope.bError` (unchanged) | last -> FALSE |
| 13 telescope | last -> FALSE | last -> FALSE |
| 15 hydraulics | `fbHydraulics.Error` (unchanged) | last -> FALSE |

The selector is 4 BCD bits, so 0-15 is complete. Horn: option (a) below (code unchanged apart from the
comment).

Check: CI build only (TcBuild, no TcUnit in HalfBROT yet). Real tests come with TcUnit (#32, plan
`2026-10-11-commands-state-machines-io.md`), which also covers HalfBROT #5 for this block.

### Part B: MONETcommon `FB_MonetPendantControl` (on a branch, not released before the on-site test)

All edits in `MONETcommon/MONETcommon/FB_MonetPendantControl.TcPOU` (UTF-8 with BOM, CRLF). No new method, no
new GUID; `<LineIds>` not edited by hand.

**B1, error lamp** (l. 64-65 and l. 76-78):

```
// selector reads 0: pendant disconnected (or selector at 0). Only lights the error lamp, no stop (#12)
bError := (nSelection = 0);
...
// error lamp: bError by default; the selected function overwrites it below with its own error
ObLampError := bError;
```

Delete `//R_Enable(CLK := enable);` (l. 76). The assignment stays **before** the `RETURN` (l. 81-84), so it
also applies with the key off. Comment of `bError` (l. 21): `// pendant disconnected or selector at 0; only
lights the error lamp`.

**B2, azimuth error in selection 8 (needs Tim's decision, recommended).** After B1, selection 8 with the key on
shows FALSE even when the azimuth has an error (today: whatever was left). Add one line to the `8:` branch,
like HalfBROT does and like selections 6/7: `ObLampError := fbAzimuth.Error;` (`Error` is the BROTLib
`FB_BaseAxis` property). Without B2 the lamp is simply dark on selection 8.

**B3, horn: option (a) (recommended until Tim knows whether a horn exists).** Code unchanged except the comment
of `bHorn` (l. 22): `// no AT %Q*, not wired: nothing sounds (#12)`. README (MONETcommon l. 72) gets "the horn
variable is not wired". The other options, for when the hardware question is answered:

- **(b) Real output.** Rename `bHorn` to `ObHorn AT%Q* : BOOL;` (next to `ObLampError`, l. 47), give it a
  function (MONETcommon has no selection 14; e.g. a `14:` branch with key on: `ObHorn := TRUE` and all four
  direction lamps TRUE, as a lamp/horn test like HalfBROT's), release MONETcommon, then in MONETS and MONETN
  link `MAIN.PendantControl.ObHorn` to the horn's terminal channel. Needs the terminal and channel **from the
  wiring plan** (the free EL2008 channels 6-8 are a guess, not a fact). Link it in XAE (I/O > Pendant Control
  OUT > Channel n > Change Link) and commit the `.tsproj`. If the link is written by script instead (new `<Var>`
  `MAIN.PendantControl.ObHorn` under the PlcTask outputs image, the PDO entry name, a `<Link VarA="PlcTask
  Outputs^MAIN.PendantControl.ObHorn" VarB="Channel n^...">` in the `OwnerB` of "Pendant Control OUT
  (EL2008)"), open the project in XAE and check the mapping before activating. Then a horn step in the
  on-site test.
- **(c) Remove** `bHorn` and `bHorn := FALSE;` from MONETcommon only. HalfBROT's block stays (never delete
  HalfBROT blocks); it could lose its local `horn` variable and the selection 14 `horn :=` line, but keeps
  selection 14's lamps.

**MONETcommon error lamp after Part B** (MONETS/MONETN):

| Selection | Key on: today -> new | Key off (early RETURN): today -> new |
|---|---|---|
| 0 | last -> **TRUE** | last -> **TRUE** |
| 1-3 covers | `fbCovers.bError` (unchanged) | last -> FALSE |
| 4, 9-15 (no branch) | last -> FALSE | last -> FALSE |
| 5 focus | `fbFocus.Error` (unchanged) | last -> FALSE |
| 6 derotator | `fbDerotator.Error` (unchanged) | last -> FALSE |
| 7 elevation | `fbElevation.bError` (unchanged) | last -> FALSE |
| 8 azimuth | last -> FALSE (with B2: `fbAzimuth.Error`) | last -> FALSE |

No other output changes: direction lamps, `bEnable` and every command to the axes/covers/focus/hydraulics are
untouched, and nothing reads `ObLampError`.

**B4, TcUnit test** (MONETcommonTests has **no pendant test**; suites today: `FB_MonetCoverControl_Tests`,
`F_AzimuthOffsetOnAxis_Tests`, `F_HomeBelowHorizon_Tests`, `FB_MethodVarProbe_Tests`). The pendant inputs are
`AT%I*` inside the block, not plain variables, so a test cannot set selector, key or buttons from ST (same
limit as the cover test, see `MONETcommonTests/README.md`). What can be tested is the unlinked state, which is
exactly the selection 0 / key off case:

- New `MONETcommonTests/MONETcommonTests/MONETcommonTests/POUs/FB_MonetPendantControl_Tests.TcPOU`
  (`EXTENDS TcUnit.FB_TestSuite`, fresh GUIDs for the POU and the method, e.g. Python `uuid.uuid4()`; do not
  copy GUIDs), modelled on `FB_MonetCoverControl_Tests.TcPOU`. One test `Unlinked_Pendant_Lights_Error_Lamp`:
  `fbPendant : MONETcommon.FB_MonetPendantControl(comm := fbComm)` with `FB_NullComm`, no references passed
  (safe: with the key off the block returns before the `CASE`, nothing is dereferenced), one call, then assert
  `fbPendant.ObLampError` TRUE, `fbPendant.bEnable` FALSE, `ObLampUp/Down/Left/Right` FALSE. Reading the
  internal `VAR`s from outside compiles (MONETS/MONETN `MAIN` already reads `PendantControl.IbResetButton`). It
  fails on today's code (`ObLampError` stays FALSE) and passes after B1.
- Add it to `MONETcommonTests.plcproj` (`<Compile Include="POUs\FB_MonetPendantControl_Tests.TcPOU">`), declare
  an instance in the test `MAIN`, add a row to the README table. TcUnit is sized to 32 suites, 5 are used then.
- This runs in CI: MONETcommon's `tests.yml` installs the branch's MONETcommon and runs MONETcommonTests on the
  self-hosted runner's user-mode runtime (last runs green on 2026-10-11). Tim can also run it in XAE.

**Release rule for Part B.** Implement on a branch (`fix/pendant-error-lamp`), PR to `develop` open but **not
merged** until the on-site test passed, so no other MONETcommon release picks it up untested. For the test,
Tim installs the branch build of MONETcommon on the telescope's engineering PC, builds and activates MONETS (or
MONETN) with it, runs the checklist, then either keeps it (merge, release, update MONETS/MONETN) or
re-activates the released version. Which telescope(s) the test must cover before release: Tim's decision (the
code is identical on both; the wiring may not be).

**On-site test issue** (file in **MONETcommon**, where the changed code lives; labels `on site`,
`hardware test`; links HalfBROT #12 and this plan). Checklist, watched at the pendant and online in XAE
(`MAIN.PendantControl.nSelection`, `bError`, `ObLampError`, `bEnable`):

1. Starting state: telescope idle in a safe state, nothing moving, hydraulics as needed for steps 4-5.
2. Selector at 0 (if the selector has that position), key off: error lamp **on**, direction lamps off.
   Key on: still on. Nothing moves.
3. Pendant unplugged (only if safe and allowed; TwinSAFE/enable-chain behaviour is unknown, note what
   happens): online `nSelection = 0`, `ObLampError = TRUE`; note whether anything else reacts (E-stop,
   TwinSAFE, telescope stop). Plug back in: lamp follows the selector again.
4. Key on, selector through 1-8 with no error present: error lamp **off** on every position (today it may
   stay lit from an earlier position). Direction lamps as before.
5. Key on, with a real error on one function (e.g. a focus or derotator error, whatever can be provoked
   safely): lamp on at that selection, **off** when turning to another selection without that error, on
   again when turning back. Repeat with the key off: lamp off on 1-15.
6. Selection 8 (only with B2): lamp shows an azimuth error, if one can be provoked safely; else check online
   that `ObLampError = fbAzimuth.Error`.
7. Reset button: with an error shown, press reset: the error clears as today (MAIN reset path unchanged) and
   the lamp goes off in the same selection.
8. During steps 2-7: no axis, cover, focus, derotator or hydraulics command and no movement caused by the
   selector/key changes beyond what today's code does (the lamp change itself must not trigger anything;
   stale motion commands after a selector change are the known MONETcommon #3 / HalfBROT #5, not this test).
9. Horn: with option (a) nothing to test, record whether a horn exists and where it is wired (answers the
   open question). With option (b): selection 14, key on, horn sounds and stops when the key goes off or the
   selector moves.

## Not changed

- Stale motion commands after a selector or key change and the missing interlock with automatic mode
  (HalfBROT #5, MONETcommon #3), the inverted cover open/close sense in the MONET pendant (MONETcommon #10),
  no debouncing of the BCD selector: separate issues, separate plans. If MONETcommon #3 is done first, B1 moves
  with the restructured lamp code; the behaviour stays the same.
- No error lamp for the selected function while the key is off (both blocks): the lamp shows only the
  "disconnected" error then. Changing that is a new feature, not #12.
- `fbTelescope.bReset := reset_switch` and `TelescopeControlEnable` in HalfBROT selection 13 run in both modes,
  as today.
- No safety function: selection 0 only lights a lamp.
- IAG50cm's own `FB_PendantControl` is not touched.

## Risks

- **Part A:** none on the telescopes (not instantiated). External users of HalfBROT see three behaviour
  changes: no restart from the pendant in automatic mode, offsets per press instead of per cycle, error lamp
  cleared on selection change.
- **Part B:** the error lamp now goes dark on selections without error, so an operator who relied on a lamp
  "left over" from an earlier selection no longer sees it (this is the intended fix; B2 closes the azimuth
  gap). A pendant with the selector at 0 now has the error lamp lit permanently. Low risk for motion:
  `ObLampError` is an output only and nothing reads it.
- **Unknown hardware:** if the pendant selector never reads 0 while connected, the new lamp state is only
  visible online.

## Checks

- **Part A:** CI TcBuild of HalfBROT on the PR branch; the log must end with `Build complete -- 0 errors, 0
  warnings`. No hardware check possible (not used on any IAG telescope); TcUnit later via #32.
- **Part B:** CI TcBuild and Tests of MONETcommon on the branch (build clean as above; the new
  `Unlinked_Pendant_Lights_Error_Lamp` passes, the existing suites still pass). XAE (Tim): MONETS and MONETN
  build against the branch's MONETcommon. Then the on-site test issue above; only after it passed: merge,
  MONETcommon release, MONETS/MONETN update.
- **Tim's decisions:** B2 (azimuth error in selection 8: yes/no), horn option (a)/(b)/(c) and, for (b), the
  terminal/channel from the wiring plan, which telescope(s) the on-site test covers.
