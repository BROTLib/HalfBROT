# Clamp the published oil level to 0..100 % (#123)

**Status: implemented on `develop`, not compiled, not run on a PLC.**

Issue: HalfBROT#123. Seen on MONET/S (HalfBROT 0.7.3, 2026-10-26, telescope parked): `fOilLevel` between -4.0 and +3.6 % at raw 611 to 1157.

## Problem

`FB_HydraulicsControl` scaled the raw level with `F_YREAL(..., 0, 100, cut := FALSE)`, which extrapolates linearly. A working sensor
below `nMinPanLevel` (the oversized pan on MONET/S is never drained completely) gave a negative percentage, one above
`nMaxPanLevel` more than 100 %.

## Change

`FB_HydraulicsControl.TcPOU:138`: `cut := TRUE`, so the published level is clamped to 0..100 %. The raw validity check
(`F_OilLevelValid` with `nMinValidPanLevel`/`nMaxValidPanLevel`) and the suction logic use the raw value and are unchanged.

Clamping itself is covered by `FB_YREAL_Tests` in BROTLib; no new test.

Effects: the 5 % telemetry trigger sees flat values beyond the ends; the failure message and the Hydraulics visualization show the
clamped level.

Related: BROTLib/MONETS#58, #36.
