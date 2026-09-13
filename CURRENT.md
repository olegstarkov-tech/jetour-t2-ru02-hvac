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

### RU05 binding refresh behavior

- `HvacApplication.onCreate()` calls `CarConfigUtil.init(applicationContext)` before later HVAC view initialization including `view/b.I0(context)`.
- RU05 manifest has `HvacApplication` and exported `HvacService`; no normal HVAC Activity entry is declared.
- `BottomLayoutBindingImpl.onFieldChange(...)` always returns `false`.
- The bottom binding does not register `EolConfig` or `CarConfigUtil` as observable dependencies.
- `EolConfig.updateConfig()` only replaces static arrays; no direct route to `BottomLayoutBindingImpl.requestRebind()` exists.
- Fragrance predicate is reevaluated when the bottom binding's own dirty flags are set, including `invalidateAll()` and `setHvacContentView(...)`.
- Therefore a late RU05 config load/update does **not automatically** recompute existing Fragrance visibility.

### RU05 same-process UI reconstruction path

- `view/b.t1()` removes the old main HVAC child from `HvacContentViewBinding.rootLayout`, clears old listeners, inflates a **new** `HvacMainViewBinding`, assigns its new `bottomLayout`, and adds the new root back.
- `view/b.I0(context)` calls `L0()`; `L0()` calls `t1()`.
- Therefore `I0()` creates a fresh BottomLayout DataBinding tree without process death, so the RU05 Fragrance predicate is reevaluated from current static config state.

### RU05 configuration-change destroy/rebuild path — NEW PROVEN

Exact `HvacApplication` and handler trace closes the stock same-process refresh mechanism:

- `onConfigurationChanged(Configuration)` reacts when either `Locale.getDefault().getLanguage()` changes or `(uiMode & 0x30)` changes.
- It calls `view/c/a.g().j()` and then private `HvacApplication.b()`.
- `b()` snapshots `HvacService.c()` (whether HVAC is currently shown), maps that state to handler message `2` (shown) or `1` (not shown), and checks `view/b.R0()`.
- If `R0()==true`, `b()` calls `view/b.d1(false)` and `HvacService.d(false)` before posting the selected message with zero delay.
- `R0()` is exactly field `K`.
- `d1(false)` is the stock `destroyHvac` path. With `false` it keeps the outer `HvacContentViewBinding`/WindowManager root, removes the old `HvacMainViewBinding` root from `rootLayout`, tears down listeners/helpers, and finally sets `K=false`.
- Handler message `1` or `2` checks `R0()`; when false it calls `view/b.I0(applicationContext)`.
- `I0()` -> `L0()` -> `t1()` inflates a new main/bottom binding in the same process.
- Message `2` then follows the shown-state path (`C1(false)`, `HvacService.d(true)`); message `1` follows the hidden-state path (`x0(false)`, `HvacService.d(false)`).

Therefore RU05 has a fully proven OEM same-process `configuration change -> destroy old main binding -> I0/L0/t1 -> fresh BottomLayout` mechanism. Process-static `EolConfig.mCarConfig1` can survive across this rebuild.

### RU05 exported service control surface

`HvacService` is exported and `onStartCommand()` reads string extra `type`. Static dispatch contains stock commands including `OPEN_PANEL`, `CLOSE_PANEL`, `CONTROL_PANEL`, `SSS`, `HHH`, and VR open/close-fragment commands. Current evidence still does not prove those commands themselves perform the full reinflate; the configuration-change path is the proven refresh primitive.

## LIKELY

### Current RU02-generation root cause

Strongest static explanation remains a T1J UI implementation omission/regression.

### RU05-on-RU02 failed A/B test

A startup-order/stale-binding failure is technically well supported:

1. if VehicleDevice was not yet connected when RU05 `CarConfigUtil.init()` ran, config load was deferred;
2. if the first BottomLayout binding evaluated before `mCarConfig1` was populated, `getConfig(50)` returned false and Fragrance became `INVISIBLE`;
3. later `onVDConnected()`/event 918905 could populate config50 correctly;
4. that config update would not automatically refresh existing bottom-binding visibility.

The newly proven configuration-change destroy/rebuild path provides a concrete no-patch way to test this hypothesis while preserving the RU05 process and static config state.

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
- `OPEN_PANEL` by itself is already proven to recreate BottomLayout; current trace does not support that claim.
- RU05 configuration-change handling is only a hide/show path; exact code proves full same-process main/bottom binding reconstruction.

## Current open question

Live question only: after signed RU05 has loaded config50=1 on the RU02 system base, will a reversible `uiMode` configuration change trigger the proven OEM rebuild and make the Fragrance entry visible?

## Next step

When the canonical vehicle is available:

1. install/start exact signed RU05 HVAC and keep the process alive;
2. verify/wait until RU05 config50 is loaded as `1` from runtime evidence;
3. read and save current Android night/uiMode state;
4. trigger one reversible night-mode change through ADB to force `onConfigurationChanged()`;
5. verify logs for `destoryAndReshow`, `destroyHvac`, `init`, and the Fragrance existence predicate; visually check the button;
6. restore the original night-mode state immediately.

Before the changing command, first perform only the read-only state capture. Do not install additional RU05 components, patch Desay APKs, or blind-write VDBus/properties.
