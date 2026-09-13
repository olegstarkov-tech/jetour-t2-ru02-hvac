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

## LIKELY

- The leading remaining static explanation is T1J UI/resource implementation asymmetry: the config/backend/operational Fragrance stack exists, but the T1J entry ships hidden and the inspected T1J binding never exposes it.
- An external RU02 RRO/resource overlay is the last strong alternative that could make the hidden T1J layout entry visible without an in-APK setter.
- If no HVAC-targeting overlay exists in RU02 system/product/system_ext/vendor, the T1J UI implementation omission/defect becomes the leading root-cause finding.

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

## OPEN

- Does RU02 contain a static RRO/resource overlay targeting `com.desaysv.svhvac`?
- If so, does it override `bottom_layout`, `fragrance_btn`, visibility, or a resource used by that layout?
- If not, was the T1J Fragrance entry simply omitted from the visibility binding in this HVAC generation?
- Why did older signed RU05 HVAC also fail on the RU02 system base?
- Which local HVAC/CarInfo artifacts are byte-exact with the live canonical vehicle? Live reconciliation remains pending until vehicle access returns.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only firmware/runtime comparison before any write/injection experiment.
