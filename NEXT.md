# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

The HVAC application delta is closed as a Fragrance discriminator. Framework config flow is mapped through `CarConfigUtil` / `EolConfig`, and VDBus identifies event source `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.

Exact class ownership localization currently shows:

- 88 APKs across `system/app`, `system/priv-app`, `product/app`, `product/priv-app`: zero exact VehicleDevice owners.
- Expanded scan across `system/framework`, `product/framework`, `system_ext/framework`, and `system_ext` app/priv-app: 102 Java archives checked, zero exact owners.
- `vendor.img` extracted successfully: ~348.9 MB, SHA-256 `b1e7e189033a7d4347b2c730263d535955b783cb48a1a3226b6f0e8c4a9ef283`.
- First vendor scan is inconclusive because traversal reached only one archive: `/app/TimeService/TimeService.apk`.
- Vendor root contains top-level `/vehicle`, but `/vehicle` contains only `/vehicle/etc/svp_tuner_hal_conf.xml` and `/vehicle/etc/vehicle.hardkey.conf`; this path is not the missing Java owner.

## Next step

1. Recursively export the entire vendor filesystem read-only into a normal WSL ext4 directory (not `/mnt/c` or `/mnt/d`, to avoid ownership/permission quirks).
2. Build a complete recursive inventory of `.apk`, `.jar`, `.odex`, `.vdex`, `.oat`, `.apex`, `.rc`, executables and shared libraries.
3. Run exact DEX `class_def` search for `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;` across every discovered APK/JAR.
4. Separately search raw strings in native/preopt/rc candidates for `com.desaysv.ivi.vds.vdev`, `VehicleDevice`, `918905`, and `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE` to locate non-Java ownership/launch wiring if present.
5. If an exact Java owner is found, stop broad scanning and decompile only that owner.
6. If vendor is exhaustively negative, then move to OAT/VDEX/APEX/native/system-service packaging outside vendor; do not return to application APK guessing.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
