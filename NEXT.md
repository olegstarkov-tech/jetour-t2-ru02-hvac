# RU02-HVAC NEXT

## Current objective

Identify the real RU02 Fragrance availability/capability mechanism that keeps OEM Fragrance hidden despite config50=1.

## Current state

- Framework config path is mapped through VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig` -> config50.
- Native project-config transport is statically closed end-to-end and should not be reopened without contradictory evidence.
- Current static audit is against local `SVHvac_RU02_2026.apk` SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`, matching the established local RU02-labeled hash.
- Broad grep did not expose a concrete `FragranceDisplay`/`FragranceWarning` consumer and found no direct numeric cases for candidate IDs 58/59/60/61/62/64/66/71/76/88. This is inconclusive for obfuscated backend/model code.
- Visible view/dialog callbacks only prove ordinary operational behavior; no direct main-button visibility change has been found.
- `b.a.d.a.b.x0` is exactly `FragrancePresenter` and is now CLOSED as the likely hidden gate layer.
- `x0` simply obtains `IFragranceModel` from `b.a.b.a.c.e.b()`, registers/unregisters a model listener, forwards all callbacks without filtering, and directly delegates its getters/setters to the model.
- Operational mapping from x0/UI usage: `k=level`, `l=position`, `m=power`, `n/o/p=remain1/2/3`, `q/r/s=type1/2/3`.

## Next step

Trace only the concrete `IFragranceModel` implementation returned by factory `b.a.b.a.c.e.b()`:

1. inspect complete `b/a/b/a/c/e.java` and determine the exact class/object returned by `e.b()`;
2. decompile/dump that exact model implementation and any directly referenced fragrance-specific helper/proxy only;
3. map interface methods `r/m0/F0/E/H/f0/h/z/P`, listener registration `x0/o`, lifecycle `a/b/c`, and writes `B0/l/B` to concrete CarInfo/HVAC/VDBus properties/events;
4. record every numeric property/event ID and callback registration used by the model;
5. distinguish operational states (power/type/level/remain/position) from any availability/display/warning/capability state;
6. only if a concrete extra predicate/state is found, trace that one producer into CarInfo/VDBus/native backend.

Do not continue grepping `x0`, do not preselect `AC_FRAGRANCE_DISPLAY` as root cause, and do not return to broad APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle becomes available again, reconcile live HVAC/CarInfo hashes with local artifacts before runtime tests that depend on exact APK identity.
