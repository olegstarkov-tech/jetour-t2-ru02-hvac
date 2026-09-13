# RU02-HVAC NEXT

## Current objective

Identify the real RU02 Fragrance UI activation mechanism that keeps the OEM entry hidden despite config50=1.

## Current state

- Project-config transport through VehicleDevice event 918905 -> VDBus -> CarConfigUtil -> EolConfig -> config50 is statically closed end-to-end.
- Current static audit is against local `SVHvac_RU02_2026.apk` SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.
- FragrancePresenter and FragranceModel are closed as hidden availability-gate candidates; the actual model consumes only IDs `{58,59,60,61,62,71,92,93,94}`.
- T1J `res/layout/bottom_layout.xml` declares `fragrance_btn` `GONE`.
- T1H `res/layout/t1h_bottom_layout_new.xml` contains `fragrance_btn` without an initial `GONE` visibility.
- Exact smali call-site scan found `OfflineConfigManager.f()` in `T1HHvacActivity`, `T1hBottomLayoutNewBindingImpl`, and T1J `view/b` only.
- In T1J `view/b`, `f()` only gates FragranceDialog initialization in the inspected block; it does not set the main button visibility there.
- No exact `f()` call-site was found in T1J `BottomLayoutBindingImpl`.
- The empty audit section searching literal `fragrance_btn` inside smali is not conclusive because generated smali may use `fragranceBtn` fields or numeric resource IDs.

## Next step

Do one exact binding-level comparison:

1. dump the complete `executeBindings()` logic around `OfflineConfigManager.f()` in `T1hBottomLayoutNewBindingImpl.smali` and identify the exact target view/visibility operation;
2. inspect T1J `BottomLayoutBindingImpl.smali` for all `fragranceBtn` field accesses, the numeric `fragrance_btn` resource ID, and every `View.setVisibility()` call;
3. determine whether T1J has any equivalent `config50 -> fragranceBtn visibility` path;
4. only if T1J has no in-APK visibility activation path, scan RU02 RRO/overlay packages targeting `com.desaysv.svhvac`.

Do not return to broad backend-ID guessing, VehicleDevice transport, OAT/VDEX hunting, blind writes, unsigned HVAC patching, or treating T1H as the active vehicle branch.

When the vehicle becomes available again, reconcile live HVAC/CarInfo hashes and query active overlays as a runtime cross-check.
