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

- `CarConfigUtil.init()` subscribes to VehicleDevice event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`).
- Event `918905` payload is decoded via `VDVDeviceConfigStore`; key `vehicle.persist.project.ext.configs` causes `EolConfig.updateConfig(...)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; there is no extra Fragrance-specific gate there.
- `VDServiceDef` identifies producer/service `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.
- PROVEN framework chain: VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.

### Ownership localization

- Exact DEX `class_def` search found zero Java owners across scanned `system`, `product`, `system_ext`, and vendor APK/JAR populations.
- `DesaySVProjectService.apk` and `SVVDSCarStateService.apk` were explicitly DISPROVEN as VehicleDevice owners.
- Full recursive vendor inventory shows only 2 APKs, 0 JARs, 1 ODEX, 1 VDEX, 0 OAT and 0 APEX; the relevant backend is native.

### Native RU02 VehicleDevice path

RU02 vendor contains a dedicated native stack:

- `/etc/init/com.desaysv.vehicledevice@1.0-service.rc`;
- `/bin/hw/com.desaysv.vehicledevice@1.0-service`;
- `/lib64/com.desaysv.vehicledevice@1.0.so`;
- `/lib64/libdesaysv_vehicledevice.so`;
- related `libdesaysv_vehiclebus.so`, `libdesaysv_vehiclebus_backend_aidl.so`, `libdesaysv_vehiclebus_backend_aosp.so`, `libproperties_vehicle.so`, and Android Automotive Vehicle HAL libraries.

Static inspection proves:

- service executable links Vehicle HAL + Desay VehicleBus + `libdesaysv_vehicledevice.so` and imports `VehicleBusStub::get/set/bulkGet/publish/subscribe/unsubscribe`;
- service contains `vdev.service.VehicleDevice|vehiclebus` and log string `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s`;
- service registers a callback through `SVPVehicleDevice::setVehiclePropertyConfigCallback(ISVPVehiclePropertyConfigCallback*)`;
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, and `setVehiclePropertyConfigCallback` plus project-config parsing functions;
- `libdesaysv_vehicledevice.so` contains `vehicle.persist.project.ext.configs` through `ext.configs9`, project code/PN/phonelink keys, `countryCode`, and `ro.sys.ivi.eol.country.code`.

### Newly traced project-config behavior

`VehicleHal::setProjectExtConfigs(key,value)` is now statically traced:

- reads the previous value through `VehicleConfigStore::getProjectConfig(key)`;
- if the new value is empty, returns without publishing/updating;
- if the new value equals the stored value, returns without further update;
- otherwise writes `VehicleConfigStore::setProjectConfig(key,value)`;
- recognizes the first four EOL config keys by exact string comparisons/lengths and calls `setOrUpdateEOLConfigs1/2/3/4(...)` for `vehicle.persist.project.ext.configs`, `ext.configs2`, `ext.configs3`, and `ext.configs4`;
- after the store/EOL update, if the registered callback exists, invokes its virtual method with the unchanged `(key,value)` pair.

`VehicleHal::onVehiclePropertyConfigChange(key,value)` is also traced:

- logs the key/value;
- checks only whether the callback object exists;
- if present, forwards `(key,value)` directly through callback virtual slot `+0x10`;
- no country/project/market/telematics/Fragrance-specific branch is present in this function.

Therefore a hidden country/project suppression inside these two exact HAL functions is DISPROVEN. Country-related strings exist elsewhere in the same library, but they are not gating the callback in the traced functions.

The remaining unproven edge is service-side `VehicleDeviceVDS::onVehiclePropertyConfigChange(key,value)` -> construction/publication of the logical VehicleBus event (expected framework event 918905). The previous report did not show publish callsites because the grep searched the demangled spelling while llvm-objdump retained the mangled PLT name.

Raw 32-bit searches for `918905` / `917510` in the first native targets remain negative; this does not disprove the event path.

## DISPROVEN / closed unless new evidence

- Engineering writes the wrong Fragrance bit.
- New HVAC uses another config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC alone solves the issue.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` controls visibility.
- RU06 -> RU02 HVAC application deltas contain the missing-Fragrance gate.
- `OfflineConfigManager.h()` is Fragrance; it is ionizer/config91.
- `CarConfigUtil` contains a separate Fragrance-specific suppression after `getConfig(50)`.
- Any scanned ordinary APK/JAR implements VehicleDevice.
- `DesaySVProjectService.apk` or `SVVDSCarStateService.apk` implements VehicleDevice.
- Vendor needs another APK/JAR search; the relevant backend is native.
- `VehicleHal::setProjectExtConfigs()` or `VehicleHal::onVehiclePropertyConfigChange()` applies a country/project/market/telematics gate before forwarding a changed non-empty project-config value.

## Current open question

What exactly does service-side `VehicleDeviceVDS::onVehiclePropertyConfigChange(key,value)` construct and send through VehicleBus, where is event `918905` assigned, and is any remaining filtering applied in that service/VehicleBus glue layer?

## Next step

Use the existing service disassembly and search the mangled `VehicleBusStub::publish` PLT/callsite plus the xref to the `VehicleDeviceVDS::onVehiclePropertyConfigChange` log string. Then inspect only that callback function and its event construction. Follow into `libdesaysv_vehiclebus.so` only if the event ID/construction is delegated there.

Do not return to broad APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle becomes available again, pull/hash live HVAC/CarInfo APKs and reconcile artifact identity.
