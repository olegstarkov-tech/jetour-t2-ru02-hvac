# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

RU02 framework-level config flow is mapped through `VehicleDevice` event `918905` -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.

Two manually selected APK candidates have now been conclusively closed as implementation owners:

- `/system/priv-app/DesaySVProjectService/DesaySVProjectService.apk`
- `/product/app/SVVDSCarStateService/SVVDSCarStateService.apk`

Both contain raw references to `com.desaysv.ivi.vds.vdev.service.VehicleDevice`, but JADX source-tree verification shows neither defines that class. `SVVDSCarStateService` contains only a client-side `VehicleDeviceManager`.

Therefore manual package-name guessing is no longer useful.

## Next step

1. Perform an automated exact class-definition search across RU02 Android artifacts, beginning with all APKs in `system` and `product`.
2. Search for actual ownership of `com/desaysv/ivi/vds/vdev/service/VehicleDevice`, not raw DEX strings.
3. If not found in `system`/`product`, extract/search `system_ext` and relevant framework JARs next.
4. Once the owning artifact is found, fingerprint it, inspect manifest/package metadata, decompile only that artifact, and trace where event `918905` is produced and how `vehicle.persist.project.ext.configs` is sourced/filtered.
5. Follow into `VehicleService` only if the real VehicleDevice implementation directly requires it.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
