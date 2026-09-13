# RU02-HVAC NEXT

## Current objective

Determine whether an external RU02 resource overlay/RRO is the missing T1J Fragrance UI activation mechanism.

## Current state

- Project-config transport through VehicleDevice event 918905 -> VDBus -> CarConfigUtil -> EolConfig -> config50 is closed.
- FragrancePresenter and FragranceModel are closed as hidden availability-gate candidates.
- T1J `bottom_layout.xml` declares `fragrance_btn` `GONE`.
- Exact T1J `BottomLayoutBindingImpl.executeBindings()` contains no `OfflineConfigManager.f()` call and no `fragranceBtn.setVisibility()`; it only assigns the Fragrance button click listener.
- T1J `view/b` uses `OfflineConfigManager.f()` to initialize FragranceDialog, not to expose the main button.
- Therefore the inspected in-APK T1J layout/binding/view path has no `config50 -> fragrance_btn VISIBLE` path.
- T1H is only a control contrast: its layout starts without `GONE` and its generated binding contains both `f()` and an explicit `fragranceBtn.setVisibility(...)`. Do not transfer its generated logic to the T1J vehicle.

## Next step

Perform one read-only static overlay scan:

1. inspect overlay APKs in RU02 `/system`, `/product`, `/system_ext`, and `/vendor` partitions;
2. identify manifests with overlay target package `com.desaysv.svhvac`;
3. for every matching APK, decode resources and check for `bottom_layout`, `fragrance_btn`, visibility values, or relevant replacement resources;
4. inspect overlay configuration files if present for enablement/priority;
5. if no HVAC-targeting overlay exists, promote T1J UI implementation omission/defect to the leading static root-cause finding.

When the vehicle returns:

- reconcile live HVAC/CarInfo hashes;
- run `cmd overlay list --user 0 com.desaysv.svhvac` as a runtime cross-check before final root-cause closure.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H activation assumptions, blind writes, or unsigned HVAC patching.
