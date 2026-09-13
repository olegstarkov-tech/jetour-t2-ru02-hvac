# RU02-HVAC NEXT

## Current objective

Identify the real RU02 Fragrance availability/capability mechanism that keeps OEM Fragrance hidden despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Native project-config transport is now statically closed end-to-end.
- `VehicleHal::setProjectExtConfigs/onVehiclePropertyConfigChange` forwards changed property key/value pairs without a country/project/market/telematics filter in the traced path.
- Mini-debug proves the real callback is `VehicleDeviceVDS::onVehiclePropertyConfigChange(...)` at `0x7b28`; `0x7c24` is only its non-virtual thunk.
- Callback constructs event ID `918905`, inserts original `proKey` and `proValue` into a two-string `VehicleBusBundle`, then publishes through `VehicleBusStub::publish(event)`.
- Therefore missing Fragrance should no longer be attributed to failure of `vehicle.persist.project.ext.configs` event transport without new contradictory evidence.
- Current static audit is against local `SVHvac_RU02_2026.apk` SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`, matching the established local RU02-labeled hash.
- The broad runtime-input grep did not expose a semantic `FragranceDisplay`/`FragranceWarning` consumer and found no direct numeric cases for candidate IDs 58/59/60/61/62/64/66/71/76/88. This is inconclusive for obfuscated presenter/backend code.
- Visible UI callbacks only prove operational behavior: Fragrance power updates `fragranceBtn.setSelected`, while FragranceDialog handles power/type/level/remain/position. No direct main-button visibility change was found in those paths.
- `b.a.d.a.b.x0` is the concrete FragrancePresenter type used by both activity/view/dialog code and is now the highest-value static target.

## Next step

Trace `b.a.d.a.b.x0` FragrancePresenter only:

1. dump/decompile the complete `x0.java` source from the verified RU02 HVAC JADX tree;
2. identify its superclass, implemented interfaces, proxy objects, registration/unregistration methods, and callback listener types;
3. map public methods used by the UI (`l`, `n`, `o`, `p`, `q`, `u`, `w`, `x`, plus any others) to concrete property/event reads and writes;
4. enumerate every event/property ID and any CarInfo/HVAC/VDBus subscription used by x0;
5. inspect all callbacks delivered by x0 and distinguish operational state (power/type/level/remain/position) from any availability/display/capability state;
6. only if a concrete candidate visibility predicate is found, trace that one producer into CarInfo/VDBus/native backend.

Do not preselect `AC_FRAGRANCE_DISPLAY` as root cause without a concrete x0 consumer path. Do not return to broad APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle becomes available again, reconcile live HVAC/CarInfo hashes with local artifacts before runtime tests that depend on exact APK identity.
