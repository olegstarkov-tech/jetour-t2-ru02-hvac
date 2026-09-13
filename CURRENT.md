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
- Both VDBus JARs decompile cleanly with JADX (`vdbus_extra`: 9 classes/work units; `vdbus`: 324; no reported decompilation errors in captured logs).
- `vdbus_extra.jar` contains `CarConfigUtil`, `EolConfig`, config constants and Fragrance HVAC IDs. `vdbus.jar` contains the VDBus client/binder layer, `VDServiceDef`, `VDEventVehicleDevice` and `VDVDeviceConfigStore`.
- `CarConfigUtil.init()` initializes VDBus and, when `ServiceType.VEHICLE_DEVICE` connects, subscribes to event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`), registers its notify listener, commits the subscription, then calls `EolConfig.loadConfig()`.
- The same subscription/load path is also used when VehicleDevice is already connected.
- On event `918905`, `CarConfigUtil` decodes the payload with `VDVDeviceConfigStore.getValue(vDEvent)`, obtains a key/value pair, and for `vehicle.persist.project.ext.configs` calls `EolConfig.updateConfig(Utils.stringToByte(value), null, null, null, null)`. Config2..5 are handled analogously.
- `CarConfigUtil.getConfig(int)` directly returns `EolConfig.getJetourEolConfig(int)`; there is no additional Fragrance-specific gate in `CarConfigUtil` itself.
- `VDServiceDef` identifies the event source as system service package `com.desaysv.ivi.vds.vdev`, class `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.
- `VDServiceDef` separately identifies the vehicle HAL-facing service as `com.desaysv.ivi.vds.vehicle.service.VehicleService` under package `android.hardware.automotive.vehicle@2.0-service`.
- `VDEventVehicleDevice` defines `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE = 918905` and `PROJECT_RESERVE_CONFIGS = 917510`.
- Therefore the RU02 persistent-config update path is mapped at framework level as: VehicleDevice event 918905 -> `VDVDeviceConfigStore` key/value -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC predicate.

### VehicleDevice package localization

- Two strongest APK candidates were extracted read-only by exact path:
  - `/system/priv-app/DesaySVProjectService/DesaySVProjectService.apk` — 4393739 bytes, SHA-256 `2a172f9aa1a447df8ad32a733680843bfb5f131ca5c7db8dcbc507db49dd7282`, package `com.desaysv.ivi.vds.projection`, versionCode 30, versionName `11`.
  - `/product/app/SVVDSCarStateService/SVVDSCarStateService.apk` — 1877064 bytes, SHA-256 `52b3d5f63922031be273746cc3ac553c19a2d49a9508028aa81305c9988c0e1d`, package `com.desaysv.ivi.vds.carstate`, versionCode 1, versionName `carstate_20230416.1650`.
- `DesaySVProjectService.apk` manifest declares `com.desaysv.ivi.vds.projection.service.ProjectionService` and `com.desaysv.ivi.vds.projection.DesaySVProjectManagerService`; it does NOT declare `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.
- `SVVDSCarStateService.apk` manifest declares `com.desaysv.ivi.vds.carstate.service.CarStateService`; it does NOT declare `VehicleDevice`.
- Both APK DEXes contain strings for package `com.desaysv.ivi.vds.vdev` and class `com.desaysv.ivi.vds.vdev.service.VehicleDevice`, but string presence alone proves only a reference, not that the implementation class is defined in the APK.
- `DesaySVProjectService.apk` additionally contains VDBus symbol strings such as `VDEventVehicleDevice`, `VDVDeviceConfigStore`, and `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`; `SVVDSCarStateService.apk` contains a `VehicleDeviceManager` client class. Exact class ownership still needs source-tree verification.

## DISPROVEN / closed unless new evidence

- Engineering writes the wrong Fragrance bit.
- New HVAC uses another config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC alone solves the issue.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` controls visibility.
- RU06 -> RU02 HVAC DEX/manifest/resource differences contain the missing-Fragrance gate.
- `OfflineConfigManager.h()` is a Fragrance predicate; it is ionizer/config91.
- `CarConfigUtil` contains a separate Fragrance-specific suppression after `getConfig(50)`; current RU02 code shows `getConfig()` delegates directly to `EolConfig`.
- DEX string presence of `com.desaysv.ivi.vds.vdev.service.VehicleDevice` is sufficient to identify the APK that implements VehicleDevice. Exact class definition must be verified.

## Current open question

Which RU02 backend condition inside or upstream of `VehicleDevice` / `VehicleService` suppresses or fails to publish the Fragrance capability/event path despite persistent config50 being readable as 1 and HVAC containing the expected Fragrance logic?

## Next step

Decompile the two extracted candidate APKs and determine whether either actually defines `sources/com/desaysv/ivi/vds/vdev/service/VehicleDevice.java`. If one does, trace event `918905` production there. If neither does, broaden package localization to the remaining RU02 APKs using exact class-definition search rather than raw string hits.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile local artifact labels.

Do not install more packages, perform blind VDBus/property/config writes, or return to unsigned HVAC APK patching without a concrete mechanism.
