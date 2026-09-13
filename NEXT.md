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
- the existing ion-animation calls themselves are unchanged;
- both JADX runs process 1449 classes and report 22 errors, but the captured console logs contain only the counts, so the individual error sets have not been compared.

The canonical vehicle is temporarily unavailable for about six days. This defers live APK hash identity but does not block offline analysis.

Immediate read-only offline discriminator:

1. identify the implementation and meaning of `com.desaysv.svhvac.h.a.a().h()` in RU06 and RU02-labeled HVAC;
2. enumerate all call sites to that helper and determine whether it is ion/air-quality-specific or a broader feature/capability gate;
3. verify the `K1(boolean)` delta directly at smali/DEX level rather than relying only on JADX;
4. if unrelated to Fragrance, decode/diff the HVAC AndroidManifest and `resources.arsc` changes;
5. if those also do not explain Fragrance, pivot to system/framework/backend comparison (`vdbus_extra.jar`, VehicleDevice, VehicleService and related persistent config/event path) using the available firmware payloads.

Artifact identity remains open: earlier live `dumpsys` CarInfo versionName matches the local artifact currently labeled RU06, not the RU02-TEL-2026-labeled artifact. When the vehicle becomes available, pull/hash the live CarInfo and HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- No blind VDBus/property writes.
- No unsigned system APK patch path.
- No package replacement until a concrete static/runtime hypothesis exists.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
