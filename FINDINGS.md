# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`.
- Difference is `0x40` = bit6.
- `vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` returns `(mCarConfig1[12] >> 6) & 1`.
- `vehicle.persist.project.ext.configs` is the source for `mCarConfig1`.
- Current HVAC has been observed starting with ext config containing `C5` at byte12.
- Current HVAC contains Fragrance code/resources.
- Current HVAC gate is `getConfig(50)==1 && !isT1H_PHEV()`.
- Older signed HVAC was installed and really executed from `/data/app`, but did not restore Fragrance.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`.
- Investigated Java/smali did not reveal `fragranceBtn.setVisibility(VISIBLE)`; DataBinding does attach a click listener.
- `a2(fragranceBtn,z)` controls enabled/clickable state, not visibility.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from the current HVAC APK.
- Installing the old HVAC APK alone solves the problem.
- T1H is the active vehicle branch for this T1J bench.
- `a2(fragranceBtn,z)` is the visibility gate.

## OPEN

- Which RU02 system/backend condition prevents Fragrance from becoming available despite config50=1?
- Why does the old HVAC APK also fail on the current RU02 system base?
- Is the `fragrance_btn GONE` layout state a symptom of a firmware/resource packaging decision, or one part of a wider system-level gating mechanism?

## Constraints

- No Desay/platform private signing key is available, so modified system APK deployment is not currently a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
