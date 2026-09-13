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

Static inspection proves:

- native service executable links `libdesaysv_vehiclebus_backend_aidl.so`, `libdesaysv_vehiclebus_backend_aosp.so`, `libdesaysv_vehiclebus.so`, `libdesaysv_vehicledevice.so`, `com.desaysv.vehicledevice@1.0.so`, and `android.hardware.automotive.vehicle@2.0.so`;
- service imports `VehicleBusStub::get`, `set`, `bulkGet`, `publish`, `subscribe`, `unsubscribe` and `VehicleBusBundle` accessors;
- service binary contains `vdev.service.VehicleDevice|vehiclebus` and log string `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s`;
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, and `setVehiclePropertyConfigCallback`;
- `VehicleHal::setProjectExtConfigs(key,value)` compares against the stored value, ignores empty/unchanged updates, writes changed project config to `VehicleConfigStore`, updates EOL caches for ext-config keys, then invokes the registered property-config callback with the same key/value;
- `VehicleHal::onVehiclePropertyConfigChange(key,value)` only logs and forwards the same key/value through the registered callback; no country/project/market/telematics condition was observed in that forwarding function;
- `libdesaysv_vehicledevice.so` contains `vehicle.persist.project.ext.configs` through `ext.configs9`, project code/PN/phonelink keys, `countryCode`, and `ro.sys.ivi.eol.country.code`.

### Event 918905 native ownership and bundle construction

Targeted disassembly of `/vendor/bin/hw/com.desaysv.vehicledevice@1.0-service` proves the service binary itself materializes framework event ID `918905` (`0x000E0579`):

- at `0x7ba8`: `mov w8, #0x579` followed at `0x7bb0` by `movk w8, #0xe, lsl #16`;
- at `0x7ca4`: `mov w8, #0x579` followed at `0x7cac` by `movk w8, #0xe, lsl #16`.

In each of those blocks the event-ID construction is immediately followed by two `VehicleBusBundle::putString(...)` calls:

- first block: `0x7bc4` and `0x7bd8`;
- second block: `0x7cc0` and `0x7cd4`.

This is PROVEN evidence that the service-side 918905 path constructs a two-string bundle. Given the upstream callback signature and framework `VDVDeviceConfigStore` decoding, the leading interpretation is that these are the property key/value pair, but the exact string-key names/argument identities still need decoding before marking that part PROVEN.

The service imports `VehicleBusStub::publish(VehicleBusEvent const&)`. `readelf -rW` shows `publish` as an `R_AARCH64_ABS64` relocation at address `0x11088`, unlike `VehicleBusStub::set`, which has an ordinary `R_AARCH64_JUMP_SLOT`/PLT entry. Therefore no direct `bl publish@plt` is expected; the publish edge is likely indirect via a stored function pointer/table slot. That indirect call edge is not yet PROVEN.

The event-ID sites remain in the same local code region as xrefs to `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s`, so event 918905 is definitively owned by the native VehicleDevice service glue.

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
- Event `918905` exists only in the Java framework; the native service explicitly constructs `0x000E0579`.
- A direct PLT call to `VehicleBusStub::publish()` should exist in the 918905 block; relocation evidence shows `publish` is referenced through an ABS64 data relocation instead.

## Current open question

What exact service-side function owns the 918905/two-`putString` blocks, what are the two bundle field names/arguments, and how does that block invoke the `VehicleBusStub::publish` pointer relocated at `0x11088`? Once that exact publication bridge is closed, inspect only branches in that function/path for conditions capable of suppressing or transforming `vehicle.persist.project.ext.configs`.

## Next step

Resolve the narrow 918905 publication bridge only:

1. disassemble the exact `0x7b80-0x7d10` blocks with raw register flow;
2. resolve rodata/string addresses used by the two `VehicleBusBundle::putString` calls in each 918905 block;
3. determine which arguments are property key and property value;
4. identify the ELF section and all code xrefs to the `VehicleBusStub::publish` relocation slot at `0x11088`;
5. prove the indirect call edge from the 918905 callback block to that slot or identify the exact wrapper/table that performs it.

Do not return to broad APK guessing, OAT/VDEX hunting, or unsigned HVAC patching.

When the vehicle becomes available again, pull/hash live HVAC/CarInfo APKs and reconcile artifact identity.
