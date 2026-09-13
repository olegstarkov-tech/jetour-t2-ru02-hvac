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
- `OfflineConfigManager.f()` is exactly `getConfig(50)==1 && !isT1H_PHEV()`; canonical car is T1J.
- Native VehicleDevice/event-918905 transport is closed end-to-end and contains no observed country/project/market/telematics suppression.

### HVAC operational path

Static audit artifact: `SVHvac_RU02_2026.apk`, SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.

- FragrancePresenter is a thin delegate with no visibility/capability gate.
- `ModelFactory.b()` returns `FragranceModel`.
- FragranceModel uses only module `327690`, IDs `{58,59,60,61,62,71,92,93,94}`: type1/2/3, power, level, position/channel, remain1/2/3.
- Initial reads use the same set; writes are only 61/62/71.
- IDs 64/66/76/88 are not in the real FragranceModel subscription/read/write path.
- Therefore presenter/model contains ordinary operational state only, not a second display/availability gate.

### T1J UI visibility path — CLOSED inside current APK

- T1J `res/layout/bottom_layout.xml` declares `fragrance_btn` with `android:visibility="gone"`.
- Exact `BottomLayoutBindingImpl.executeBindings()` inspection proves T1J `fragranceBtn` receives an `OnClickListener` but has **no `setVisibility()` call** in the generated binding.
- T1J `BottomLayoutBindingImpl` has no `OfflineConfigManager.f()` call-site.
- T1J `view/b.smali` does call `OfflineConfigManager.f()`, but the inspected call gates FragranceDialog initialization only; it does not change main-button visibility.
- Existing Fragrance callbacks change selected/operational state, not main-button visibility.
- Therefore, within the decoded T1J base-layout + generated-binding + inspected view path of this APK, there is no `config50 -> fragrance_btn VISIBLE` activation path.

### T1H control contrast only

- T1H is not the canonical vehicle branch.
- `t1h_bottom_layout_new.xml` contains `fragrance_btn` without initial `GONE`.
- `T1hBottomLayoutNewBindingImpl` does call `OfflineConfigManager.f()` and contains an explicit `fragranceBtn.setVisibility(...)` operation.
- Generated T1H logic is not a simple transferable activation formula for T1J; nearby generated branches consume unrelated predicates too. `OfflineConfigManager.c()` is config104 / `isBehindSeatHeatExist`.

### Static RU02 overlay scan — CLOSED

Read-only scan used available exact RU02 partition images:

- `system.img`;
- `product.img`;
- `system_ext.img`;
- `vendor.img`.

Standard overlay and overlay-config directories were extracted/scanned. The scan found exactly **2 overlay APKs** in scope. Both manifests have `android:targetPackage="android"`.

No static overlay APK targeting `com.desaysv.svhvac` was found, and therefore no scanned RU02 static RRO overrides HVAC `bottom_layout`, `fragrance_btn`, or its visibility.

This closes the static system/product/system_ext/vendor RRO explanation for the hidden T1J Fragrance entry. A runtime/dynamic overlay outside these static partition images remains a live cross-check only until the vehicle returns.

## DISPROVEN / closed without new evidence

- Wrong Engineering bit or wrong config ID.
- Fragrance removed from current HVAC.
- Old HVAC APK alone solves it.
- T1H is the active branch for this car.
- `a2()` controls visibility.
- Hidden gate in CarConfigUtil, VehicleDevice/event-918905 transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- The current T1J `BottomLayoutBindingImpl` contains a hidden config50-to-fragrance visibility setter.
- A scanned RU02 static overlay in system/product/system_ext/vendor targets `com.desaysv.svhvac` and fixes/unhides `fragrance_btn`.

## Leading explanation

The strongest static explanation is now a **T1J UI implementation omission/asymmetry** in this HVAC generation:

- Fragrance config50 is valid and reaches HVAC;
- Fragrance operational model/backend exists;
- T1J layout ships the entry as `GONE`;
- T1J generated binding never makes it visible;
- no static RU02 HVAC-targeting RRO was found to compensate for that omission.

This is the leading root-cause finding, but final closure still needs consistency with the older signed RU05 HVAC result and later a live runtime overlay/hash cross-check.

## Current open question

Did the older signed RU05 HVAC that also failed on the RU02 system already contain the same T1J `fragrance_btn=GONE` + missing visibility-binding omission?

If RU05 has the same omission, one UI-generation defect explains both current RU02-labeled HVAC behavior and the failed old-HVAC A/B test. If RU05 has a real T1J visibility path, then the old-HVAC failure requires a separate system-base explanation.

## Next step

One deterministic static RU05-vs-RU02 T1J UI comparison:

1. decode RU05 `bottom_layout.xml` and record `fragrance_btn` initial visibility;
2. inspect RU05 `BottomLayoutBindingImpl.executeBindings()` for `fragranceBtn.setVisibility()` and `OfflineConfigManager.f()`;
3. inspect RU05 T1J `view/b` Fragrance-existence call-sites;
4. compare those exact paths against the now-proven RU02 behavior;
5. classify whether the T1J UI omission predates RU02.

When vehicle access returns, perform only the pending runtime cross-checks: live HVAC/CarInfo hashes and `cmd overlay list --user 0 com.desaysv.svhvac`.

Do not return to backend-ID guessing, VehicleDevice transport, broad APK guessing, blind writes, or unsigned APK patching.
