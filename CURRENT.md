# RU02-HVAC CURRENT

Status: active
Canonical branch: `main`
Recovery marker: `RU02-HVAC-CONTINUE`

## Bench

Jetour T2 / T1J, official RU, Desay SV 8155, dealer firmware 00.00.02, telematics present.

## Mission

Determine why OEM Fragrance remains hidden despite enabled vehicle config and identify the real RU02 activation mechanism.

## PROVEN

### Current RU02-generation config/UI path

- Engineering Fragrance changes config1 byte12 `0x85 -> 0xC5`; delta `0x40` = bit6.
- `ID_CAR_CONFIG_FRAGRANCE = 50`; current RU02 `EolConfig` maps it from `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- Current RU02-generation HVAC `OfflineConfigManager.f()` is exactly `getConfig(50)==1 && !isT1H_PHEV()`; canonical car is T1J.
- Native VehicleDevice/event-918905 transport is closed end-to-end.
- Current T1J `bottom_layout.xml` declares `fragrance_btn` `GONE`.
- Current T1J `BottomLayoutBindingImpl` has no `fragranceBtn.setVisibility(...)` and no Fragrance predicate call; `view/b` uses the predicate only for FragranceDialog initialization.
- No scanned static RU02 overlay targets `com.desaysv.svhvac`.
- Therefore current RU02-generation T1J has no proven in-APK/static-RRO `config50 -> fragrance_btn VISIBLE` path.

### RU05 T1J UI/predicate path

Exact signed RU05 HVAC: `SVHvac_RU05.apk`, SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.

- RU05 T1J `bottom_layout.xml` does not initially hide `fragrance_btn`.
- RU05 `BottomLayoutBindingImpl` explicitly applies `fragranceBtn.setVisibility(...)`.
- Exact flow: `OfflineConfigManager.e() -> (true ? VISIBLE(0) : INVISIBLE(4)) -> fragranceBtn`.
- RU05 `OfflineConfigManager.e()` is exactly `isFragranceExist`.
- Exact `e()` body is only `CarConfigUtil.getConfig(50)==1`; no second T1H/PHEV/market/telematics/project condition exists.

### RU05 embedded config stack and mapping

- RU05 APK packages its own `CarConfigUtil`, `EolConfig`, `ReserveConfigConstants`, VDBus classes and VehicleDevice-event classes.
- RU05 `CarConfigUtil.getConfig(int)` directly calls packaged `EolConfig.getJetourEolConfig(int)`.
- RU05 `getJetourEolConfig(50)` is exactly `(mCarConfig1[12] >> 6) & 1`, identical to RU02.
- RU05 `EolConfig.loadConfig()` requests `vehicle.persist.project.ext.configs` through VDBus `getOnce(0xe0006)` and stores parsed `value` into static `mCarConfig1`.
- Missing event/payload falls back to empty string before parsing.
- RU05 `CarConfigUtil.init()` loads immediately only when VehicleDevice is already connected; otherwise it calls `bindService()` and returns, with `loadConfig()` deferred to `onVDConnected()`.
- Later event `0xe0579` updates config arrays through `EolConfig.updateConfig(...)`.

### RU05 binding refresh behavior — NEW PROVEN

- `HvacApplication.onCreate()` calls `CarConfigUtil.init(applicationContext)` before later HVAC view initialization including `view/b.I0(context)`.
- RU05 manifest has `HvacApplication` and exported `HvacService`; no normal HVAC Activity entry is declared.
- `BottomLayoutBindingImpl.onFieldChange(...)` always returns `false`.
- The bottom binding does not register `EolConfig` or `CarConfigUtil` as observable dependencies.
- `EolConfig.updateConfig()` only replaces static arrays; no direct route to `BottomLayoutBindingImpl.requestRebind()` exists.
- Fragrance predicate is reevaluated when the bottom binding's own dirty flags are set, including `invalidateAll()` and `setHvacContentView(...)`.
- Therefore a late RU05 config load/update does **not automatically** recompute existing Fragrance visibility.

## LIKELY

### Current RU02-generation root cause

Strongest static explanation remains a T1J UI implementation omission/regression.

### RU05-on-RU02 failed A/B test

A startup-order/stale-binding failure is now technically well supported:

1. if VehicleDevice was not yet connected when RU05 `CarConfigUtil.init()` ran, config load was deferred;
2. if the first BottomLayout binding evaluated before `mCarConfig1` was populated, `getConfig(50)` returned false and Fragrance became `INVISIBLE`;
3. later `onVDConnected()`/event 918905 could populate config50 correctly;
4. that config update would not automatically refresh existing bottom-binding visibility.

This exact live sequence remains LIKELY rather than PROVEN because `CarConfigUtil.init()` runs before `view/b.I0()` and VehicleDevice may have connected before first binding evaluation.

## DISPROVEN / closed without new evidence

- Wrong Engineering bit or wrong current config ID.
- Fragrance removed from current HVAC.
- Old HVAC APK alone always solves the problem.
- T1H is the active branch for this car.
- `a2()` controls visibility.
- Current hidden suppression is in VehicleDevice transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- Current T1J binding contains a hidden Fragrance visibility setter.
- A scanned static RU02 HVAC-targeting overlay unhides the entry.
- RU05 has the same T1J GONE/missing-binding defect.
- RU05 uses another Fragrance config ID or another byte/bit.
- RU05 adds a second T1H/PHEV/market/telematics/project gate after config50.
- A late RU05 `EolConfig.updateConfig()` automatically refreshes existing BottomLayout binding through observable DataBinding registration.

## Current open question

What exact RU05 service/view lifecycle creates or recreates `BottomLayoutBinding`, and can that stock path be triggered over ADB after config50 is loaded while keeping the process alive?

## Next step

Trace only:

1. where RU05 BottomLayout binding is inflated/created;
2. all assignments to `setHvacContentView` / variable ID 2;
3. owning `view/b` show/hide/recreate lifecycle;
4. exported `HvacService` actions/commands that may safely trigger view/binding recreation without process death.

If such a path exists, the live no-patch experiment becomes: signed RU05 APK -> wait for config load -> trigger stock view/binding recreation -> check whether OEM Fragrance button appears.

Do not install additional RU05 components yet, do not patch Desay APKs, and do not blind-write VDBus/properties.
