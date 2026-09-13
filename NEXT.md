# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

The RU06 vs RU02-labeled HVAC application delta is effectively closed as a Fragrance discriminator. The framework config path is now mapped through `CarConfigUtil` and `EolConfig`, and the VDBus event source is identified as `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.

Exact class ownership localization has progressed:

- `DesaySVProjectService.apk` does not define VehicleDevice.
- `SVVDSCarStateService.apk` does not define VehicleDevice; it contains only client-side `VehicleDeviceManager`.
- Exact DEX `class_def` scanning was run across all APKs in RU02 `system/app`, `system/priv-app`, `product/app`, and `product/priv-app`.
- 88 APKs were checked for descriptor `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Zero exact owners were found.

Therefore do not continue manual APK-name guessing and do not infer ownership from raw DEX strings.

## Next step

1. Extract RU02 `system_ext.img` from the existing `payload.bin` if not already extracted.
2. Expand exact class-definition search to:
   - `system_ext` APKs and JARs;
   - `/system/framework` JARs;
   - `/product/framework` JARs;
   - `/system_ext/framework` JARs.
3. Use the same exact DEX class-table test for `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`, not a raw string grep.
4. If an exact owner is found, fingerprint and decompile only that owner, then trace event `918905` production plus origin/filtering of `vehicle.persist.project.ext.configs`.
5. If no owner is found in those Java archives, next investigate preoptimized/oat/apex/native service packaging rather than returning to application APKs.
6. Follow into `VehicleService` only where the actual VehicleDevice implementation/reference graph requires it.

When the vehicle is available again, pull/hash live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
