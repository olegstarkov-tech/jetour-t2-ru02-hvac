# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- `vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` returns `(mCarConfig1[12] >> 6) & 1`.
- `vehicle.persist.project.ext.configs` is the source for `mCarConfig1`.
- Current HVAC has been observed starting with ext config containing `C5` at byte12.
- Current HVAC contains Fragrance code/resources.
- Current HVAC Fragrance gate is `getConfig(50)==1 && !isT1H_PHEV()`.
- Older signed RU05 HVAC really executed on the current RU02 system but did not restore Fragrance; replacing HVAC APK alone is not a solution.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`; investigated binding/code did not reveal a visibility path, and `a2(fragranceBtn,z)` only controls enabled/clickable state.
- RU06 vs RU02-labeled HVAC is extremely close: 2948 identical ZIP entries, only DEX/manifest/resources/signature metadata differ; Fragrance layouts are byte-identical.
- Focused Java and direct smali diff prove the only RU06 -> RU02 HVAC behavior change is `K1(boolean)` adding `OfflineConfigManager.h()` / config91 (`isIonExist`) before `ionAnimation`. It is not the Fragrance/config50 predicate.
- RU06 and RU02-labeled `OfflineConfigManager.java` are byte-identical; `h()` is config91/ionizer and `f()` is config50/Fragrance.
- Decoded RU06 vs RU02-labeled HVAC manifest is semantically identical.
- Decoded resource changes are localization-only: 13 changed common `strings.xml` plus one RU02-only Malay locale file; no layout/bool/integer/style/id/array/drawable/alias/visibility change exists.
- Therefore the observed RU06 -> RU02 HVAC APK delta provides no mechanism explaining hidden Fragrance.
- RU06 vs RU02-labeled CarInfo generated Java differs only in `BuildConfig.VERSION_NAME`; normalized JADX warning/error reports are equivalent.
- Earlier live CarInfo `versionName` matches the local artifact labeled RU06, so live APK hash identity remains to be reconciled when the vehicle is available.
- RU05 HVAC/CarInfo expose a larger embedded VDBus/carconfig implementation set than RU06/RU02-generation artifacts, proving a packaging/architecture generation boundary.
- RU02 OTA payload contains 25 partitions including Android `system` (923.1 MB), `system_ext` (80.8 MB), `product` (5.6 GB), `vendor` (348.9 MB) and separate `system_qnx` (3.0 GB).
- RU02 Android `system.img` has been extracted read-only: 967962624 bytes, SHA-256 `19ac46038cd8662891a8da6319844b31b0cadaf5e82e156108a91b474771265d`, ext2 filesystem.
- The correct framework directory inside this image is `/system/framework`.
- `/system/framework` contains `vdbus.jar` (1395532 bytes), `vdbus_extra.jar` (111020 bytes), and `chery-platform-internal.jar` (41604 bytes), all present in the 2026-04-10 RU02 image; related candidates include `car-frameworks-service.jar` and `desaysv-car-frameworks-service-extension.jar`.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC alone solves the problem.
- T1H is the active vehicle branch for this T1J bench.
- `a2(fragranceBtn,z)` is the visibility gate.
- `OfflineConfigManager.h()` is a Fragrance predicate; it is ionizer/config91.
- RU06 -> RU02 HVAC DEX/manifest/resource deltas contain the missing-Fragrance gate.

## OPEN

- Which RU02 system/backend condition prevents Fragrance from becoming available despite config50=1?
- Why does old RU05 HVAC also fail on the RU02 system base?
- Which local CarInfo/HVAC artifacts are byte-exact with the live canonical vehicle?
- Does RU02 runtime resolve VDBus/carconfig classes from system framework/shared libraries rather than bundled copies in old RU05 HVAC?
- Which system/framework/service component suppresses or fails to publish the Fragrance capability/event path despite persistent config50 being readable as 1?

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
