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
- Older signed RU05 HVAC was installed and really executed from `/data/app`, but did not restore Fragrance.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`.
- Investigated Java/smali did not reveal `fragranceBtn.setVisibility(VISIBLE)`; DataBinding does attach a click listener.
- `a2(fragranceBtn,z)` controls enabled/clickable state, not visibility.
- A six-APK comparison set exists for RU05, RU06 and RU02-TEL: paired `SVHvac` + `SVVDSCarInfo` for each generation.
- Artifact hashes from the current comparison set:
  - `SVHvac_RU02_2026.apk` SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.
  - `SVHvac_RU05.apk` SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.
  - `SVHvac_RU06.apk` SHA-256 `36772386560e34eb09089e7036d27b4b18fe803471217c483db649900a108c06`.
  - `SVVDSCarInfo_RU02_TEL_2026.apk` SHA-256 `b2de95dd8c11ef9d32b79da3b43f89de2d0c148490b46140e44f3ec7e6aa83b3`.
  - `SVVDSCarInfo_RU05.apk` SHA-256 `be0a12b9995ef9dee8fcb4aff460687012141be62a68b90803033263687c8242`.
  - `SVVDSCarInfo_RU06.apk` SHA-256 `6b449fcd7e80677b09b97bd984fffffb403ab3d2c3cbf0174935ef8ea66ddbcb`.
- RU02 and RU06 HVAC are structurally very close by first-pass inventory: APK sizes `209865320` vs `209832552`, DEX sizes `6031108` vs `6031076`, identical manifest size `7188`, identical `bottom_layout.xml` size `12156`, and identical `dialog_fragrance_layout.xml` size `3180`.
- RU05 HVAC is a distinct generation: APK `93028104`, DEX `6275412`, manifest `6456`, `bottom_layout.xml` `7620`; Fragrance layouts/resources are still present.
- RU02 and RU06 `SVVDSCarInfo` are extremely close structurally: both APK size `2789840`, DEX `4210872`, `resources.arsc` `666316`, manifest `3436`; their whole-APK SHA-256 values differ.
- RU05 `SVVDSCarInfo` is a distinct generation: APK `3019216`, DEX `4926660`.
- A quick DEX string scan on RU05 HVAC and RU05 CarInfo exposes a much larger embedded VDBus/carconfig implementation symbol set, including `VehicleDevice`, `VehicleService`, `EolConfig`, `CarConfigUtil`, `ID_CAR_CONFIG_FRAGRANCE`, multiple `ID_AC_FRAGRANCE*` constants and Binder interface classes. The same scan on RU06/RU02 exposes mainly client references. This establishes a packaging/architecture generation change between RU05 and RU06; it does not by itself establish the Fragrance root cause or runtime class-loading precedence.

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
- Why does the old RU05 HVAC APK also fail on the current RU02 system base?
- Is the `fragrance_btn GONE` layout state a symptom of a firmware/resource packaging decision, or one part of a wider system-level gating mechanism?
- Are RU06 and RU02 `SVVDSCarInfo` `classes.dex` actually byte-identical despite different APK hashes?
- Which exact differences in HVAC and/or VDS CarInfo appeared at the RU05 -> RU06 generation boundary?
- Does runtime class loading on RU02 resolve VDBus/carconfig classes from the system framework/shared library instead of any copies bundled in the old RU05 APK?

## Constraints

- No Desay/platform private signing key is available, so modified system APK deployment is not currently a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
