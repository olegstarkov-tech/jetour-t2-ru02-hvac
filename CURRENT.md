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
- `ID_CAR_CONFIG_FRAGRANCE = 50` and `EolConfig` maps it from `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- `OfflineConfigManager.f()` is exactly `getConfig(50)==1 && !isT1H_PHEV()`; canonical car is T1J.
- Native VehicleDevice/event-918905 transport is closed end-to-end: changed project config key/value is published as event 918905 and reaches `VDVDeviceConfigStore -> CarConfigUtil -> EolConfig -> config50`.
- No country/project/market/telematics filter was found in that traced transport path.

### HVAC / Fragrance operational path

Static audit artifact: `SVHvac_RU02_2026.apk`, SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.

- HVAC contains Fragrance code/resources/dialog/UI.
- Older signed RU05 HVAC ran on RU02 system but did not restore Fragrance.
- `FragrancePresenter` (`b.a.d.a.b.x0`) is a thin presenter with no visibility/capability condition.
- `ModelFactory.b()` returns `b.a.b.a.c.b` = `FragranceModel`.
- `FragranceModel` listens only to module `327690` and IDs `{58,59,60,61,62,71,92,93,94}`.
- Mapping: 58/59/60 type1/2/3; 61 power; 62 level; 71 position/channel; 92/93/94 remain1/2/3.
- Initial reads use the same set; writes are only 61, 62, 71.
- IDs 64 (`AC_FRAGRANCE_DISPLAY`), 66 (`AC_FRAGRANCE_WARNING`), 76 and 88 are not in the actual FragranceModel subscription/read/write path.
- Therefore presenter/model contains operational state only, not a second display/availability gate.

### UI visibility asymmetry — current focus

- T1J `res/layout/bottom_layout.xml` declares `fragrance_btn` with `android:visibility="gone"`.
- T1H `res/layout/t1h_bottom_layout_new.xml` contains `fragrance_btn` without an initial `GONE` visibility.
- Exact all-smali call-site scan for `OfflineConfigManager.f()` found only:
  - `T1HHvacActivity.smali`;
  - `T1hBottomLayoutNewBindingImpl.smali`;
  - T1J `view/b.smali`.
- T1J `view/b.smali` uses `f()` only to conditionally initialize `FragranceDialog` in the inspected block; it does not set main-button visibility there.
- T1H generated binding directly reads `OfflineConfigManager.f()` inside `executeBindings()`.
- No exact `OfflineConfigManager.f()` call-site was found in T1J `BottomLayoutBindingImpl`.
- Earlier inspection of T1J binding/view code found no proven `fragranceBtn -> VISIBLE` path; `a2(fragranceBtn,z)` controls enabled/clickable only.
- The audit section that searched literal `fragrance_btn` in smali is inconclusive because generated smali may reference `fragranceBtn` fields or numeric resource IDs.

## DISPROVEN / closed without new evidence

- Wrong Engineering bit or wrong config ID.
- Fragrance removed from current HVAC.
- Old HVAC APK alone solves it.
- T1H is the active branch for this car.
- `a2()` controls visibility.
- `OfflineConfigManager.h()` is Fragrance; it is ionizer/config91.
- Hidden gate in `CarConfigUtil`, VehicleDevice/event-918905 transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.

## Current open question

Why does the T1J UI ship `fragrance_btn` as `GONE` while the Fragrance-existence predicate is wired into T1H generated binding but is absent from the T1J `BottomLayoutBindingImpl` call-site set?

This T1J/T1H UI asymmetry is PROVEN, but it is not yet the final root cause: an external RRO/resource overlay or another T1J visibility setter may still exist.

## Next step

Do one exact binding comparison:

1. resolve the T1H `OfflineConfigManager.f()` result through `T1hBottomLayoutNewBindingImpl.executeBindings()` to the exact target view and visibility value;
2. inspect T1J `BottomLayoutBindingImpl.smali` for all `fragranceBtn` field accesses, numeric resource-ID references, and all `setVisibility()` calls;
3. prove whether T1J has any equivalent `config50 -> fragranceBtn visibility` path;
4. only if absent, scan RU02 RRO/overlay packages targeting `com.desaysv.svhvac`.

Do not return to backend-ID guessing, VehicleDevice transport, broad APK guessing, blind writes, or unsigned APK patching.
