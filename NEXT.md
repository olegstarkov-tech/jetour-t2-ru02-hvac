# RU02-HVAC NEXT

## Current objective

Resolve the exact RU05 T1J Fragrance predicate implementation and explain why signed RU05 HVAC still hid Fragrance on the RU02 system base.

## Current state

- Current RU02-generation config transport through VehicleDevice event 918905 -> VDBus -> CarConfigUtil -> EolConfig -> config50 is closed.
- Current FragrancePresenter and FragranceModel are closed as hidden availability-gate candidates.
- Current RU02-generation T1J `bottom_layout.xml` declares `fragrance_btn` `GONE`.
- Exact current T1J `BottomLayoutBindingImpl.executeBindings()` contains no current-generation `OfflineConfigManager.f()` call and no `fragranceBtn.setVisibility()`; it only assigns the Fragrance button click listener.
- Current T1J `view/b` uses current-generation `OfflineConfigManager.f()` to initialize FragranceDialog, not to expose the main button.
- No scanned static RU02 overlay targets `com.desaysv.svhvac`.
- Therefore the leading explanation for the current RU02-generation HVAC remains a T1J UI implementation omission/regression.

RU05 comparison now proves a different path:

- exact RU05 HVAC hash is `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`;
- RU05 T1J `bottom_layout.xml` does not set `fragrance_btn` `GONE`;
- RU05 `BottomLayoutBindingImpl` explicitly controls `fragranceBtn.setVisibility(...)`;
- exact generated data flow is `OfflineConfigManager.e() -> v14 -> (true ? 0 : 4) -> v13 -> fragranceBtn.setVisibility(v13)`;
- RU05 `OfflineConfigManager.e()` is definitively `isFragranceExist` by its exact log string;
- RU05 `f()` is `isFrontWindHeatExist`, confirming method-letter mapping differs from RU02-generation;
- therefore RU05 does not share the current T1J UI omission and its failed A/B test is a separate predicate/config-resolution problem.

## Next step

Use the already-generated `/mnt/c/Users/olegs/Desktop/HVAC_WSL/RU05_OfflineConfigManager_exact.txt` only:

1. print just the `METHOD e()` block;
2. capture the exact constant passed to `CarConfigUtil.getConfig(...)`;
3. capture any extra condition such as project/PHEV checks;
4. after that, trace only the RU05 config source/implementation needed to explain why the predicate could evaluate false on RU02.

Do not rerun broad APK comparisons or decompile work.

When the vehicle returns:

- reconcile live HVAC/CarInfo hashes;
- run `cmd overlay list --user 0 com.desaysv.svhvac` as runtime/dynamic-overlay cross-check;
- if static analysis requires it, run focused RU05-vs-current predicate logging only after the exact source path is known.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H activation assumptions, blind writes, or unsigned HVAC patching.
