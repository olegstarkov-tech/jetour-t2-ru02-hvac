# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Next step

Do not return to APK patching and do not install more packages yet.

The focused RU06 vs RU02-labeled application comparison is now effectively closed as a Fragrance discriminator:

- CarInfo generated Java differs only in `BuildConfig.VERSION_NAME`; no functional application-code delta is currently visible.
- HVAC DEX behavior differs only in `com/desaysv/svhvac/view/b.java::K1(boolean)`.
- RU02 adds `OfflineConfigManager.h()` plus early return before the existing `ionAnimation` block.
- `h()` is definitively `isIonExist`, checking config ID 91.
- Fragrance is separately `OfflineConfigManager.f()`, checking config ID 50 plus `!isT1H_PHEV()`.
- Direct apktool/smali diff confirms the same ionizer-only DEX delta.
- Decoded `AndroidManifest.xml` RU06 vs RU02-labeled is semantically identical.
- Decoded resource comparison finds only localization/string changes: 13 differing common `strings.xml` files plus one RU02-only `values-ms-rMY/strings.xml`.
- No decoded layout, bool, integer, style, id, array, drawable, alias or visibility resource changed.
- Russian Fragrance strings are unchanged. Fragrance-related resource differences are only translations/localization and the default-English typo fix `fragnance` -> `fragrance`.
- Therefore the observed RU06 -> RU02 HVAC APK delta does not provide a mechanism for hiding Fragrance.

The canonical vehicle is temporarily unavailable for about six days. This defers live APK hash identity but does not block offline analysis.

Immediate read-only offline task:

1. pivot to the RU02 system/framework/backend path rather than further HVAC APK diffing;
2. inventory the available RU02 firmware payload for `vdbus_extra.jar`, VehicleDevice, VehicleService and related Desay vehicle/VDBus components;
3. extract/decompile the relevant RU02 framework/service artifacts and map the Fragrance/config50 path end-to-end: persistent config -> EolConfig/CarConfigUtil -> VDBus/VehicleDevice/VehicleService -> HVAC client/event path;
4. identify any firmware-specific capability, market/project, telematics or event/update gating that can make config50 present yet leave Fragrance unavailable;
5. use RU05/RU06 firmware system components only as comparison references when matching artifacts are available; do not transfer conclusions automatically across firmware generations.

Artifact identity remains open: earlier live `dumpsys` CarInfo versionName matches the local artifact currently labeled RU06, not the RU02-TEL-2026-labeled artifact. When the vehicle becomes available, pull/hash the live CarInfo and HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- No package replacement until a concrete static/runtime hypothesis exists.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
