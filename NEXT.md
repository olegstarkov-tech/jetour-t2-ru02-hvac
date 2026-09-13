# RU02-HVAC NEXT

## Current objective

Identify the RU02 backend condition that blocks OEM Fragrance despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Ordinary Java ownership search is negative across scanned `system`, `product`, and `system_ext` APK/JAR containers.
- Full recursive `vendor.img` inventory is complete.
- Vendor contains only 2 APKs, 0 JARs, 1 ODEX, 1 VDEX, 0 OAT and 0 APEX.
- Vendor contains a dedicated native Desay VehicleDevice stack: init rc `com.desaysv.vehicledevice@1.0-service.rc`, service executable `com.desaysv.vehicledevice@1.0-service`, interface libraries `com.desaysv.vehicledevice@1.0.so`, and implementation library `libdesaysv_vehicledevice.so`.
- Related native vehicle bus and Android Vehicle HAL components are present in the same vendor image.

## Next step

Inspect only this native VehicleDevice launch/implementation chain read-only:

1. capture the VehicleDevice init rc;
2. record SHA-256/file type and dynamic dependencies for the service executable and `libdesaysv_vehicledevice.so`;
3. inspect exported/dynamic symbols and strings for config/property/store/vehiclebus/Vehicle HAL terms plus Fragrance/project/market/region/TBox/capability terms;
4. search for references to `vehicle.persist.project.ext.configs`, event 918905 / `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`, and nearby config-store concepts;
5. follow into `init.desaysv.vehicle.rc`, vehiclebus libraries or Vehicle HAL only when direct dependencies/references require it.

Do not return to broad APK guessing or unsigned HVAC patching.

When the vehicle is available again, reconcile live HVAC/CarInfo hashes with local artifacts.
