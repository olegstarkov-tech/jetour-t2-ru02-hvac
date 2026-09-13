# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Current state

The HVAC application delta is closed as a Fragrance discriminator. Framework config flow is mapped through `CarConfigUtil` / `EolConfig`, and VDBus identifies event source `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.

Exact class ownership localization currently shows:

- 88 APKs across `system/app`, `system/priv-app`, `product/app`, `product/priv-app`: zero exact VehicleDevice owners.
- Expanded scan across `system/framework`, `product/framework`, `system_ext/framework`, and `system_ext` app/priv-app: 102 Java archives checked, zero exact owners.
- `vendor.img` extracted successfully: ~348.9 MB, SHA-256 `b1e7e189033a7d4347b2c730263d535955b783cb48a1a3226b6f0e8c4a9ef283`.
- First vendor scan is inconclusive because only 1 archive was actually reached by the traversal logic. Do NOT mark vendor negative from that run.

## Next step

1. Inspect the full vendor root/directory layout from the already-generated `RU02_VehicleDevice_vendor_owner_scan.txt` and enumerate actual locations of APK/JAR/preopt files.
2. Fix vendor traversal to cover every real APK/JAR location rather than assuming `/app`, `/priv-app`, `/framework` only.
3. Repeat exact DEX `class_def` search for `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;` across all discovered vendor Java containers.
4. Also inventory nearby `.odex`, `.vdex`, `.oat`, and `.apex` files so escalation is evidence-driven if ordinary vendor archives remain negative.
5. Only after exhaustive vendor coverage should investigation move to OAT/VDEX/APEX/native/system-service packaging.
6. If an exact owner is found, stop broad scanning and decompile only that owner to trace event `918905` and `vehicle.persist.project.ext.configs` production/filtering.

When the vehicle becomes available again, pull/hash live CarInfo/HVAC APKs and reconcile labels before any package replacement experiment.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- Prefer read-only extraction/decompile/static comparison.
- No blind VDBus/property/config writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
