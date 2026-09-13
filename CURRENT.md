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
  - `vdbus.jar` — 1395532 bytes; SHA-256 `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`;
  - `vdbus_extra.jar` — 111020 bytes; SHA-256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`;
  - `chery-platform-internal.jar` — 41604 bytes;
  - related candidates: `car-frameworks-service.jar`, `desaysv-car-frameworks-service-extension.jar`.
- `vdbus_extra.jar` contains `CarConfigUtil`, `EolConfig`, config constants and Fragrance HVAC IDs. `vdbus.jar` contains VDBus client/binder layer, `VDServiceDef`, `VDEventVehicleDevice` and `VDVDeviceConfigStore`.
- `CarConfigUtil.init()` subscribes to event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`) from `ServiceType.VEHICLE_DEVICE`, registers its notify listener, commits, and calls `EolConfig.loadConfig()`.
- On event `918905`, `CarConfigUtil` decodes `VDVDeviceConfigStore`, reads key/value, and for `vehicle.persist.project.ext.configs` calls `EolConfig.updateConfig(Utils.stringToByte(value), null, null, null, null)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; there is no extra Fragrance-specific gate there.
- `VDServiceDef` names the event source as system service package `com.desaysv.ivi.vds.vdev`, class `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.
- `VehicleService` is separately identified as HAL-facing `com.desaysv.ivi.vds.vehicle.service.VehicleService` under package `android.hardware.automotive.vehicle@2.0-service`.
- The framework-level config update chain is PROVEN as: VehicleDevice event 918905 -> `VDVDeviceConfigStore` key/value -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC predicate.

### VehicleDevice ownership search

- Two strongest APK candidates were extracted and decompiled:
  - `/system/priv-app/DesaySVProjectService/DesaySVProjectService.apk` — SHA-256 `2a172f9aa1a447df8ad32a733680843bfb5f131ca5c7db8dcbc507db49dd7282`, package `com.desaysv.ivi.vds.projection`.
  - `/product/app/SVVDSCarStateService/SVVDSCarStateService.apk` — SHA-256 `52b3d5f63922031be273746cc3ac553c19a2d49a9508028aa81305c9988c0e1d`, package `com.desaysv.ivi.vds.carstate`.
- Neither APK defines `com.desaysv.ivi.vds.vdev.service.VehicleDevice`; `SVVDSCarStateService` contains only client-side `VehicleDeviceManager`.
- An exact DEX `class_def` scan was then run across all APKs under RU02 `system/app`, `system/priv-app`, `product/app`, and `product/priv-app`.
- 88 APKs were checked.
- Result: zero APKs define descriptor `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Therefore VehicleDevice is not implemented in any of those 88 APKs; earlier raw string hits were references/shared service tables only.

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
- Any APK under the scanned RU02 `system/app`, `system/priv-app`, `product/app`, or `product/priv-app` defines VehicleDevice; exact class-table scan found none across 88 APKs.

## Current open question

Where is the actual RU02 implementation of `com.desaysv.ivi.vds.vdev.service.VehicleDevice`, and which backend condition inside/upstream of it suppresses or fails to publish the Fragrance capability/event path despite persistent config50 being readable as 1?

## Next step

Expand exact class-definition localization to RU02 `system_ext` plus framework archives (`system/framework`, `product/framework`, and `system_ext/framework`). Do not return to manual APK-name guessing. If the exact owner is found, decompile only that owner and trace event `918905` production and the origin/filtering of `vehicle.persist.project.ext.configs`.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile local artifact labels.

Do not install more packages, perform blind VDBus/property/config writes, or return to unsigned HVAC APK patching without a concrete mechanism.
