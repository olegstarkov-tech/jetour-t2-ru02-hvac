# RU02-HVAC NEXT

## Current objective

Identify the real RU02 Fragrance availability/capability mechanism that keeps OEM Fragrance hidden despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Native project-config transport is now statically closed end-to-end.
- `VehicleHal::setProjectExtConfigs/onVehiclePropertyConfigChange` forwards changed property key/value pairs without a country/project/market/telematics filter in the traced path.
- Native `VehicleDeviceVDS` callback constructs event ID `918905`, inserts original `proKey` and `proValue` into a two-string `VehicleBusBundle`, then calls `VehicleBusStub::publish(event)` through vtable slot `+0x38`.
- Primary class vptr is `0x11050`; slot `0x11088` is relocated directly to `VehicleBusStub::publish(VehicleBusEvent const&)`, proving the indirect call.
- Bundle field-name objects are globals at `0x135a0` and `0x135b8`; their payload values are proven to be callback key/value, but literal field names remain unresolved.
- Therefore missing Fragrance should no longer be attributed to failure of `vehicle.persist.project.ext.configs` event transport without new contradictory evidence.

## Next step

Do one final targeted symbol pass on the same service binary before pivoting away from transport:

1. extract `.gnu_debugdata` from `/vendor/bin/hw/com.desaysv.vehicledevice@1.0-service`;
2. decompress it and enumerate local symbols/vtables with `nm -C -n` / `readelf -Ws`;
3. identify exact names for the callback at `0x7b28`, its secondary thunk around `0x7c24`, and the vtable beginning at `0x11050` if mini-debug data contains them;
4. inspect xrefs/initializers for global `std::string` objects `0x135a0` and `0x135b8` to recover their literal field names if possible;
5. once documented, stop spending effort on 918905 transport and pivot to other Fragrance capability/availability inputs in RU02 backend/framework/HVAC logic.

Do not return to broad APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle becomes available again, reconcile live HVAC/CarInfo hashes with local artifacts and run only runtime tests implied by the static findings.
