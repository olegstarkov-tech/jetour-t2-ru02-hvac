# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Next step

Do not return to APK patching and do not install more packages yet.

The internal-entry hash discriminator is complete. RU06 -> RU02 is a narrow package delta:

- HVAC: 2948 identical entries, 6 changed, 0 added/removed. The Fragrance XML layouts are byte-identical. Changed payload is limited to DEX, manifest, resources table and signature metadata.
- CarInfo: 593 identical entries, 5 changed, 0 added/removed. `resources.arsc` is byte-identical. Changed payload is limited to DEX, manifest and signature metadata.
- RU06 and RU02 CarInfo DEX files are equal-size but not byte-identical.

Immediate read-only task:

1. Decompile RU06 and RU02 `SVVDSCarInfo` with the same JADX version.
2. Produce a recursive source-file diff and list only changed Java classes.
3. Repeat for RU06 and RU02 HVAC.
4. Inspect only those changed classes for Fragrance, config50, VDBus, VehicleDevice/VehicleService, market/model, telematics, capability/support/availability gating, and startup/config refresh behavior.
5. Decode/diff the two AndroidManifests and only then inspect `resources.arsc` deltas in HVAC if code diff does not explain the change.

Use RU05 afterward as the older reference architecture to understand any changed class or contract found in RU06/RU02; do not start with a full RU05 source-tree diff because the generation delta is much larger.

After the focused class diff, decide whether the next target is a concrete changed application class or system/framework/shared-library resolution (`vdbus_extra.jar`, VehicleDevice, VehicleService).

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- No blind VDBus/property writes.
- No unsigned system APK patch path.
- No package replacement until the static diff produces a concrete hypothesis.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
