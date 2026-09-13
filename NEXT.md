# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

The RU06 vs RU02-labeled HVAC application delta is effectively closed as a Fragrance discriminator: the only confirmed DEX behavior change is ionizer/config91-specific, decoded manifest is semantically identical, and decoded resource changes are localization-only.

The canonical vehicle is temporarily unavailable; live APK hash identity is deferred without blocking offline analysis.

The RU02 OTA payload contains Android `system`, `system_ext`, `product`, `vendor` and separate `system_qnx`. Android `system.img` has been extracted read-only (967962624 bytes; SHA-256 `19ac46038cd8662891a8da6319844b31b0cadaf5e82e156108a91b474771265d`).

Inside the image the correct framework directory is `/system/framework`. It contains:

- `vdbus.jar` — 1395532 bytes;
- `vdbus_extra.jar` — 111020 bytes;
- `chery-platform-internal.jar` — 41604 bytes;
- related candidates `car-frameworks-service.jar` and `desaysv-car-frameworks-service-extension.jar`.

## Next step

1. Extract exact RU02 `/system/framework/vdbus_extra.jar` and `/system/framework/vdbus.jar` from `system.img`.
2. Record SHA-256 and archive contents.
3. Decompile both read-only.
4. Map Fragrance/config50 through `EolConfig`, `CarConfigUtil`, VDBus and any `VehicleDevice`/`VehicleService` references.
5. Expand to the other framework jars only if the call/reference graph requires it.

When the vehicle is available again, pull/hash the live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
