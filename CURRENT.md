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

### RU05 configuration-change destroy/rebuild path

- `onConfigurationChanged(Configuration)` reacts when either `Locale.getDefault().getLanguage()` changes or `(uiMode & 0x30)` changes.
- It calls `view/c/a.g().j()` and then private `HvacApplication.b()`.
- `b()` snapshots `HvacService.c()` (whether HVAC is currently shown), maps that state to handler message `2` (shown) or `1` (not shown), and checks `view/b.R0()`.
- If `R0()==true`, `b()` calls `view/b.d1(false)` and `HvacService.d(false)` before posting the selected message with zero delay.
- `R0()` is exactly field `K`.
- `d1(false)` is the stock `destroyHvac` path. With `false` it keeps the outer `HvacContentViewBinding`/WindowManager root, removes the old `HvacMainViewBinding` root from `rootLayout`, tears down listeners/helpers, and finally sets `K=false`.
- Handler message `1` or `2` checks `R0()`; when false it calls `view/b.I0(applicationContext)`.
- `I0()` -> `L0()` -> `t1()` inflates a new main/bottom binding in the same process.
- Message `2` then follows the shown-state path (`C1(false)`, `HvacService.d(true)`); message `1` follows the hidden-state path (`x0(false)`, `HvacService.d(false)`).
- Therefore RU05 has a fully proven OEM same-process `configuration change -> destroy old main binding -> I0/L0/t1 -> fresh BottomLayout` mechanism. Process-static `EolConfig.mCarConfig1` can survive across this rebuild.

### New live RU05 identity check — 2026-11-12, PROVEN

- Canonical RU02/T1J vehicle is live and running signed RU05 HVAC from `/data/app/~~ReYPfSxE7hU_esE2vR25pg==/com.desaysv.svhvac-CQIUkgXPxSaJCkFmnERMPA==/base.apk`.
- Live `sha256sum` exactly `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`, matching the audited RU05 source artifact.
- Live `pidof com.desaysv.svhvac` returned `13304` at that observation. PID is a point-in-time value, not fixed identity.
- `dumpsys package` identifies active data-app version `Chery-8155_11_98bba4f_2024-12-19_2412202254_R`; preinstalled 2026 variant still resides at `/product/app/SVHvac` with `Chery-8155_11_30e96fe2_2026-02-25_2604101629_R`.
- No fresh live proof yet that the RU05 process has actually loaded config50=1. Next step is read-only capture of *existing* logcat; do not clear logs, force-stop, or toggle day/night beforehand.

### Prior live RU05 day/night toggle observation — NEW DIRECT EVIDENCE

- During an earlier live test on the running canonical vehicle with signed RU05 HVAC installed, the user manually switched the HU quick-shade day/night/auto modes multiple times.
- Fragrance did **not** appear after those user-visible day/night/auto switches.
- This is direct negative live evidence against a simplistic claim that "any day/night toggle after RU05 startup is enough".
- However that historical test did not capture ADB/logcat evidence proving that the shade control actually changed Android `Configuration.uiMode & 0x30`, triggered the exact RU05 `onConfigurationChanged -> b() -> d1(false) -> I0/t1` path, preserved the same process, or occurred after RU05 `getConfig(50)` had become `1`.
- Therefore the specific stale-binding/reinflate hypothesis is **weakened but not yet disproven**.

### RU05 exported service control surface

`HvacService` is exported and `onStartCommand()` reads string extra `type`. Static dispatch contains stock commands including `OPEN_PANEL`, `CLOSE_PANEL`, `CONTROL_PANEL`, `SSS`, `HHH`, and VR open/close-fragment commands. Current evidence still does not prove those commands themselves perform the full reinflate; the configuration-change path is the proven refresh primitive.

### Live Fragrance visibility confirmation — 2026-11-12

- User directly confirmed: after the **real** 16:26 Android day(16) -> night(32) configuration event, with the identical signed RU05 process PID 13304 running and stock `HvacApplication.onConfigurationChanged -> b() -> Handler` callbacks observed, **the Fragrance button remained absent**.
- **PROVEN negative test:** changing Android night mode is NOT sufficient to expose Fragrance on this car with RU05 HVAC. Do not repeat uninstrumented day/night switches.
- Earlier capture shows the RU05 PID 13304 unsubscribed from carinfo callbacks at 16:26:00.396–.432 and re-subscribed at .588–.627, consistent with UI teardown/reinitialization. A direct `t1()` runtime log is absent, so label the exact new bottom view execution as static inference, not independently logged.
- **OPEN:** RU05 in-process `CarConfigUtil/EolConfig` state: RU05 APK's `CarConfigUtil.init` contains unconditional Android `Log.i("VDB-EXTRA-CarConfigUtil", "init")`, yet no such tag from PID 13304 occurs in full log; `CarInfoProxy` `[init]` does appear. This is noteworthy evidence supporting possible class resolution/framework shadowing or alternative runtime implementation, not proof yet.
- Next focused read-only diagnostic: inspect device `BOOTCLASSPATH` and `dumpsys package` for shared/uses-library class sources; determine if framework `vdbus_extra.jar` could take precedence over embedded RU05 config classes. Do not probe Java static state with invented Binder transactions, alter configs, reinstall or toggle blindly.

### Live logcat investigation — 2026-11-12, PROVEN / OPEN

Full user log `RU05_FRAGRANCE_CONFIG_LOG.txt` was analyzed locally (297,718 lines). Key timeline:

- `16:20:34.548` Android installed RU05 update and switched `com.desaysv.svhvac` system package code path to `/data/app/...`, while retaining data.
- `16:20:53.132-16:20:53.172` system started process PID `13304` for `com.desaysv.svhvac.service.HvacService`; startup reports `VDRouter client=com.desaysv.svhvac` and binds `VEHICLE_DEVICE` by `16:20:53.233`.
- `16:20:53.317` RU05 `view/b.y0 -> K0 -> O0 -> I0 -> HvacApplication.onCreate` logs a caught/reported `Resources$NotFoundException: String resource ID #0x7f0f00a0`. It did not terminate PID 13304; its relation to Fragrance is unproven.
- `16:20:59.502-16:20:59.504` VehicleDevice backend returned `vehicle.persist.project.ext.configs=b50197b55012513034300000c50a0004800c0010000100050100e10800000000` (32 bytes; byte12=`C5`, config50=1). **Important attribution:** concurrent request was initiated by PID `13373`, `com.desaysv.svuploadlogapp`, which logged `VDB-EXTRA-CarConfigUtil init/onVDConnected` and subscription event 918905, **not HVAC PID 13304**. This proves backend stores bit6=1, not that RU05 process loaded it.
- `16:26:00.388` Android night-mode genuinely changed `16 -> 32` without language change; system UiModeManager applied configuration. The existing RU05 PID `13304` executed `HvacApplication.onConfigurationChanged -> b() -> HvacService.d` at `16:26:00.435`; same PID later reached `HvacApplication$a.handleMessage -> HvacService.d` at `16:26:00.880`. This confirms the exact stock configuration-change and handler code actually executed at runtime after startup, without HVAC process death. Exact `t1()` is statically reachable; there is no direct runtime t1 log in this capture.
- Full log contains **zero** `VDB-EXTRA-EolConfig`, `[loadConfig] carConfig1`, `isFragranceExist` lines and no `VDB-EXTRA-CarConfigUtil` records from PID `13304`. Absence of these lines does **not prove** calls never occurred or predicate=false. Other clients log CarConfigUtil normally; RU05 class-loader/logging path now merits targeted checking if UI stays hidden.
- The user has not yet confirmed whether the Fragrance button was visible immediately after this 16:26 live rebuild; do not infer the outcome from the log alone.

Interpretation: The old uncertainty "does day/night actually change Android uiMode on this car and invoke RU05 b()?" is **resolved positively for this 16:26 event**. Remaining blocker: **RU05 process-local config50 state and observed UI result**. Do not repeat blind night toggles to prove a trigger already observed.

### Live BOOTCLASSPATH evidence — 2026-11-12

- Direct ADB `printenv BOOTCLASSPATH` shows `/system/framework/vdbus_extra.jar:/system/framework/vdbus.jar:/system/framework/chery-platform-internal.jar` at the end of the boot class path (alongside standard Android framework jars).
- `dumpsys package com.desaysv.svhvac | Select-String 'usesLibraries|usesLibraryFiles|usesOptionalLibraries|codePath|sharedUserId'` returned only active `/data/app/.../com.desaysv.svhvac...` and original `/product/app/SVHvac` codePaths; no matching uses-library lines in that filtered output. This does **not** exclude BOOTCLASSPATH class loading.
- Normal Android parent-first class loading means any fully qualified class present in boot class path can shadow an APK-packaged class of the same name. **HYPOTHESIS:** RU05's packaged `com.desaysv.ivi.extra.project.carconfig.CarConfigUtil` and/or `EolConfig` is shadowed by classes in system `vdbus_extra.jar`; this could account for absent RU05 `CarConfigUtil.init` logging. The live framework JAR's class inventory has not yet been checked in this step; presence of the jar alone is **not proof** of any specific duplicate class or its runtime semantics.
- One next read-only operator step: `adb pull /system/framework/vdbus_extra.jar` to PC, hash and upload for offline DEX inspection; no device writes, reinstall, night-mode toggles or config changes.

## LIKELY

### Current RU02-generation root cause

Strongest static explanation remains a T1J UI implementation omission/regression.

### RU05-on-RU02 failed A/B test

Startup-order/stale-binding remains possible, but confidence is reduced by the historical live day/night/auto no-effect observation.

The decisive future test must not merely toggle a visible day/night control. It must simultaneously prove at runtime:

1. RU05 `isFragranceExist` / config50 is already `true` before rebuild;
2. the toggle actually causes RU05 `onConfigurationChanged` and `destoryAndReshow`;
3. PID/process remains alive;
4. fresh BottomLayout is created;
5. Fragrance visibility result after that exact fresh binding.

If those conditions are all observed and the button still remains absent, the stale-binding workaround is disproven and RU05 runtime class/config resolution or another UI runtime effect must be investigated.

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
- Any arbitrary quick-shade day/night/auto toggle is already proven sufficient to expose Fragrance; direct live observation shows no visible effect in the prior test.

## Current open question

Did the prior quick-shade day/night/auto switching actually execute the exact Android `uiMode` configuration-change rebuild **after RU05 config50 had become 1**? If yes, the stale-binding workaround is effectively disproven. If no, a controlled ADB/logcat test remains necessary.

## Next step

When the canonical vehicle is available, do not start by blindly toggling night mode again. First establish the runtime preconditions read-only:

1. run exact signed RU05 HVAC;
2. capture PID;
3. prove from logs/runtime that RU05 `isFragranceExist` / config50 is `1`;
4. capture current Android `uiMode` and start focused logcat;
5. only then trigger one reversible ADB `uiMode` change and verify whether the exact `onConfigurationChanged -> destoryAndReshow -> destroyHvac -> I0/t1` chain executes with the same PID.

Do not install additional RU05 components, patch Desay APKs, or blind-write VDBus/properties.
