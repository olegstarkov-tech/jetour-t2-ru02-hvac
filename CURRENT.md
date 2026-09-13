# RU02-HVAC CURRENT

Status: active
Canonical branch: `main`
Recovery marker: `RU02-HVAC-CONTINUE`

## Bench

- Jetour T2 / T1J
- Official Russian dealer vehicle
- Desay SV 8155
- Dealer RU firmware 00.00.02
- Telematics present

This bench is distinct from Desay 00.00.08 / D08 work.

## Current mission

Determine why OEM Fragrance / Aromatization does not appear/work even when vehicle configuration enables it, and identify the actual RU02 system mechanism controlling availability.

## Latest PROVEN state

### Config path

- Engineering Fragrance 0/1 changes `config1` byte12: `0x85` OFF -> `0xC5` ON; delta `0x40` = bit6.
- `/system/framework/vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` maps Fragrance as `(mCarConfig1[12] >> 6) & 1`.
- `mCarConfig1` source is `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed reading ext config with byte12=`C5`.
- Current HVAC `OfflineConfigManager.f()` is `getConfig(50)==1 && !isT1H_PHEV()`.
- Vehicle is T1J; T1H is not the active branch without new evidence.

### HVAC application path

- Current HVAC contains Fragrance classes/resources/UI logic.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`; investigated binding/code does not show a proven path setting it VISIBLE.
- `a2(fragranceBtn,z)` controls enabled/clickable state only.
- Older correctly signed RU05 HVAC was installed and executed from `/data/app` on the RU02 system base, but Fragrance still did not appear. APK replacement alone is not a solution.
- RU06 vs RU02-labeled HVAC is extremely close: 2948 byte-identical ZIP entries; Fragrance layouts are byte-identical.
- Focused JADX plus direct smali diff proves the only RU06 -> RU02 DEX behavior delta is `K1(boolean)` adding `OfflineConfigManager.h()` / config91 (`isIonExist`) before `ionAnimation`.
- `OfflineConfigManager.h()` is ionizer/config91; `OfflineConfigManager.f()` is Fragrance/config50. The helper source is byte-identical between RU06 and RU02-labeled HVAC.
- Decoded RU06 vs RU02-labeled manifest is semantically identical.
- Decoded resource differences are localization/string-only; no layout/bool/integer/style/id/array/drawable/alias/visibility resource differs. Russian Fragrance strings are unchanged.
- Therefore the observed RU06 -> RU02 HVAC APK delta contains no mechanism explaining missing Fragrance.

### CarInfo / artifact identity

- RU06 vs RU02-labeled CarInfo generated Java differs only in `BuildConfig.VERSION_NAME`.
- Earlier live CarInfo `versionName` matches the local artifact labeled RU06, not local `RU02_TEL_2026`; live APK hash identity remains unresolved until the vehicle is available.
- RU05 -> RU06 is a real packaging/architecture generation boundary: RU05 HVAC/CarInfo expose a much larger embedded VDBus/carconfig implementation symbol set; newer artifacts mainly expose client references.

### RU02 firmware/framework

- RU02 OTA payload contains 25 partitions including Android `system` (923.1 MB), `system_ext` (80.8 MB), `product` (5.6 GB), `vendor` (348.9 MB) and separate `system_qnx` (3.0 GB).
- Android `system.img` was extracted read-only: 967962624 bytes; SHA-256 `19ac46038cd8662891a8da6319844b31b0cadaf5e82e156108a91b474771265d`; ext2 filesystem.
- Correct framework path inside the image is `/system/framework`.
- Exact RU02 framework targets located there:
  - `vdbus.jar` — 1395532 bytes;
  - `vdbus_extra.jar` — 111020 bytes;
  - `chery-platform-internal.jar` — 41604 bytes;
  - related candidates: `car-frameworks-service.jar`, `desaysv-car-frameworks-service-extension.jar`.

## DISPROVEN / closed unless new evidence

- Engineering writes the wrong Fragrance bit.
- New HVAC uses another config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC alone solves the issue.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` controls visibility.
- RU06 -> RU02 HVAC DEX/manifest/resource differences contain the missing-Fragrance gate.
- `OfflineConfigManager.h()` is a Fragrance predicate; it is ionizer/config91.

## Current open question

Which RU02 system/framework/backend condition suppresses or fails to publish the Fragrance capability/event path despite persistent config50 being readable as 1 and the HVAC application containing the expected Fragrance predicate/resources?

## Next step

Extract exact RU02 `/system/framework/vdbus_extra.jar` and `vdbus.jar`, record hashes/archive contents, decompile read-only, and map Fragrance/config50 through `EolConfig`, `CarConfigUtil`, VDBus and any `VehicleDevice`/`VehicleService` references. Expand to the other framework jars only if the reference graph requires it.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile local artifact labels.

Do not install more packages, perform blind VDBus/property/config writes, or return to unsigned HVAC APK patching without a concrete mechanism.
