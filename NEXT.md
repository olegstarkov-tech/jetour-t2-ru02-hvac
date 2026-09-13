# RU02-HVAC NEXT

## Current objective

Identify the RU02 native backend condition that blocks OEM Fragrance despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event `918905` -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Relevant RU02 backend is native: `com.desaysv.vehicledevice@1.0-service` + `libdesaysv_vehicledevice.so` + Desay VehicleBus libraries.
- Service imports `VehicleBusStub::publish()` and registers `ISVPVehiclePropertyConfigCallback` through `SVPVehicleDevice::setVehiclePropertyConfigCallback(...)`.
- Service contains `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s`.
- `VehicleHal::setProjectExtConfigs(key,value)` is traced: ignore empty/unchanged values; otherwise update store/EOL config then invoke registered callback with the same key/value.
- `VehicleHal::onVehiclePropertyConfigChange(key,value)` is traced: log then directly forward key/value to callback when present.
- No country/project/market/telematics/Fragrance-specific filter exists in these two HAL functions.
- Previous publish-callsite section was blank because grep searched demangled `VehicleBusStub::publish`, while llvm-objdump retained the mangled PLT spelling.

## Next step

Do one focused service-side trace:

1. locate the PLT relocation/address for mangled `_ZN14VehicleBusStub7publishERK15VehicleBusEvent` in `com.desaysv.vehicledevice@1.0-service`;
2. find every `bl` callsite to that PLT address in the existing service disassembly;
3. locate the code xref to the service log string `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s` (string file offset `0x4929` in the current binary report; verify its runtime VA from ELF sections rather than assuming offset==VA);
4. dump the complete containing callback function and correlate its event ID/bundle construction with the publish call;
5. only if event ID construction is delegated, follow that exact referenced function into `libdesaysv_vehiclebus.so` or its backend.

Do not broaden back to APK/OAT/VDEX searches or blind writes.

When the vehicle is available again, reconcile live HVAC/CarInfo hashes and run only runtime tests implied by this static trace.
