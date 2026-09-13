# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

The RU06 vs RU02-labeled HVAC application delta is effectively closed as a Fragrance discriminator: the only confirmed DEX behavior change is ionizer/config91-specific, decoded manifest is semantically identical, and decoded resource changes are localization-only.

The canonical vehicle is temporarily unavailable; live APK hash identity is deferred without blocking offline analysis.

RU02 framework extraction is now complete for the first VDBus layer:

- `vdbus.jar` SHA-256 `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`;
- `vdbus_extra.jar` SHA-256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`;
- both decompile cleanly with JADX.

The config update path is now mapped:

1. `CarConfigUtil.init()` initializes VDBus.
2. On `ServiceType.VEHICLE_DEVICE` connection it subscribes to event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`), registers `VDNotifyListener`, commits, and calls `EolConfig.loadConfig()`.
3. On event `918905`, payload is decoded through `VDVDeviceConfigStore` into key/value.
4. Key `vehicle.persist.project.ext.configs` causes `EolConfig.updateConfig(Utils.stringToByte(value), null, null, null, null)`.
5. `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`.
6. `VDServiceDef` names the event source as system service `com.desaysv.ivi.vds.vdev.service.VehicleDevice` in package `com.desaysv.ivi.vds.vdev`.
7. `VehicleService` is a separate HAL-facing service: `com.desaysv.ivi.vds.vehicle.service.VehicleService` under `android.hardware.automotive.vehicle@2.0-service`.

## Next step

1. Locate the actual RU02 APK/package that implements `com.desaysv.ivi.vds.vdev.service.VehicleDevice` in the available firmware images (`product`, `system_ext`, or `system`).
2. Extract only that package read-only and record hash/version/manifest.
3. Decompile it and trace where event `918905` / `VDVDeviceConfigStore` is produced, especially the source of `vehicle.persist.project.ext.configs` and any project/market/telematics/capability filtering.
4. Follow into `VehicleService` only if direct references from `VehicleDevice` require it.
5. Do not broaden into unrelated framework jars until this service path is exhausted.

When the vehicle is available again, pull/hash live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
