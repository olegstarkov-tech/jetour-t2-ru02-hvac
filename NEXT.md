# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

The RU06 vs RU02-labeled HVAC application delta is effectively closed as a Fragrance discriminator: the only confirmed DEX behavior change is ionizer/config91-specific, decoded manifest is semantically identical, and decoded resource changes are localization-only.

The canonical vehicle is temporarily unavailable; live APK hash identity is deferred without blocking offline analysis.

RU02 framework extraction is complete for the first VDBus layer:

- `vdbus.jar` SHA-256 `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`;
- `vdbus_extra.jar` SHA-256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`;
- both decompile cleanly with JADX.

The config update path is mapped:

1. `CarConfigUtil.init()` initializes VDBus.
2. On `ServiceType.VEHICLE_DEVICE` connection it subscribes to event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`), registers `VDNotifyListener`, commits, and calls `EolConfig.loadConfig()`.
3. On event `918905`, payload is decoded through `VDVDeviceConfigStore` into key/value.
4. Key `vehicle.persist.project.ext.configs` causes `EolConfig.updateConfig(Utils.stringToByte(value), null, null, null, null)`.
5. `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`.
6. `VDServiceDef` names the event source as `com.desaysv.ivi.vds.vdev.service.VehicleDevice` in package `com.desaysv.ivi.vds.vdev`.
7. `VehicleService` is a separate HAL-facing service: `com.desaysv.ivi.vds.vehicle.service.VehicleService` under `android.hardware.automotive.vehicle@2.0-service`.

Two strongest APK candidates have been extracted and fingerprinted:

- `/system/priv-app/DesaySVProjectService/DesaySVProjectService.apk`
  - SHA-256 `2a172f9aa1a447df8ad32a733680843bfb5f131ca5c7db8dcbc507db49dd7282`
  - package `com.desaysv.ivi.vds.projection`
  - manifest services: `ProjectionService`, `DesaySVProjectManagerService`
- `/product/app/SVVDSCarStateService/SVVDSCarStateService.apk`
  - SHA-256 `52b3d5f63922031be273746cc3ac553c19a2d49a9508028aa81305c9988c0e1d`
  - package `com.desaysv.ivi.vds.carstate`
  - manifest service: `CarStateService`

Both DEXes contain the raw string `com.desaysv.ivi.vds.vdev.service.VehicleDevice`, but neither manifest declares that service. Therefore raw string search is insufficient to identify the implementation APK. `DesaySVProjectService` also contains shared VDBus symbols (`VDEventVehicleDevice`, `VDVDeviceConfigStore`, `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`), while `SVVDSCarStateService` contains a `VehicleDeviceManager` client class.

## Next step

1. Decompile these two extracted APKs read-only with JADX.
2. Check for the exact source path `sources/com/desaysv/ivi/vds/vdev/service/VehicleDevice.java` and for an actual `class VehicleDevice` definition.
3. If found in one APK, trace event `918905` production and the origin/filtering of `vehicle.persist.project.ext.configs` there.
4. If absent in both, do not infer from raw strings; broaden localization across the remaining RU02 APKs using exact class-definition/source-path search.
5. Follow into `VehicleService` only if the actual VehicleDevice implementation references it.
6. Do not broaden into unrelated framework jars until VehicleDevice ownership is resolved.

When the vehicle is available again, pull/hash live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
