# RU02-HVAC NEXT

## Current objective

Identify the RU02 native backend condition that blocks OEM Fragrance despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Ordinary Java ownership search is negative across scanned `system`, `product`, `system_ext`, and vendor APK/JAR populations.
- Full vendor inventory proves the relevant backend is native, not a hidden APK/JAR.
- Native service `/vendor/bin/hw/com.desaysv.vehicledevice@1.0-service` links Android Automotive Vehicle HAL and Desay VehicleBus layers.
- `libdesaysv_vehicledevice.so` proves the project-config callback flow and contains no country/project/market/telematics filter inside `VehicleHal::onVehiclePropertyConfigChange(key,value)` itself.
- Native service explicitly constructs event 918905 (`0x000E0579`) at `0x7ba8/0x7bb0` and `0x7ca4/0x7cac`.
- Each 918905 block immediately performs two `VehicleBusBundle::putString(...)` calls: `0x7bc4/0x7bd8` and `0x7cc0/0x7cd4` respectively.
- `VehicleBusStub::publish()` is imported through an `R_AARCH64_ABS64` relocation at `0x11088`, not an ordinary PLT/JUMP_SLOT call. `VehicleBusStub::set()` does have a normal JUMP_SLOT/PLT entry.
- Therefore the remaining native bridge problem is now very narrow: decode the two bundle string fields and prove the indirect call path through the `publish` slot.

## Next step

Do one focused service-side trace only:

1. disassemble `0x7b80-0x7d10` with enough raw register flow to recover arguments to all four `VehicleBusBundle::putString` calls;
2. map any rodata/string constants loaded before those calls and identify the exact bundle field names;
3. determine whether the two values passed are the callback property key/value;
4. identify the ELF section containing relocation address `0x11088` and enumerate every code xref to that slot/page+offset;
5. follow only those xrefs until the exact indirect branch/call to `VehicleBusStub::publish()` is proven;
6. once the callback -> 918905 bundle -> publish edge is closed, inspect only conditional branches in that exact function/path for country/project/product/telematics/capability filtering.

Do not return to APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle is available again, reconcile live HVAC/CarInfo hashes with local artifacts and run only runtime tests implied by this static trace.
