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

The focused HVAC RU06 vs RU02-labeled generated-source diff is also complete:

- exactly one generated Java file differs: `com/desaysv/svhvac/view/b.java`;
- the only shown Java-code delta is `K1(boolean)`;
- RU02-labeled code adds `com.desaysv.svhvac.h.a.a().h()` as a guard around the existing `ionAnimation` update block;
- `com.desaysv.svhvac.h.a` is `OfflineConfigManager`;
- RU06 and RU02-labeled decompiled `OfflineConfigManager.java` are byte-identical;
- `h()` is definitively `isIonExist`, checking config ID 91 and logging `isIonExist`;
- Fragrance is separately `f()`, checking config ID 50 plus `!isT1H_PHEV()`;
- call sites of `h()` are ionizer-specific (`onIonClick`, `ion is no exist`, `isIonExistShow`);
- therefore the sole generated-Java RU06->RU02 delta is currently classified as ionizer/config91-specific, not Fragrance/config50-specific.

The canonical vehicle is temporarily unavailable for about six days. This defers live APK hash identity but does not block offline analysis.

Immediate read-only offline discriminator:

1. verify the `K1(boolean)` RU06 vs RU02 delta directly at smali/DEX level rather than relying only on JADX;
2. if smali confirms only the `config91` ionizer guard, close the Java-code delta as unrelated to Fragrance;
3. decode and diff the HVAC AndroidManifest and `resources.arsc` changes;
4. if those also do not explain Fragrance, pivot to system/framework/backend comparison (`vdbus_extra.jar`, VehicleDevice, VehicleService and related persistent config/event path) using the available firmware payloads;
5. use RU05 only as the older reference generation once a concrete changed framework/backend component is identified.

Artifact identity remains open: earlier live `dumpsys` CarInfo versionName matches the local artifact currently labeled RU06, not the RU02-TEL-2026-labeled artifact. When the vehicle becomes available, pull/hash the live CarInfo and HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- No blind VDBus/property writes.
- No unsigned system APK patch path.
- No package replacement until a concrete static/runtime hypothesis exists.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
