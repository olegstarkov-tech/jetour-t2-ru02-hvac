# RU02-HVAC NEXT

## Current objective

Identify the real RU02 Fragrance UI activation mechanism that keeps the OEM entry hidden despite config50=1.

## Current state

- Project-config transport through VehicleDevice event 918905 -> VDBus -> CarConfigUtil -> EolConfig -> config50 is statically closed end-to-end.
- Current static audit is against local `SVHvac_RU02_2026.apk` SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`, matching the established local RU02-labeled artifact hash.
- `b.a.d.a.b.x0` is exactly `FragrancePresenter` and contains no hidden availability/visibility condition; it directly delegates to `IFragranceModel` and forwards callbacks.
- `ModelFactory.b()` returns `b.a.b.a.c.b`, compiled as `FragranceModel`.
- `FragranceModel` implements `IFragranceModel` and `CarInfoHelper.ISpiListener`.
- Its exact module is `327690` and exact subscribed/read ID set is `{58,59,60,61,62,71,92,93,94}`.
- Mapping: 58/59/60 = type1/type2/type3, 61 = power, 62 = level, 71 = position/channel, 92/93/94 = remain1/remain2/remain3.
- Writes are only IDs 61, 62, and 71.
- IDs 64 (`AC_FRAGRANCE_DISPLAY`), 66 (`AC_FRAGRANCE_WARNING`), 76 (welcome fragrance), and 88 (happy-egg fragrance) are not consumed by this actual FragranceModel subscription/read/write path.
- Therefore the traced presenter/model path contains ordinary operational state only and no second display/availability capability input.
- Base `bottom_layout.xml` still declares `fragrance_btn` as `GONE`, and inspected `view/b.java` / binding code has no proven path making it visible.

## Next step

Perform one deterministic UI/resource activation audit:

1. enumerate every exact call-site of `OfflineConfigManager.f()` / Fragrance existence predicate;
2. enumerate every source reference to `fragranceBtn` / `fragrance_btn`, especially any `setVisibility`, inflation, include, binding-adapter, or resource-selection path;
3. decode/list every qualified `res/layout*` resource containing `fragrance_btn` and record its initial visibility;
4. identify which concrete bottom layout resource is inflated for the T1J path;
5. scan RU02 system/product/system_ext/vendor overlay APK manifests/resources for overlays targeting `com.desaysv.svhvac` or overriding the relevant layout/resource;
6. decide from evidence whether activation is in-APK UI construction or an external resource overlay.

Do not return to broad backend ID guessing, VehicleDevice transport, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching.

When the vehicle becomes available again, reconcile live HVAC/CarInfo hashes and query active overlays as a runtime cross-check.
