# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- `ID_CAR_CONFIG_FRAGRANCE = 50`; `EolConfig` maps it from `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- Current HVAC Fragrance predicate is `getConfig(50)==1 && !isT1H_PHEV()`.
- VehicleDevice/event-918905 transport is statically closed end-to-end to `CarConfigUtil -> EolConfig -> config50`; no country/project/market/telematics suppression was found in the traced transport.
- Current static HVAC artifact is `SVHvac_RU02_2026.apk`, SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.
- FragrancePresenter contains no visibility/capability condition and directly delegates to FragranceModel.
- FragranceModel uses only module `327690` with IDs `{58,59,60,61,62,71,92,93,94}` = type1/type2/type3, power, level, position/channel, remain1/remain2/remain3.
- FragranceModel initial reads use the same nine IDs; writes are only IDs 61, 62, 71.
- IDs 64 (`AC_FRAGRANCE_DISPLAY`), 66 (`AC_FRAGRANCE_WARNING`), 76 and 88 are not part of the real FragranceModel subscription/read/write path.
- T1J `res/layout/bottom_layout.xml` declares `fragrance_btn` as `GONE`.
- Exact T1J `BottomLayoutBindingImpl.executeBindings()` inspection proves `fragranceBtn` is assigned an `OnClickListener`, but no `setVisibility()` is called on `fragranceBtn`.
- Exact T1J binding contains no `OfflineConfigManager.f()` call-site.
- T1J `view/b` does call `OfflineConfigManager.f()`, but the inspected branch only initializes FragranceDialog and does not change main-button visibility.
- Fragrance operational callbacks update selected/type/level/remain/position state but do not make the main T1J button visible.
- Therefore the decoded T1J base-layout/generated-binding/inspected-view path contains no `config50 -> fragrance_btn VISIBLE` activation path.
- T1H control branch differs: `t1h_bottom_layout_new.xml` has no initial `GONE`, and `T1hBottomLayoutNewBindingImpl` contains both `OfflineConfigManager.f()` and explicit `fragranceBtn.setVisibility(...)`.
- T1H generated visibility logic is not a simple transferable activation formula for T1J; nearby generated branches also consume unrelated config predicates. `OfflineConfigManager.c()` is config104 / `isBehindSeatHeatExist`.
- Static overlay scan across available RU02 `system`, `product`, `system_ext`, and `vendor` partition images found exactly 2 overlay APKs in the standard overlay directories.
- Both scanned overlay APK manifests target package `android`.
- No scanned static overlay targets `com.desaysv.svhvac`; no static RU02 RRO was found that overrides HVAC `bottom_layout`, `fragrance_btn`, or its visibility.

## LIKELY

- The leading static root-cause explanation is a T1J UI implementation omission/asymmetry: Fragrance config/backend/model exists and config50 reaches HVAC, but the T1J entry ships `GONE` and the T1J generated binding does not expose it.
- Because the static HVAC-targeting RRO alternative is now closed, this omission is substantially stronger than the previous backend/display-ID hypotheses.
- Final root-cause closure should still reconcile the older signed RU05 HVAC A/B result and later confirm that no runtime/dynamic overlay is active on the live vehicle.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC APK alone solves the problem.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` is the visibility gate.
- `OfflineConfigManager.h()` is Fragrance; it is ionizer/config91.
- Hidden Fragrance suppression is in CarConfigUtil, VehicleDevice/event-918905 transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- The real FragranceModel consumes 64/66/76/88 as an extra availability gate.
- T1J `BottomLayoutBindingImpl` contains a hidden `OfflineConfigManager.f()` or `fragranceBtn.setVisibility()` activation path.
- A static RU02 overlay in the scanned system/product/system_ext/vendor overlay directories targets `com.desaysv.svhvac` and unhides the Fragrance entry.

## OPEN

- Did RU05 T1J HVAC already contain the same `fragrance_btn=GONE` + missing binding visibility path?
- If RU05 has a real T1J visibility path, why did that signed old HVAC still fail on the RU02 system base?
- Is any runtime/dynamic overlay active on the live vehicle outside the static partition images? Pending `cmd overlay list --user 0 com.desaysv.svhvac` when vehicle access returns.
- Which local HVAC/CarInfo artifacts are byte-exact with the live canonical vehicle? Live reconciliation remains pending until vehicle access returns.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only firmware/runtime comparison before any write/injection experiment.
