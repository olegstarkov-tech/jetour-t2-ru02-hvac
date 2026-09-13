# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

The RU06 vs RU02-labeled HVAC application delta is effectively closed as a Fragrance discriminator. The framework config path is mapped through `CarConfigUtil` and `EolConfig`, and VDBus identifies the event source as `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.

Exact class ownership localization now excludes ordinary Java containers in three partitions:

- 88 APKs across `system/app`, `system/priv-app`, `product/app`, `product/priv-app`: zero exact owners.
- `system_ext.img` extracted read-only; SHA-256 `1b9902976b45277875d4a9b79d49fa4a26599dc9cdb682459c5fa1acfe47582b`.
- Expanded exact DEX `class_def` scan across `system/framework`, `product/framework`, `system_ext/framework`, plus `system_ext` app/priv-app locations checked 102 Java archives total.
- Zero exact definitions of `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;` were found.

Therefore do not return to manual APK-name guessing or raw string greps.

## Next step

1. Extract RU02 `vendor.img` from the existing payload read-only.
2. Record vendor image SHA-256 and root layout.
3. Run the same exact DEX `class_def` search for `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;` across vendor APK/JAR locations:
   - `/app`, `/priv-app`, `/framework`;
   - `/vendor/app`, `/vendor/priv-app`, `/vendor/framework` if the image contains an inner `/vendor` directory.
4. If an exact owner is found, fingerprint and decompile only that owner, then trace event `918905` production and the origin/filtering of `vehicle.persist.project.ext.configs`.
5. Only if vendor also has zero exact owner, investigate OAT/VDEX/APEX/native/system-service packaging and boot/preopt class ownership.
6. Follow into `VehicleService` only where the actual VehicleDevice implementation/reference graph requires it.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
