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

### Config / HVAC path

- Engineering Fragrance 0/1 changes `config1` byte12: `0x85` OFF -> `0xC5` ON; delta `0x40` = bit6.
- `/system/framework/vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` maps Fragrance as `(mCarConfig1[12] >> 6) & 1`.
- `mCarConfig1` source is `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed reading ext config with byte12=`C5`.
- Current HVAC `OfflineConfigManager.f()` is `getConfig(50)==1 && !isT1H_PHEV()`.
- Current HVAC contains Fragrance classes/resources/UI logic. RU06 vs RU02-labeled HVAC DEX/manifest/resource differences do not explain missing Fragrance.
- Older correctly signed RU05 HVAC executed on RU02 system base but still did not restore Fragrance; APK replacement alone is not a solution.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`; investigated binding/code has no proven visibility path. `a2(fragranceBtn,z)` controls enabled/clickable only.

### Framework backend path

- `vdbus.jar` SHA-256 `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`.
- `vdbus_extra.jar` SHA-256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`.
- `CarConfigUtil.init()` subscribes to VehicleDevice event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`).
- Event `918905` payload is decoded via `VDVDeviceConfigStore`; key `vehicle.persist.project.ext.configs` causes `EolConfig.updateConfig(...)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; there is no extra Fragrance-specific gate there.
- `VDServiceDef` identifies producer/service `com.desaysv.ivi.vds.vdev.service.VehicleDevice` in package `com.desaysv.ivi.vds.vdev`.
- `VehicleService` is separately HAL-facing: `com.desaysv.ivi.vds.vehicle.service.VehicleService` under `android.hardware.automotive.vehicle@2.0-service`.
- PROVEN framework chain: VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.

### Firmware images / ownership localization

- `system.img`: 967962624 bytes, SHA-256 `19ac46038cd8662891a8da6319844b31b0cadaf5e82e156108a91b474771265d`.
- `system_ext.img`: about 80.8 MB, SHA-256 `1b9902976b45277875d4a9b79d49fa4a26599dc9cdb682459c5fa1acfe47582b`.
- `vendor.img`: about 348.9 MB, SHA-256 `b1e7e189033a7d4347b2c730263d535955b783cb48a1a3226b6f0e8c4a9ef283`.
- Exact DEX `class_def` scan across 88 APKs in `system/app`, `system/priv-app`, `product/app`, `product/priv-app` found zero definitions of `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Expanded exact scan across `system/framework`, `product/framework`, `system_ext/framework`, and `system_ext` app/priv-app locations checked 102 archives and also found zero exact owners.
- `DesaySVProjectService.apk` and `SVVDSCarStateService.apk` were explicitly decompiled and DISPROVEN as VehicleDevice owners.
- Full recursive vendor export is now complete (`/home/olegs/RU02_vendor_dump`, about 385 MB).
- Vendor contains only two APKs (`TimeService.apk` and `overlay/framework-res-overlay.apk`), zero JARs, one ODEX, one VDEX, zero OAT and zero APEX. Therefore the missing backend is not hidden in an unscanned ordinary vendor APK/JAR population.

### Native vendor VehicleDevice stack

Full vendor inventory reveals a dedicated native Desay VehicleDevice stack:

- init rc: `/etc/init/com.desaysv.vehicledevice@1.0-service.rc`;
- executable: `/bin/hw/com.desaysv.vehicledevice@1.0-service`;
- HIDL/interface libraries: `/lib/com.desaysv.vehicledevice@1.0.so` and `/lib64/com.desaysv.vehicledevice@1.0.so`;
- implementation library: `/lib64/libdesaysv_vehicledevice.so`;
- related vehicle bus libraries include `libdesaysv_vehiclebus.so`, `libdesaysv_vehiclebus_backend_aidl.so`, `libdesaysv_vehiclebus_backend_aosp.so`, `libproperties_vehicle.so`, and `libutils_vehicle.so` in vendor lib/lib64.
- Vendor also contains HAL-facing `/bin/hw/android.hardware.automotive.vehicle@2.0-service.g6` and init rc `/etc/init/android.hardware.automotive.vehicle@2.0-service.g6.rc`.

This is PROVEN evidence that RU02 contains a native vendor VehicleDevice service stack. It is now the highest-priority backend target. The exact relationship between this native `com.desaysv.vehicledevice@1.0-service` and the framework logical service name `com.desaysv.ivi.vds.vdev.service.VehicleDevice` is not yet proven and must be traced from init/binary/library evidence rather than assumed.

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
- Any scanned ordinary APK/JAR in RU02 `system`, `product`, or `system_ext` defines VehicleDevice.
- `/vehicle` contains the Java VehicleDevice implementation; it contains only config files.
- Vendor needs another broad APK-name search; full recursive inventory shows only two APKs and no JARs.

## Current open question

How does the native RU02 vendor VehicleDevice stack (`com.desaysv.vehicledevice@1.0-service` / `libdesaysv_vehicledevice.so`) feed the framework VDBus VehicleDevice path/event `918905`, and is there a project/market/telematics/capability condition there that suppresses the Fragrance path despite persistent config50 being readable as 1?

## Next step

Inspect only the native VehicleDevice launch/implementation chain read-only:

1. dump `com.desaysv.vehicledevice@1.0-service.rc`;
2. fingerprint and inspect dynamic dependencies/strings/symbols for `/bin/hw/com.desaysv.vehicledevice@1.0-service` and `/lib64/libdesaysv_vehicledevice.so`;
3. search those exact targets for `918905`, `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`, `vehicle.persist.project.ext.configs`, `Fragrance`, config/property/store terms, project/market/country/region/TBox/telematics/capability terms, and references to vehiclebus/Vehicle HAL;
4. inspect `init.desaysv.vehicle.rc` and `android.hardware.automotive.vehicle@2.0-service.g6.rc` only where references require it.

Do not return to broad APK guessing or unsigned HVAC patching.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile local artifact labels.
