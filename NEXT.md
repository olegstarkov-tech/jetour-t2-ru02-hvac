# RU02-HVAC NEXT

## Current objective

Identify the RU02 native backend condition that blocks OEM Fragrance despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Ordinary Java ownership search is negative across scanned `system`, `product`, `system_ext`, and vendor APK/JAR populations.
- Full vendor inventory proves the relevant backend is native, not a hidden APK/JAR.
- Native service `/vendor/bin/hw/com.desaysv.vehicledevice@1.0-service` links both Android Automotive Vehicle HAL and Desay VehicleBus layers.
- The service imports `VehicleBusStub::publish()` plus get/set/bulkGet/subscribe/unsubscribe and contains `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s`.
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, `setVehiclePropertyConfigCallback`, and related project-config handlers.
- `VehicleHal::setProjectExtConfigs(key,value)` suppresses empty/unchanged values, stores changed project config and forwards the same key/value through the property-config callback.
- `VehicleHal::onVehiclePropertyConfigChange(key,value)` contains no country/project/market/telematics filter; it logs and forwards the same pair to the callback.
- Native service disassembly now proves construction of event ID 918905 (`0x000E0579`) at two sites: `0x7ba8/0x7bb0` and `0x7ca4/0x7cac`.
- Those sites lie in the same local service region as xrefs to the exact `VehicleDeviceVDS::onVehiclePropertyConfigChange` log string.

## Next step

Do one narrow service-side disassembly trace only:

1. dump `0x7b40-0x7d20` from `com.desaysv.vehicledevice@1.0-service`;
2. capture `readelf -rW` / PLT relocation information for `VehicleBusStub::publish`;
3. identify the function/basic-block boundaries containing `0x7ba8` and `0x7ca4`;
4. decode the `VehicleBusEvent` constructor fields and `VehicleBusBundle::putString/putInt` calls around those blocks;
5. prove the exact call edge to `VehicleBusStub::publish()` or, if indirect, identify the exact virtual/PLT target;
6. only then inspect conditional branches in that exact publication path for country/project/product/telematics/capability gating.

Do not return to APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle is available again, reconcile live HVAC/CarInfo hashes with local artifacts and run only runtime tests implied by this static trace.
