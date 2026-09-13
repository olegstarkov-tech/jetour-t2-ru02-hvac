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

However a more important artifact-identity inconsistency is now open: the earlier live vehicle `dumpsys package com.desaysv.ivi.vds.carinfo` versionName (`Chery-8155_11_e5fc2d4_2025-09-26_2509261457_R`) matches the local artifact labeled RU06, whereas the local artifact labeled RU02-TEL-2026 decompiles with versionName `Chery-8155_11_8f9dc0d_2026-04-10_2604101647_R`.

Immediate read-only discriminator:

1. pull the live `com.desaysv.ivi.vds.carinfo` APK from the canonical T1J RU02 vehicle;
2. calculate its SHA-256 and capture live `versionName/versionCode`;
3. compare against the local RU06 and RU02-TEL-2026 CarInfo hashes;
4. correct artifact labels if needed;
5. only then perform the focused HVAC source diff using the artifact that is proven to correspond to the live RU02 stack.

After live identity is resolved, continue with changed HVAC classes first; if the HVAC code delta is only build metadata too, pivot directly to system/framework/shared-library resolution (`vdbus_extra.jar`, VehicleDevice, VehicleService) instead of further APK replacement.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- No blind VDBus/property writes.
- No unsigned system APK patch path.
- No package replacement until the static/live identity is established.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
