# RU02-HVAC NEXT

## Current objective

Identify the RU02 native backend condition that blocks OEM Fragrance despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Ordinary Java ownership search is negative across scanned `system`, `product`, `system_ext`, and vendor APK/JAR populations.
- Full vendor inventory proves the relevant backend is native, not a hidden APK/JAR.
- Native service `/vendor/bin/hw/com.desaysv.vehicledevice@1.0-service` links both Android Automotive Vehicle HAL and Desay VehicleBus layers.
- The service imports `VehicleBusStub::publish()` plus get/set/bulkGet/subscribe/unsubscribe.
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, `setVehiclePropertyConfigCallback`, and related project-config handlers.
- The same implementation library contains `vehicle.persist.project.ext.configs` through `ext.configs9`, project code/PN/phonelink keys, `countryCode`, and `ro.sys.ivi.eol.country.code`.
- This localizes the exact RU02 project-config ingestion layer. The still-unproven edge is how its key/value callback becomes the framework VehicleBus/VDBus event path and whether a condition/filter is applied there.
- Raw literal 32-bit searches for 918905/917510 in the first three native targets were negative; this does not close the event path.

## Next step

Do one focused static trace, not another broad scan:

1. inspect defined symbols/classes in `com.desaysv.vehicledevice@1.0-service` for the implementation of `ISVPVehiclePropertyConfigCallback` or equivalent project-property callback;
2. locate service call sites to `VehicleBusStub::publish()`;
3. disassemble `VehicleHal::onVehiclePropertyConfigChange(...)`, `VehicleHal::setProjectExtConfigs(...)`, and `VehicleHal::requestProjectConfigs()` from `libdesaysv_vehicledevice.so`;
4. correlate callback key/value handling with VehicleBus event construction/publish;
5. only if construction is delegated, follow the exact referenced function into `libdesaysv_vehiclebus.so` or its selected backend;
6. specifically note branches involving country/project/product IDs, `ro.sys.ivi.eol.country.code`, telematics/TBox/capability, or filtering of `vehicle.persist.project.ext.configs`.

Do not return to APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle is available again, reconcile live HVAC/CarInfo hashes with local artifacts and run only the runtime tests implied by this static trace.
