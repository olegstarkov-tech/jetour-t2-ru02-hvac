# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Next step

Do not return to APK patching and do not install more packages yet.

A six-APK RU05/RU06/RU02-TEL comparison set is now available. The immediate read-only discriminator is to compare internal APK entries by hash before full decompile:

1. hash `classes.dex`, `resources.arsc`, `AndroidManifest.xml` and other matching ZIP entries for all three HVAC APKs and all three `SVVDSCarInfo` APKs;
2. determine whether RU06 and RU02 `SVVDSCarInfo` code is byte-identical despite different whole-APK hashes;
3. determine exactly which internal entries differ between RU06 and RU02 HVAC;
4. then run focused JADX diff only on changed code paths and Fragrance/VDBus/carconfig-related classes.

The current leading structural observation is a generation boundary between RU05 and RU06: RU05 APKs expose a larger embedded VDBus/carconfig implementation symbol set, whereas RU06/RU02 appear to use the newer packaging model. Treat this as a direction, not a root-cause conclusion.

After the internal-entry diff, decide whether to inspect class-loader/shared-library resolution (`vdbus_extra.jar`) or a concrete changed HVAC/VDS code path first.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- No blind VDBus/property writes.
- No unsigned system APK patch path.
- No package replacement until the static diff produces a concrete hypothesis.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
