# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Next step

Do not return to APK patching and do not install more packages yet.

The focused CarInfo RU06 vs RU02-labeled source diff is complete:

- only `com/desaysv/ivi/vds/carinfo/BuildConfig.java` differs in generated Java;
- the only shown Java delta is `VERSION_NAME`;
- the normalized JADX WARN/ERROR reports are identical as 662-line multisets, including the same AndroidX `DiffUtil.java:107` RegionMakerVisitor failure;
- therefore there is currently no evidence of a functional application-code difference in CarInfo between these two artifacts.

The focused HVAC RU06 vs RU02-labeled DEX diff is now also closed as Fragrance-irrelevant:

- exactly one generated Java file differs: `com/desaysv/svhvac/view/b.java`;
- the only behavior delta is `K1(boolean)`;
- RU02-labeled code adds `OfflineConfigManager.h()` before the existing `ionAnimation` block;
- `h()` is definitively `isIonExist`, checking config ID 91;
- Fragrance is separately `f()`, checking config ID 50 plus `!isT1H_PHEV()`;
- direct apktool/smali diff confirms RU02 inserts only the `h()` call plus early `return-void`; the extracted `h()Z` method itself is unchanged;
- therefore this DEX delta does not explain missing Fragrance.

The canonical vehicle is temporarily unavailable for about six days. This defers live APK hash identity but does not block offline analysis.

Immediate read-only offline discriminator:

1. decode and diff RU06 vs RU02-labeled HVAC `AndroidManifest.xml` using the same apktool/aapt toolchain;
2. produce a normalized resource-table diff for `resources.arsc`, focusing first on values/bools/integers/styles/ids/aliases and any entries referenced by Fragrance UI;
3. verify whether any resource indirection changes `fragrance_btn` visibility, layout selection or feature-specific boolean/integer values despite byte-identical Fragrance XML layouts;
4. if manifest/resources do not explain Fragrance, pivot to system/framework/backend comparison (`vdbus_extra.jar`, VehicleDevice, VehicleService and related persistent config/event path) using the available firmware payloads;
5. use RU05 as the older reference generation once a concrete changed framework/backend component is identified.

Artifact identity remains open: earlier live `dumpsys` CarInfo versionName matches the local artifact currently labeled RU06, not the RU02-TEL-2026-labeled artifact. When the vehicle becomes available, pull/hash the live CarInfo and HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- No blind VDBus/property writes.
- No unsigned system APK patch path.
- No package replacement until a concrete static/runtime hypothesis exists.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
