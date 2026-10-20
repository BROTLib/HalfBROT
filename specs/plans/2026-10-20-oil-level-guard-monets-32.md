# Oil level plausibility guard (MONETS#32)

Date: 2026-10-20. Status: **HalfBROT part implemented** (PR on `fix/oil-level-guard-32`), not compiled in XAE, not tested on a PLC. The MONETcommon tests and the `MAIN` settings of MONETS/MONETN are still to do (section 6).
Repos: BROTLib/HalfBROT (logic), MONETcommon (TcUnit tests), MONETS and MONETN (`MAIN`). Main file: `HalfBROT/HalfBROT/POUs/FB_HydraulicsControl.TcPOU`.

## 1. Problem
`InOilLevel` (INT, `AT%I*`) is PDO `OilLevel` of the EL3164 "Mirror Temperature IN". It is a **0..10 V** input (confirmed by Tim, the issue text says 4..20 mA by mistake), so there is no live zero in the input: a cut wire reads about 0.

**The sensor** (Reventec LS100/LS range, capacitive, datasheets `LS100-Drawing.pdf` and `LS-Range-Liquid-Level-Sensors.pdf`; Tim named them and confirmed the same model on both telescopes, the type plate on the pan was not checked): analog output 0..5 V, standard calibration **0.25 V empty to 4.75 V full**, supply 6..30 V, accuracy +-1 % of full scale, 10 bit. Analog error condition: **0.1 V (programmable)**; the datasheet does not say exactly when it is output, presumably on an internal fault. At 10 V = 32767 counts that is 0.25 V = 819, 4.75 V = 15563, 5 V (the highest the sensor can output) = 16383, 0.1 V = 328. The measured empty pan (883..1330) fits the 0.25 V offset. A cut signal wire or a lost supply reads 0 V; the sensor's own fault reads about 0.1 V (raw ~330).
The level is `fOilLevel := F_YREAL(InOilLevel, nMinPanLevel, nMaxPanLevel, 0, 100, cut := FALSE)` and was never checked.
MS-02 (2026-10-20, MONET/S): `InOilLevel` forced to 0 gave -10.59 %, published as `OilLevel=-10.588236` 8 times in 40 s, no error, warning, flag or log entry.
Over-range was not tried (it would start the suction pump on a nearly empty pan).

Raw values measured on MONET/S (scale 900..9400):

| raw | meaning |
|---|---|
| 0 | broken wire (forced) |
| 883..1330 | empty pan |
| 2302 / 6887 | suction stop / start |
| 9416 | pan-full float switch trips |
| 328 | sensor error output 0.1 V (datasheet, not measured) |
| 819 | empty at 0.25 V (datasheet standard calibration) |
| 15563 / 16383 | 4.75 V (probe full) / 5 V (the sensor's highest output); 16383 is also the old library default of `nMaxPanLevel`, not measured |
| 32767 | EL3164 full scale (10 V), unverified against the Beckhoff documentation |

MONETN (scale 3000..10200) has never been measured; its 3000 is suspect, the MONET/S 3500 of the same origin was wrong.

## 2. Fix (this PR)
Two pure functions in HalfBROT, called from `FB_HydraulicsControl` (MONETcommon depends on HalfBROT, so they cannot live there):
- `F_OilLevelValid(nRaw, nMinValid, nMaxValid)`: both limits inclusive.
- `F_OilSuctionLatch(bLatched, bLevelValid, fLevel, fMinPercent, fMaxPercent, bPanMaximum, bMainPumpEnabled)`: replaces `srSuctionTrigger`.

New inputs `nMinValidPanLevel := -32768`, `nMaxValidPanLevel := 32767` (the defaults accept every INT: no guard, the old behaviour; the telescopes set the limits in `MAIN`), new output `bOilLevelInvalid`, `statusWord` bit 14.

| Condition | `OilLevel` publish | Suction start | Suction stop | Log | Flags |
|---|---|---|---|---|---|
| valid | yes, also on a change of more than 5 % | float switch or level > max | level < min | none | bit 14 = 0 |
| invalid | **no**, the 5 % trigger is ignored | **float switch only** | none by level; a running suction is kept while `ObOilPumpEnable`, released when the main pump is switched off | one WARN at the rising edge with the raw value | `bOilLevelInvalid`, bit 14 |
| invalid to valid | yes (the status word change triggers it) | normal | normal | one HINT 'OK' | cleared |

The hydraulics failure message says 'oil level invalid, raw ...' when the level is invalid.

## 3. Decisions (Opus recommendations, taken)
- **D1 suction while invalid:** level-based start blocked, level-based stop ignored, the float switch still starts it; the latch is released when `ObOilPumpEnable` is FALSE. Without that release a wire break during a suction would keep it running all night with the hydraulics off (the latch does not depend on `bEnable`). `ObOilPumpEnable` and not `bPumpRunning`: the float switch resets `rsOilPumpState`, so the pump feedback drops for a moment; in the scan where the switch clears, `rsOilPumpState` (evaluated before the latch) has set the enable again, so a float-started suction is not dropped. Rejected: "invalid resets the latch" (cycles at the float switch, every trip stops the main pump) and "suction on while invalid" (starts on an empty pan).
- **D2 flag:** new `bOilLevelInvalid` and bit 14, not `bOilWarning` (that is the physical reserve switch, published as `OilReserve`). No new MQTT field.
- **D3:** a warning, not an error.
- **D4 cutoffs (raw counts, in `MAIN`), revised after the sensor datasheet:** the lower limit must be **above the sensor's own error output (0.1 V, raw ~330)** and below the empty level (819 by the datasheet, 883 measured): MONET/S `nOilLevelMinValid := 600` (about 0.18 V; 300 would have missed the sensor error). It catches a cut wire, lost supply (0) and the sensor error (~330). `nOilLevelMaxValid := 12000` (float switch 9416, about 27 % margin; the sensor cannot output more than 5 V = 16383, so anything above that is a wiring fault); MONET/N 600 / 16000: Tim confirmed both telescopes have the same sensor model, so the electrical limits (error output, 0.25 V empty, 5 V maximum) apply to both; the MONET/N pan itself (empty and float switch raw values, scale 3000..10200) has still not been measured, so check that its empty reading is above 600 before it goes live.
- **D5 tests:** HalfBROT has no test project, so the suites go into MONETcommonTests (it references HalfBROT) and move once `HalfBROTTests` exists.
- **D6:** the terminal's `Status__Underrange/Overrange/Error` bits stay unlinked: the raw limits already catch the cases and a new `AT%I*` would clash with the I/O struct move (#31/#32).

## 4. Backward compatibility
With the default inputs every INT is valid, `F_OilSuctionLatch` equals the old SR, publishing is unchanged and bit 14 stays 0. The only rename is `srSuctionTrigger` to `bSuctionLatched`. Only MONETcommon, MONETS and MONETN reference HalfBROT (IAG50cm, AstroBROT, MONETRoof do not). MONETS/MONETN must pin the release that contains this change, otherwise CI builds the old library and the new inputs do not compile.

## 5. Risks
- A real overfull pan above the upper limit counts as invalid: the level start is blocked and only the float switch remains.
- A sensor stuck inside the valid range is not detected (the 140 s hydraulics watchdog is the net).
- A latched suction may run partly dry while the level is invalid; the suction pump's dry-run tolerance is **unknown**.
- If the level goes invalid while the suction is off, either the float switch or the 140 s watchdog trips first (about 130 s from raw 2302, 155 s from empty). Both are safe, each interrupts once.
- **Not checked:** how pyobs and Influx decode `StatusWord` (does anything compare the whole word?), whether the sensor can exceed 10 V, whether 32767 is the EL3164 full scale, XAE compile, the semver bump for additive inputs (patch assumed).

## 6. Still to do
1. MONETcommonTests: `F_OilLevelValid_Tests` (0, 330 (sensor error), 599, 600, 883, 1330, 6887, 9416, 12000, 12001, 16383, 32767, defaults) and `F_OilSuctionLatch_Tests` (valid truth table equal to the old SR, invalid not latched with 182 %/375 % stays FALSE, invalid latched with the pump enabled stays TRUE, with the pump disabled FALSE, float switch always TRUE). After the HalfBROT release is installed.
2. MONETS `MAIN`: constants and the two inputs in the `HydraulicsControl(...)` call, `bOilLevelInvalid` on the slow-blink line, pin the new HalfBROT. Closes #32 after the on-site check. Comment on #32: 0..10 V, not 4..20 mA. MONETN the same with 600 / 16000.
3. IAG test plan: MS-02 step 4 with the safe procedure below, `srSuctionTrigger.Q1` to `bSuctionLatched`.

## 7. On-site check (MONET/S, someone at the cabinet)
Record at 10 Hz: `InOilLevel`, `fOilLevel`, `bOilLevelInvalid`, `statusWord`, `bSuctionLatched`, `ObOilSuctionEnable`, `ObOilPumpEnable`, `bSuctionRunning`, `bHydraulicsFailure`; MQTT `hydraulics/base/#`, the log.
1. Baseline: hydraulics off, `bOilLevelInvalid` FALSE, `OilLevel` every 5 s, bit 14 = 0.
2. Wire break: force `InOilLevel := 0`, and 330 (the sensor error level), each for 40 s: one WARN with the raw value, no `OilLevel` in MQTT while the other fields keep coming, `StatusWord` + 16384, no suction, no error. Release: HINT 'OK', `OilLevel` is back.
3. Boundaries: 599 invalid, 600 valid, 12001 invalid.
4. Over-range, safe: hydraulics off, pan below 15 %; additionally force `ObOilSuctionEnable := FALSE` (or open the suction pump breaker); force 16383, then 32767: invalid, `bSuctionLatched` stays FALSE. Positive control: 12000 (valid, about 130 %) sets `bSuctionLatched`, 1000 clears it. Remove the forces input first, then output.
5. Invalid while sucking: hydraulics on, suction latched, force 0: the suction keeps running; hydraulics off: it stops. Keep the dry run under about 1 min.
6. Invalid with suction off and the main pump running: force 0 for 60 s, no start. Do not wait for the float switch or the 140 s watchdog.
7. Influx: bit 14 in `StatusWord` during step 2; check the pyobs decoding.
8. MONET/N: measure the empty pan, float switch and start/stop raw values first (as MS-01/MS-01b), then repeat 1-3.
