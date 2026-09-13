# RU02-HVAC NEXT

## Current objective

Test whether the same T1J Fragrance UI omission already exists in the older signed RU05 HVAC that also failed on the RU02 system base.

## Current state

- Project-config transport through VehicleDevice event 918905 -> VDBus -> CarConfigUtil -> EolConfig -> config50 is closed.
- FragrancePresenter and FragranceModel are closed as hidden availability-gate candidates.
- T1J `bottom_layout.xml` declares `fragrance_btn` `GONE`.
- Exact T1J `BottomLayoutBindingImpl.executeBindings()` contains no `OfflineConfigManager.f()` call and no `fragranceBtn.setVisibility()`; it only assigns the Fragrance button click listener.
- T1J `view/b` uses `OfflineConfigManager.f()` to initialize FragranceDialog, not to expose the main button.
- Therefore the inspected in-APK T1J layout/binding/view path has no `config50 -> fragrance_btn VISIBLE` path.
- T1H remains control-only: its layout/binding differs and must not be treated as the active vehicle branch.
- Static overlay scan across available RU02 system/product/system_ext/vendor partition images found only 2 overlay APKs, both targeting package `android`.
- No scanned static overlay targets `com.desaysv.svhvac`; the static RRO explanation is closed.
- The leading static explanation is now a T1J UI implementation omission/asymmetry.

## Next step

Perform one deterministic RU05-vs-RU02 T1J UI comparison:

1. verify the known RU05 HVAC artifact hash before analysis;
2. decode RU05 `res/layout/bottom_layout.xml` and record initial `fragrance_btn` visibility;
3. inspect RU05 `BottomLayoutBindingImpl.executeBindings()` for all `fragranceBtn` field accesses, `setVisibility()` calls, and `OfflineConfigManager.f()` call-sites;
4. inspect RU05 T1J `view/b` for all Fragrance-existence predicate call-sites and any direct main-button visibility operation;
5. compare those exact paths with the proven RU02 T1J behavior;
6. if RU05 has the same omission, promote the UI omission to a cross-generation explanation for both current behavior and the failed old-HVAC A/B test;
7. if RU05 has a valid T1J visibility path, pivot specifically to why that signed RU05 APK still failed on the RU02 system base.

When the vehicle returns:

- reconcile live HVAC/CarInfo hashes;
- run `cmd overlay list --user 0 com.desaysv.svhvac` as a runtime/dynamic-overlay cross-check.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H activation assumptions, blind writes, or unsigned HVAC patching.
