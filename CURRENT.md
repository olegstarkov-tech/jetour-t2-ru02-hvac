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
- Generated T1H logic is not a simple `f()==true -> VISIBLE`: its dirty-flag/register flow maps values through visibility constants and also involves other config predicates. Do not use T1H as the activation formula for T1J.
- `OfflineConfigManager.c()` is `isBehindSeatHeatExist` / config104, confirming some nearby T1H branches concern unrelated rear-seat configuration.

## DISPROVEN / closed without new evidence

- Wrong Engineering bit or wrong config ID.
- Fragrance removed from current HVAC.
- Old HVAC APK alone solves it.
- T1H is the active branch for this car.
- `a2()` controls visibility.
- Hidden gate in CarConfigUtil, VehicleDevice/event-918905 transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- The current T1J `BottomLayoutBindingImpl` contains a hidden config50-to-fragrance visibility setter.

## Current open question

Given that the current T1J APK ships `fragrance_btn=GONE` and its generated binding has no visibility activation path, is an **external RU02 resource overlay/RRO** expected to replace or override the HVAC layout/resource at runtime?

If no such overlay exists, the strongest remaining static explanation is a T1J UI implementation omission/defect in this HVAC generation: the feature/config/backend path exists, but the T1J entry is never made visible.

## Next step

Read-only scan of RU02 static overlays only:

1. inspect `/system`, `/product`, `/system_ext`, `/vendor` overlay APK manifests for target package `com.desaysv.svhvac`;
2. for any matching overlay, decode its resources and check whether it overrides `bottom_layout` / `fragrance_btn` or relevant visibility resources;
3. if no HVAC-targeting overlay exists, record T1J UI implementation omission/defect as the leading root-cause finding;
4. when the car returns, cross-check active overlays with `cmd overlay list --user 0 com.desaysv.svhvac` and reconcile live HVAC hash before finalizing.

Do not return to backend-ID guessing, VehicleDevice transport, broad APK guessing, blind writes, or unsigned APK patching.
