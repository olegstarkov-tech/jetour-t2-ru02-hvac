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

### Ownership localization

- Exact DEX `class_def` scan across 88 APKs in `system/app`, `system/priv-app`, `product/app`, `product/priv-app` found zero definitions of `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Expanded exact scan across `system/framework`, `product/framework`, `system_ext/framework`, and `system_ext` app/priv-app locations checked 102 archives and also found zero exact owners.
- `DesaySVProjectService.apk` and `SVVDSCarStateService.apk` were explicitly decompiled and DISPROVEN as VehicleDevice owners.
- Full recursive `vendor.img` inventory is complete. Vendor contains only 2 APKs, 0 JARs, 1 ODEX, 1 VDEX, 0 OAT and 0 APEX, so the missing backend is not an unscanned ordinary vendor Java package.

### Native RU02 VehicleDevice path

RU02 vendor contains a dedicated native Desay VehicleDevice stack:

- `/etc/init/com.desaysv.vehicledevice@1.0-service.rc`;
- `/bin/hw/com.desaysv.vehicledevice@1.0-service`;
- `/lib64/com.desaysv.vehicledevice@1.0.so`;
- `/lib64/libdesaysv_vehicledevice.so`;
- related `libdesaysv_vehiclebus.so`, `libdesaysv_vehiclebus_backend_aidl.so`, `libdesaysv_vehiclebus_backend_aosp.so`, `libproperties_vehicle.so`, and Android Automotive Vehicle HAL libraries.

Static inspection now proves:

- native service executable links `libdesaysv_vehiclebus_backend_aidl.so`, `libdesaysv_vehiclebus_backend_aosp.so`, `libdesaysv_vehiclebus.so`, `libdesaysv_vehicledevice.so`, `com.desaysv.vehicledevice@1.0.so`, and `android.hardware.automotive.vehicle@2.0.so`;
- service imports `VehicleBusStub::get`, `set`, `bulkGet`, `publish`, `subscribe`, `unsubscribe` and `VehicleBusBundle` accessors;
- service binary contains string `vdev.service.VehicleDevice|vehiclebus`, strongly linking this native stack to the logical VDBus VehicleDevice role, though exact registration semantics still require targeted disassembly;
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectConfigs`, `parseProjectConfigs`, `parseProjectConfigsMap`, `setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, and `setVehiclePropertyConfigCallback`;
- `libdesaysv_vehicledevice.so` contains `vehicle.persist.project.ext.configs` through `ext.configs9`, `vehicle.persist.project.code`, `vehicle.persist.project.pn`, and `vehicle.persist.project.phonelink.configs`;
- it also contains `countryCode` and `ro.sys.ivi.eol.country.code` plus log string `VehicleHal::onVehiclePropertyConfigChange proKey :%s, proValue :%s`.

Therefore the exact RU02 native project-config layer has now been found. The leading static model is:

`Android Automotive Vehicle HAL -> libdesaysv_vehicledevice.so / VehicleHal -> property-config callback -> native VehicleDevice service -> VehicleBusStub publish -> framework VDBus VehicleDevice path -> CarConfigUtil/EolConfig`.

The callback-to-publish edge and assignment/construction of logical event `918905` are not yet PROVEN and are the immediate target.

Raw 32-bit searches for `918905` and `917510` in the three first native targets were negative. This is not evidence that the event path is absent; the ID may be assigned in glue/backend code or represented indirectly.

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
- `/vehicle` contains the Java VehicleDevice implementation.
- Vendor needs another broad APK/JAR search; the relevant backend is native.

## Current open question

Where exactly is the RU02 native callback-to-VehicleBus publication bridge for project config changes, and does it apply a country/project/market/telematics/capability condition that can explain missing Fragrance despite config50=1 being readable?

## Next step

Perform targeted symbol/disassembly tracing only around:

1. `VehicleHal::onVehiclePropertyConfigChange(...)`;
2. `VehicleHal::setProjectExtConfigs(...)` / `requestProjectConfigs()`;
3. the service-side implementation of `ISVPVehiclePropertyConfigCallback` or equivalent callback class;
4. call sites to `VehicleBusStub::publish()`;
5. if event construction is delegated, follow only into `libdesaysv_vehiclebus.so` / backend library actually referenced by those call sites.

Do not return to broad APK guessing, OAT/VDEX hunting, or unsigned HVAC patching.

When the vehicle becomes available again, pull/hash live HVAC/CarInfo APKs and reconcile artifact identity.
