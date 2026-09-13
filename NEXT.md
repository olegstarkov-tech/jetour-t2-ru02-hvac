# RU02-HVAC NEXT

## Current objective

Explain why signed RU05 HVAC still hid Fragrance on the RU02 system base even though its T1J visibility predicate is exactly `CarConfigUtil.getConfig(50)==1`.

## Current state

- Current RU02-generation config transport through VehicleDevice event 918905 -> VDBus -> CarConfigUtil -> EolConfig -> config50 is closed.
- Current FragrancePresenter and FragranceModel are closed as hidden availability-gate candidates.
- Current RU02-generation T1J `bottom_layout.xml` declares `fragrance_btn` `GONE`.
- Exact current T1J `BottomLayoutBindingImpl.executeBindings()` contains no current-generation `OfflineConfigManager.f()` call and no `fragranceBtn.setVisibility()`; it only assigns the Fragrance button click listener.
- Current T1J `view/b` uses current-generation `OfflineConfigManager.f()` to initialize FragranceDialog, not to expose the main button.
- No scanned static RU02 overlay targets `com.desaysv.svhvac`.
- Therefore the leading explanation for the current RU02-generation HVAC remains a T1J UI implementation omission/regression.

RU05 comparison is now fully resolved at the UI predicate level:

- exact RU05 HVAC hash is `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`;
- RU05 T1J `bottom_layout.xml` does not set `fragrance_btn` `GONE`;
- RU05 `BottomLayoutBindingImpl` explicitly controls `fragranceBtn.setVisibility(...)`;
- generated flow is `OfflineConfigManager.e() -> (true ? 0 : 4) -> fragranceBtn.setVisibility(...)`;
- RU05 `e()` is definitively `isFragranceExist`;
- exact `e()` body loads `0x32` = 50, calls `CarConfigUtil.getConfig(50)`, and returns true iff result equals 1;
- there is no second T1H/PHEV/market/telematics/project condition;
- therefore the old APK should show the button whenever the config implementation it actually uses reports config50=1.

## Next step

Do one narrow read-only RU05 class-resolution/config-source audit:

1. inspect the exact RU05 manifest for `uses-library` / shared-library declarations related to VDBus/carconfig;
2. inspect the RU05 APK-embedded `CarConfigUtil` and `EolConfig` classes, especially `getConfig`, initialization, load/update path, property/event keys and VDBus registration;
3. compare only those methods/keys with exact RU02 `/system/framework/vdbus_extra.jar`;
4. determine whether RU05 on RU02 would resolve its embedded classes or a parent/shared-system implementation;
5. identify the smallest concrete mismatch capable of making RU05 `getConfig(50)` return non-1 while current RU02 sees config50=1.

Do not rerun broad APK comparisons or backend scans.

When the vehicle returns:

- reconcile live HVAC/CarInfo hashes;
- run `cmd overlay list --user 0 com.desaysv.svhvac` as runtime/dynamic-overlay cross-check;
- if needed, run focused RU05 class/predicate logging only after static class resolution is understood.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H activation assumptions, blind writes, or unsigned HVAC patching.
