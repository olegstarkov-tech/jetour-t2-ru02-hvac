# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- `ID_CAR_CONFIG_FRAGRANCE = 50`; `EolConfig` maps it from `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- Current RU02-generation HVAC Fragrance predicate is `getConfig(50)==1 && !isT1H_PHEV()`.
- VehicleDevice/event-918905 transport is statically closed end-to-end to `CarConfigUtil -> EolConfig -> config50`; no country/project/market/telematics suppression was found in the traced transport.
- Current static HVAC artifact is `SVHvac_RU02_2026.apk`, SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.
- FragrancePresenter contains no visibility/capability condition and directly delegates to FragranceModel.
- FragranceModel uses only module `327690` with IDs `{58,59,60,61,62,71,92,93,94}` = type1/type2/type3, power, level, position/channel, remain1/remain2/remain3.
- FragranceModel initial reads use the same nine IDs; writes are only IDs 61, 62, 71.
- IDs 64 (`AC_FRAGRANCE_DISPLAY`), 66 (`AC_FRAGRANCE_WARNING`), 76 and 88 are not part of the real FragranceModel subscription/read/write path.
- RU02-generation T1J `res/layout/bottom_layout.xml` declares `fragrance_btn` as `GONE`.
- Exact RU02-generation T1J `BottomLayoutBindingImpl.executeBindings()` inspection proves `fragranceBtn` is assigned an `OnClickListener`, but no `setVisibility()` is called on `fragranceBtn`.
- Exact RU02-generation T1J binding contains no `OfflineConfigManager.f()` call-site.
- RU02-generation T1J `view/b` does call `OfflineConfigManager.f()`, but the inspected branch only initializes FragranceDialog and does not change main-button visibility.
- Fragrance operational callbacks update selected/type/level/remain/position state but do not make the main current-generation T1J button visible.
- Therefore the decoded current T1J base-layout/generated-binding/inspected-view path contains no `config50 -> fragrance_btn VISIBLE` activation path.
- T1H control branch differs: `t1h_bottom_layout_new.xml` has no initial `GONE`, and `T1hBottomLayoutNewBindingImpl` contains both current-generation `OfflineConfigManager.f()` and explicit `fragranceBtn.setVisibility(...)`.
- T1H generated visibility logic is not a simple transferable activation formula for T1J; nearby generated branches also consume unrelated config predicates. Current-generation `OfflineConfigManager.c()` is config104 / `isBehindSeatHeatExist`.
- Static overlay scan across available RU02 `system`, `product`, `system_ext`, and `vendor` partition images found exactly 2 overlay APKs in the standard overlay directories.
- Both scanned overlay APK manifests target package `android`.
- No scanned static overlay targets `com.desaysv.svhvac`; no static RU02 RRO was found that overrides HVAC `bottom_layout`, `fragrance_btn`, or its visibility.
- Exact older signed HVAC artifact `SVHvac_RU05.apk` SHA-256 is `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.
- RU05 T1J `bottom_layout.xml` contains `fragrance_btn` without an initial `GONE` visibility.
- RU05 T1J `BottomLayoutBindingImpl.executeBindings()` explicitly calls `fragranceBtn.setVisibility(...)`.
- Exact RU05 generated-binding register flow is `OfflineConfigManager.e() -> v14 -> (v14 ? 0 : 4) -> v13 -> fragranceBtn.setVisibility(v13)`.
- Exact RU05 OfflineConfigManager mapping proves `e()` is `isFragranceExist`.
- Exact RU05 `e()` smali calls `CarConfigUtil.getConfig(0x32)` = `getConfig(50)`, compares only to `1`, logs `isFragranceExist`, and returns that boolean.
- RU05 `e()` contains no second T1H/PHEV/market/telematics/project condition.
- Therefore RU05 T1J Fragrance visibility is exactly `CarConfigUtil.getConfig(50)==1 ? VISIBLE(0) : INVISIBLE(4)`.

## LIKELY

- For the current RU02-generation HVAC, the leading static root-cause explanation remains a T1J UI implementation omission/regression: config/backend/model exists and config50 reaches HVAC, but the T1J entry ships `GONE` and the generated binding never exposes it.
- Because the static HVAC-targeting RRO alternative is closed, the current-generation UI omission is substantially stronger than prior backend/display-ID hypotheses.
- The signed RU05 failure on the RU02 system base is a separate compatibility/config-resolution problem. Since RU05 has a valid visibility path controlled only by `getConfig(50)==1`, the implementation actually used by RU05 on RU02 apparently did not present config50 as 1 at binding evaluation time. Exact class resolution/source path remains to be proven.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit in the current RU02-generation path.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC APK alone solves the problem.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` is the visibility gate.
- Current-generation `OfflineConfigManager.h()` is Fragrance; it is ionizer/config91.
- Hidden Fragrance suppression is in current CarConfigUtil, VehicleDevice/event-918905 transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- The real current FragranceModel consumes 64/66/76/88 as an extra availability gate.
- Current RU02-generation T1J `BottomLayoutBindingImpl` contains a hidden `OfflineConfigManager.f()` or `fragranceBtn.setVisibility()` activation path.
- A static RU02 overlay in the scanned system/product/system_ext/vendor overlay directories targets `com.desaysv.svhvac` and unhides the Fragrance entry.
- RU05 T1J has the same `fragrance_btn=GONE` plus missing visibility-binding omission as RU02-generation HVAC.
- RU05 uses another Fragrance config ID instead of 50.
- RU05 Fragrance visibility includes a second T1H/PHEV/market/telematics/project gate after config50.

## OPEN

- Which `CarConfigUtil/EolConfig` implementation did RU05 actually resolve/use when installed on the RU02 system base: its embedded RU05 copy or a parent/shared system copy?
- How does the RU05-generation config implementation populate config1/config50, and why could it return a non-1 value while the current RU02 path sees byte12=`C5` / config50=1?
- Is any runtime/dynamic overlay active on the live vehicle outside the static partition images? Pending `cmd overlay list --user 0 com.desaysv.svhvac` when vehicle access returns.
- Which local HVAC/CarInfo artifacts are byte-exact with the live canonical vehicle? Live reconciliation remains pending until vehicle access returns.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only firmware/runtime comparison before any write/injection experiment.
