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
- `system_ext.img` was extracted read-only: about 80.8 MB; SHA-256 `1b9902976b45277875d4a9b79d49fa4a26599dc9cdb682459c5fa1acfe47582b`; ext2 filesystem.
- Exact RU02 framework targets in `/system/framework` include:
  - `vdbus.jar` — SHA-256 `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`;
  - `vdbus_extra.jar` — SHA-256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`.
- `vdbus_extra.jar` contains `CarConfigUtil`, `EolConfig`, config constants and Fragrance HVAC IDs. `vdbus.jar` contains VDBus client/binder layer, `VDServiceDef`, `VDEventVehicleDevice` and `VDVDeviceConfigStore`.
- `CarConfigUtil.init()` subscribes to event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`) from `ServiceType.VEHICLE_DEVICE`, registers its notify listener, commits, and calls `EolConfig.loadConfig()`.
- On event `918905`, `CarConfigUtil` decodes `VDVDeviceConfigStore`, reads key/value, and for `vehicle.persist.project.ext.configs` calls `EolConfig.updateConfig(Utils.stringToByte(value), null, null, null, null)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; there is no extra Fragrance-specific gate there.
- `VDServiceDef` names the event source as system service package `com.desaysv.ivi.vds.vdev`, class `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.
- `VehicleService` is separately identified as HAL-facing `com.desaysv.ivi.vds.vehicle.service.VehicleService` under package `android.hardware.automotive.vehicle@2.0-service`.
- The framework-level config update chain is PROVEN as: VehicleDevice event 918905 -> `VDVDeviceConfigStore` key/value -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC predicate.

### VehicleDevice ownership search

- Two strongest APK candidates were extracted/decompiled and DISPROVEN as owners: `DesaySVProjectService.apk` and `SVVDSCarStateService.apk`.
- Exact DEX `class_def` scan across RU02 `system/app`, `system/priv-app`, `product/app`, and `product/priv-app` checked 88 APKs and found zero owners for descriptor `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Search was then expanded to Java archives across `system/framework`, `product/framework`, `system_ext/framework`, and `system_ext` app/priv-app locations using the same exact DEX class-table test.
- Total archives checked in that expanded pass: 102.
- Result: zero exact owners; no scanned APK/JAR in `system`, `product`, or `system_ext` defines `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Therefore raw string hits seen earlier are references/shared service tables only, and ordinary Java containers in these three partitions are now excluded as the VehicleDevice implementation location.
- `vendor` has not yet been scanned and remains the next untested Android partition before considering preoptimized/oat/apex/native packaging.

## DISPROVEN / closed unless new evidence

- Engineering writes the wrong Fragrance bit.
- New HVAC uses another config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC alone solves the issue.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` controls visibility.
- RU06 -> RU02 HVAC DEX/manifest/resource differences contain the missing-Fragrance gate.
- `OfflineConfigManager.h()` is a Fragrance predicate; it is ionizer/config91.
- `CarConfigUtil` contains a separate Fragrance-specific suppression after `getConfig(50)`.
- Raw DEX string presence identifies the VehicleDevice implementation APK.
- `DesaySVProjectService.apk` or `SVVDSCarStateService.apk` implements VehicleDevice.
- Any scanned ordinary APK/JAR in RU02 `system`, `product`, or `system_ext` defines VehicleDevice; exact class-table scanning found none.

## Current open question

Where is the actual RU02 implementation of `com.desaysv.ivi.vds.vdev.service.VehicleDevice`, and which backend condition inside/upstream of it suppresses or fails to publish the Fragrance capability/event path despite persistent config50 being readable as 1?

## Next step

Extract RU02 `vendor.img` read-only and run the same exact DEX class-definition search across vendor APK/JAR locations (`/app`, `/priv-app`, `/framework`, and equivalent `/vendor/...` layout if present). Only if vendor also has zero exact owner should investigation move to preoptimized OAT/VDEX/APEX/native/system-service packaging.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile local artifact labels.

Do not install more packages, perform blind VDBus/property/config writes, or return to unsigned HVAC APK patching without a concrete mechanism.
