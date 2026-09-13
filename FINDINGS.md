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
- RU05 HVAC is a distinct generation: APK `93028104`, DEX `6275412`, manifest `6456`, `bottom_layout.xml` `7620`; Fragrance layouts/resources are still present.
- RU06 and RU02 HVAC are extremely close at ZIP-entry level:
  - `2948` identical entries;
  - only `6` changed entries;
  - no added/removed entries;
  - changes are `classes.dex`, `AndroidManifest.xml`, `resources.arsc`, `META-INF/CERT.RSA`, `META-INF/CERT.SF`, `META-INF/MANIFEST.MF`;
  - `bottom_layout.xml`, `dialog_fragrance_layout.xml`, and `fragrance_type_item.xml` are byte-identical.
- HVAC RU06 `classes.dex` is `6031076` bytes; RU02 is `6031108` bytes. The code delta is small in size.
- RU05 `SVVDSCarInfo` is a distinct generation: APK `3019216`, DEX `4926660`.
- RU06 and RU02 `SVVDSCarInfo` are extremely close at ZIP-entry level:
  - `593` identical entries;
  - only `5` changed entries;
  - no added/removed entries;
  - `resources.arsc` is byte-identical;
  - changes are `classes.dex`, `AndroidManifest.xml`, and the three v1 signature files.
- RU06 and RU02 CarInfo `classes.dex` are the same size (`4210872`) but have different SHA-256 (`d86f1bcee2b90531ebf2ca7463cbd1bcea11444d18580b25cbf13aeca48f41e6` vs `a3609586a6c601b4bdefb9c106ff8900c17c3ec79f4818f9ffce2b853666969b`); therefore the DEX code is NOT byte-identical.
- RU05 -> RU06 CarInfo also changes only five ZIP entries, but its DEX shrinks materially (`4926660` -> `4210872`), confirming the generation boundary is concentrated in code rather than resources.
- A quick DEX string scan on RU05 HVAC and RU05 CarInfo exposes a much larger embedded VDBus/carconfig implementation symbol set, including `VehicleDevice`, `VehicleService`, `EolConfig`, `CarConfigUtil`, `ID_CAR_CONFIG_FRAGRANCE`, multiple `ID_AC_FRAGRANCE*` constants and Binder interface classes. The same scan on RU06/RU02 exposes mainly client references. This establishes a packaging/architecture generation change between RU05 and RU06; it does not by itself establish the Fragrance root cause or runtime class-loading precedence.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from the current HVAC APK.
- Installing the old HVAC APK alone solves the problem.
- T1H is the active vehicle branch for this T1J bench.
- `a2(fragranceBtn,z)` is the visibility gate.
- RU06 and RU02 `SVVDSCarInfo` `classes.dex` are byte-identical. They are equal-size but have different hashes.

## OPEN

- Which RU02 system/backend condition prevents Fragrance from becoming available despite config50=1?
- Why does the old RU05 HVAC APK also fail on the current RU02 system base?
- Is the `fragrance_btn GONE` layout state a symptom of a firmware/resource packaging decision, or one part of a wider system-level gating mechanism?
- Which exact Java/smali classes differ between RU06 and RU02 HVAC and CarInfo?
- Which exact differences appeared at the RU05 -> RU06 generation boundary?
- Does runtime class loading on RU02 resolve VDBus/carconfig classes from the system framework/shared library instead of any copies bundled in the old RU05 APK?

## Constraints

- No Desay/platform private signing key is available, so modified system APK deployment is not currently a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
