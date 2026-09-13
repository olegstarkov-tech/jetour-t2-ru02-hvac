# RU02-HVAC CURRENT

Status: active
Canonical branch: `main`
Recovery marker: `RU02-HVAC-CONTINUE`

## Bench

Jetour T2 / T1J, official RU, Desay SV 8155, dealer firmware 00.00.02, telematics present.

## Mission

Determine why OEM Fragrance remains hidden despite enabled vehicle config and identify the real RU02 activation mechanism.

## PROVEN

### Config and transport

- Engineering Fragrance changes config1 byte12 `0x85 -> 0xC5`; bit6 is Fragrance.
- `ID_CAR_CONFIG_FRAGRANCE = 50`; `EolConfig` maps it from `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- Current RU02-generation HVAC `OfflineConfigManager.f()` is exactly `getConfig(50)==1 && !isT1H_PHEV()`; canonical car is T1J.
- Native VehicleDevice/event-918905 transport is closed end-to-end and contains no observed country/project/market/telematics suppression.

### HVAC operational path

Static audit artifact: `SVHvac_RU02_2026.apk`, SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.

- FragrancePresenter is a thin delegate with no visibility/capability gate.
- `ModelFactory.b()` returns `FragranceModel`.
- FragranceModel uses only module `327690`, IDs `{58,59,60,61,62,71,92,93,94}`: type1/2/3, power, level, position/channel, remain1/2/3.
- Initial reads use the same set; writes are only 61/62/71.
- IDs 64/66/76/88 are not in the real FragranceModel subscription/read/write path.
- Therefore presenter/model contains ordinary operational state only, not a second display/availability gate.

### RU02-generation T1J UI visibility path — CLOSED inside current APK

- T1J `res/layout/bottom_layout.xml` declares `fragrance_btn` with `android:visibility="gone"`.
- Exact `BottomLayoutBindingImpl.executeBindings()` inspection proves T1J `fragranceBtn` receives an `OnClickListener` but has **no `setVisibility()` call** in the generated binding.
- T1J `BottomLayoutBindingImpl` has no `OfflineConfigManager.f()` call-site.
- T1J `view/b.smali` does call `OfflineConfigManager.f()`, but the inspected call gates FragranceDialog initialization only; it does not change main-button visibility.
- Existing Fragrance callbacks change selected/operational state, not main-button visibility.
- Therefore, within the decoded T1J base-layout + generated-binding + inspected view path of this APK, there is no `config50 -> fragrance_btn VISIBLE` activation path.

### T1H control contrast only

- T1H is not the canonical vehicle branch.
- `t1h_bottom_layout_new.xml` contains `fragrance_btn` without initial `GONE`.
- `T1hBottomLayoutNewBindingImpl` does call current-generation `OfflineConfigManager.f()` and contains an explicit `fragranceBtn.setVisibility(...)` operation.
- Generated T1H logic is not a simple transferable activation formula for T1J; nearby generated branches consume unrelated predicates too. Current-generation `OfflineConfigManager.c()` is config104 / `isBehindSeatHeatExist`.

### Static RU02 overlay scan — CLOSED

Read-only scan used available exact RU02 partition images: `system.img`, `product.img`, `system_ext.img`, `vendor.img`.

- Standard overlay and overlay-config directories were extracted/scanned.
- Exactly **2 overlay APKs** were found in scope.
- Both manifests have `android:targetPackage="android"`.
- No static overlay APK targeting `com.desaysv.svhvac` was found.
- Therefore no scanned RU02 static RRO overrides HVAC `bottom_layout`, `fragrance_btn`, or its visibility.

A runtime/dynamic overlay outside these static partition images remains a live cross-check only until the vehicle returns.

### RU05 T1J UI comparison — IMPORTANT CONTRAST

Exact old signed HVAC artifact: `SVHvac_RU05.apk`, SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.

- RU05 T1J `res/layout/bottom_layout.xml` contains `fragrance_btn` **without** an initial `GONE` visibility.
- RU05 `BottomLayoutBindingImpl.executeBindings()` contains an explicit `fragranceBtn.setVisibility(...)`.
- Exact register flow in the generated binding is:
  - `OfflineConfigManager.e()` result -> `v14`;
  - `v14 ? 0 : 4` -> `v13`;
  - `fragranceBtn.setVisibility(v13)`.
- Thus RU05 has a real T1J feature-predicate-driven Fragrance visibility path: predicate true -> `VISIBLE(0)`, false -> `INVISIBLE(4)`.
- Exact RU05 `OfflineConfigManager` method/log mapping is now proven from smali:
  - `c()` = `isDiverSeatHeatExist`;
  - `d()` = `isDiverSeatVentilationExist`;
  - `e()` = `isFragranceExist`;
  - `f()` = `isFrontWindHeatExist`;
  - `g()` = `isIonExist`;
  - `h()` = `isPM25Exist`;
  - `i()` = `isPassengerSeatHeatExist`;
  - `j()` = `isPassengerSeatVentilationExist`;
  - `k()` = `isULTravioletExist`;
  - `l()` = `isWheelHeatExist`.
- Therefore RU05 Fragrance visibility is definitively controlled by `OfflineConfigManager.e()` / `isFragranceExist`.
- The exact config ID/extra condition inside RU05 `e()` is not yet recorded because the quick grep omitted the preceding `const` line; do not assume it by method letter or from RU02 until the exact method body is read.

Therefore the hypothesis that RU05 failed because it had the **same** T1J `GONE` + missing-binding omission is DISPROVEN.

## DISPROVEN / closed without new evidence

- Wrong Engineering bit or wrong config ID in the current RU02-generation path.
- Fragrance removed from current HVAC.
- Old HVAC APK alone solves it.
- T1H is the active branch for this car.
- `a2()` controls visibility.
- Hidden gate in current CarConfigUtil, VehicleDevice/event-918905 transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- The current RU02-generation T1J `BottomLayoutBindingImpl` contains a hidden config50-to-fragrance visibility setter.
- A scanned RU02 static overlay in system/product/system_ext/vendor targets `com.desaysv.svhvac` and fixes/unhides `fragrance_btn`.
- RU05 T1J has the same `fragrance_btn=GONE` + missing visibility-binding omission as RU02-generation HVAC.
- RU05 `OfflineConfigManager.f()` is the Fragrance predicate; exact smali proves RU05 Fragrance is `e()` and `f()` is front-wind-heat.

## Leading explanation

For the current RU02-generation HVAC, the strongest static explanation remains a **T1J UI implementation omission/regression**:

- Fragrance config50 is valid and reaches current HVAC;
- Fragrance operational model/backend exists;
- current T1J layout ships the entry as `GONE`;
- current T1J generated binding never makes it visible;
- no static RU02 HVAC-targeting RRO was found to compensate.

However, this explanation does **not** explain the older signed RU05 A/B failure. RU05 has a valid T1J visibility path controlled by its own `OfflineConfigManager.e()` / `isFragranceExist`. The RU05 failure is therefore a separate compatibility/config-resolution question.

## Current open question

What exact config ID and extra conditions does RU05 `OfflineConfigManager.e()` / `isFragranceExist` use, and why did that predicate remain false when signed RU05 HVAC ran on the RU02 system base?

## Next step

One minimal read-only extraction from the already-generated `RU05_OfflineConfigManager_exact.txt`:

1. print only the `METHOD e()` block;
2. capture its exact `const` passed to `CarConfigUtil.getConfig(...)` and any second condition/project check;
3. then trace only that RU05 config source/implementation on the RU02 system base.

When vehicle access returns, perform pending runtime cross-checks: live HVAC/CarInfo hashes and `cmd overlay list --user 0 com.desaysv.svhvac`.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H activation assumptions, blind writes, or unsigned APK patching.
