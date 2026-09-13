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
- A six-APK comparison set exists for RU05, RU06 and RU02-TEL labels: paired `SVHvac` + `SVVDSCarInfo` for each generation label.
- Artifact hashes from the current comparison set:
  - `SVHvac_RU02_2026.apk` SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.
  - `SVHvac_RU05.apk` SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.
  - `SVHvac_RU06.apk` SHA-256 `36772386560e34eb09089e7036d27b4b18fe803471217c483db649900a108c06`.
  - `SVVDSCarInfo_RU02_TEL_2026.apk` SHA-256 `b2de95dd8c11ef9d32b79da3b43f89de2d0c148490b46140e44f3ec7e6aa83b3`.
  - `SVVDSCarInfo_RU05.apk` SHA-256 `be0a12b9995ef9dee8fcb4aff460687012141be62a68b90803033263687c8242`.
  - `SVVDSCarInfo_RU06.apk` SHA-256 `6b449fcd7e80677b09b97bd984fffffb403ab3d2c3cbf0174935ef8ea66ddbcb`.
- RU05 HVAC is a distinct generation: APK `93028104`, DEX `6275412`, manifest `6456`, `bottom_layout.xml` `7620`; Fragrance layouts/resources are still present.
- RU06 and RU02-labeled HVAC are extremely close at ZIP-entry level: 2948 identical entries, only 6 changed, no added/removed; Fragrance XML layouts are byte-identical. Changed entries are DEX, manifest, resources table and signature metadata.
- HVAC RU06 `classes.dex` is `6031076` bytes; RU02-labeled is `6031108` bytes.
- Focused JADX source-tree diff of RU06 vs RU02-labeled HVAC finds exactly one differing generated Java file: `com/desaysv/svhvac/view/b.java`.
- The only generated-Java delta is `K1(boolean)`: RU02-labeled HVAC adds guard `com.desaysv.svhvac.h.a.a().h()` around the existing `ionAnimation` update block. The underlying operations remain `setVisibilityAndRunOn(...)` and `setDrawables(R.array.blow_mode_ion_array)`.
- Both HVAC JADX runs report 22 errors while processing the same 1449-class workload; the captured console logs do not identify each error, so equality of the error sets is not yet proven.
- RU05 `SVVDSCarInfo` is a distinct generation: APK `3019216`, DEX `4926660`.
- RU06 and RU02-labeled `SVVDSCarInfo` are extremely close at ZIP-entry level: 593 identical entries, only 5 changed, no added/removed; `resources.arsc` is byte-identical; changes are DEX, manifest and signature metadata.
- RU06 and RU02-labeled CarInfo `classes.dex` are equal-size (`4210872`) but have different SHA-256.
- Focused JADX source-tree diff of RU06 vs RU02-labeled CarInfo found exactly one differing generated Java file: `com/desaysv/ivi/vds/carinfo/BuildConfig.java`.
- The only decompiled Java delta is `VERSION_NAME`: RU06 = `Chery-8155_11_e5fc2d4_2025-09-26_2509261457_R`; RU02-labeled = `Chery-8155_11_8f9dc0d_2026-04-10_2604101647_R`. Other shown BuildConfig fields are unchanged.
- The collected JADX WARN/ERROR reports for those two CarInfo decompiles normalize to the exact same 662-line multiset. The same AndroidX `DiffUtil.java:107` RegionMakerVisitor error appears in both.
- Earlier live `dumpsys package com.desaysv.ivi.vds.carinfo` reported versionName `Chery-8155_11_e5fc2d4_2025-09-26_2509261457_R`, matching the local artifact labeled RU06 rather than the local artifact labeled RU02-TEL-2026. Live APK hash identity is not yet established.
- RU05 -> RU06 CarInfo DEX shrinks materially (`4926660` -> `4210872`), confirming the generation boundary is concentrated in code rather than resources.
- A quick DEX string scan on RU05 HVAC and RU05 CarInfo exposes a much larger embedded VDBus/carconfig implementation symbol set, including `VehicleDevice`, `VehicleService`, `EolConfig`, `CarConfigUtil`, `ID_CAR_CONFIG_FRAGRANCE`, multiple `ID_AC_FRAGRANCE*` constants and Binder interface classes. RU06/RU02-generation artifacts expose mainly client references. This establishes a packaging/architecture generation change between RU05 and RU06; it does not by itself establish the Fragrance root cause or runtime class-loading precedence.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from the current HVAC APK.
- Installing the old HVAC APK alone solves the problem.
- T1H is the active vehicle branch for this T1J bench.
- `a2(fragranceBtn,z)` is the visibility gate.
- RU06 and RU02-labeled `SVVDSCarInfo` decompiled application logic contains broad functional Java differences; current source diff shows only BuildConfig version metadata.
- RU06 and RU02-labeled HVAC contain a broad Java-code rewrite; current generated-source diff shows one method delta in one class.

## OPEN

- Which RU02 system/backend condition prevents Fragrance from becoming available despite config50=1?
- Why does the old RU05 HVAC APK also fail on the current RU02 system base?
- Which local CarInfo/HVAC artifacts are byte-exact with the current live canonical vehicle?
- What exact condition does `com.desaysv.svhvac.h.a.a().h()` represent, and is the new `K1(boolean)` guard relevant only to ion/air-quality behavior or to a broader capability gate?
- Is the `fragrance_btn GONE` layout state a symptom of a firmware/resource packaging decision, or one part of a wider system-level gating mechanism?
- What are the decoded manifest/resources-table differences between RU06 and RU02-labeled HVAC?
- Does runtime class loading on RU02 resolve VDBus/carconfig classes from the system framework/shared library instead of any copies bundled in the old RU05 APK?

## Constraints

- No Desay/platform private signing key is available, so modified system APK deployment is not currently a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
