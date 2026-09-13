# RU02-HVAC NEXT

## Current objective

Identify the real RU02 Fragrance availability/capability mechanism that keeps OEM Fragrance hidden despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Native project-config transport is now statically closed end-to-end.
- `VehicleHal::setProjectExtConfigs/onVehiclePropertyConfigChange` forwards changed property key/value pairs without a country/project/market/telematics filter in the traced path.
- Mini-debug proves the real callback is `VehicleDeviceVDS::onVehiclePropertyConfigChange(...)` at `0x7b28`; `0x7c24` is only its non-virtual thunk.
- Callback constructs event ID `918905`, inserts original `proKey` and `proValue` into a two-string `VehicleBusBundle`, then publishes through `VehicleBusStub::publish(event)`.
- `VehicleDeviceVDS` vtable symbol is at `0x11040`; primary address point used by the object is `0x11050`; virtual slot `+0x38` resolves to relocation `0x11088 = VehicleBusStub::publish(VehicleBusEvent const&)`.
- Therefore missing Fragrance should no longer be attributed to failure of `vehicle.persist.project.ext.configs` event transport without new contradictory evidence.
- Literal names of bundle-key globals `0x135a0` and `0x135b8` remain unresolved, but their values are proven to be callback key/value and this is documentation-only.

## Next step

Pivot away from VehicleDevice/event-918905 transport.

Do one targeted static reference audit of the current RU02 HVAC decompile for Fragrance-specific runtime inputs beyond config50:

1. enumerate every Java/smali reference to Fragrance-specific HVAC/VDBus state, especially IDs corresponding to `AC_FRAGRANCE_DISPLAY`, `AC_FRAGRANCE_WARNING`, `AC_FRAGRANCE_CONSISTENCE_LEVEL`, `AC_FRAGRANCE`, POS1/2/3 fragrance type, and welcome-fragrance state;
2. identify which handlers/subscriptions receive those values and which methods touch Fragrance views, navigation, visibility, or availability;
3. separate display/availability predicates from ordinary control/status values;
4. determine the smallest additional runtime predicate that could leave `fragrance_btn` hidden while `OfflineConfigManager.f()` is true;
5. only after locating such a predicate, trace its producer into VDBus/VehicleService/native backend.

Do not return to broad APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle becomes available again, reconcile live HVAC/CarInfo hashes with local artifacts before runtime tests that depend on exact APK identity.
