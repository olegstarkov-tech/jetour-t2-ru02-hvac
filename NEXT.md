# RU02-HVAC NEXT

## Current objective

Resolve the exact RU05 T1J Fragrance visibility predicate and explain why the signed RU05 HVAC still hid Fragrance on the RU02 system base.

## Current state

- Current RU02-generation config transport through VehicleDevice event 918905 -> VDBus -> CarConfigUtil -> EolConfig -> config50 is closed.
- Current FragrancePresenter and FragranceModel are closed as hidden availability-gate candidates.
- Current RU02-generation T1J `bottom_layout.xml` declares `fragrance_btn` `GONE`.
- Exact current T1J `BottomLayoutBindingImpl.executeBindings()` contains no current-generation `OfflineConfigManager.f()` call and no `fragranceBtn.setVisibility()`; it only assigns the Fragrance button click listener.
- Current T1J `view/b` uses current-generation `OfflineConfigManager.f()` to initialize FragranceDialog, not to expose the main button.
- No scanned static RU02 overlay targets `com.desaysv.svhvac`.
- Therefore the leading explanation for the current RU02-generation HVAC is a T1J UI implementation omission/regression.

RU05 comparison changed the A/B interpretation:

- exact RU05 HVAC hash is `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`;
- RU05 T1J `bottom_layout.xml` does not set `fragrance_btn` `GONE`;
- RU05 `BottomLayoutBindingImpl` explicitly controls `fragranceBtn.setVisibility(...)`;
- exact generated data flow is `OfflineConfigManager.e() -> v14 -> (true ? 0 : 4) -> v13 -> fragranceBtn.setVisibility(v13)`;
- therefore RU05 has a real T1J Fragrance visibility path and does not share the current RU02-generation omission;
- RU05 method letters cannot be mapped from RU02: RU05 `OfflineConfigManager.f()` is used for `ivFrontWindHeat`, so RU05 `e()` must be resolved directly.

## Next step

Do one narrow read-only RU05 OfflineConfigManager trace:

1. locate exact decoded RU05 `com/desaysv/svhvac/h/a.smali`;
2. dump methods `c()` through `l()` including log strings and `CarConfigUtil`/EOL calls;
3. identify the exact body of RU05 `e()` and prove whether it is `isFragranceExist` / config50;
4. record the RU05 source class/API used to read that config;
5. only after that, investigate why this RU05 predicate could evaluate false on the RU02 system base.

When the vehicle returns:

- reconcile live HVAC/CarInfo hashes;
- run `cmd overlay list --user 0 com.desaysv.svhvac` as runtime/dynamic-overlay cross-check;
- if needed, run focused RU05-vs-current predicate logging only after the static source path is known.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H activation assumptions, blind writes, or unsigned APK patching.
