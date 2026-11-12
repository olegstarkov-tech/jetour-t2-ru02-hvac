# RU02-HVAC CURRENT

Status: active
Canonical branch: `main`
Recovery marker: `RU02-HVAC-CONTINUE`

## SUCCESS BOOT forensic comparison — 2026-11-12 (new PROVEN runtime evidence)

Direct user log files analyzed: `RU05_FRAGRANCE_CONFIG_LOG.txt` (failed after hot APK update, 31,884,609 B; PID 13304) and `RU05_FRAGRANCE_SUCCESS_BOOT.txt` (successful after full HU reboot, 20,779,601 B; PID 7601, 182,058 lines). Both runs used the exact signed RU05 HVAC, SHA256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.

### Failed RU05 launch, 16:20:53 (entry ABSENT)

- HVAC PID `13304` initialized `CarInfoProxy`, bound VehicleDevice and other services and rendered UI. **No** log `VDB-EXTRA-CarConfigUtil: init`, **no** CarConfigUtil `[onVDConnected]`, and **no** event `918905` subscribe from PID13304 across the full failed log. This negative-log evidence alone does not prove a method was never invoked, especially because RU05 static `HvacApplication.onCreate()` unconditionally calls `CarConfigUtil.init()` before `CarInfoProxy.init()`.
- The VehicleDevice `get(e0006)` attributable by timing to PID13304 at `16:20:53.233` requested **`vehicle.persist.project.pn`**, not ext.configs. (Earlier `C5` backend read at 16:20:59 was by different PID13373 `svuploadlogapp`.)
- At `16:20:53.317` PID13304 emitted nonfatal `Resources$NotFoundException: String resource ID #0x7f0f00a0` through `view/b.y0 -> K0 -> O0 -> I0 -> HvacApplication.onCreate`. Whether relevant is OPEN.
- At `16:26:00` a real Android day->night config change invoked RU05 stock rebuild coordination, but Fragrance remained absent per operator.

### Successful cold HU boot, 17:05:35 (entry VISIBLE)

- `17:05:35.048`: `I/VDB-EXTRA-CarConfigUtil(7601): init`.
- `17:05:35.111`: `VDConnector(7601)` confirms `onVDBusConnected` to VehicleDevice; `17:05:35.113`: `I/VDB-EXTRA-CarConfigUtil(7601): [onVDConnected] VDServiceDef.ServiceType.VEHICLE_DEVICE`.
- `17:05:35.114`: `VDConnector(7601): [addSubscribe] event=918905`, and `17:05:35.118`: backend `subscribe(...eventList[0]=e0579, pid=7601, packageName=com.desaysv.svhvac)`.
- `17:05:35.123`: within that very same connection/initialization sequence VehicleDevice handles `get(e0006)` for **`vehicle.persist.project.ext.configs`**, returning `b50197b55012513034300000c50a0004800c0010000100050100e10800000000` at `17:05:35.126`. This is exactly 32 bytes; index12=`0xC5`, config50=`1`. Temporal and static-call correlation strongly attribute this request to RU05 CarConfigUtil/EolConfig, though VehicleDevice's `get` line lacks explicit caller PID; do not overclaim direct caller identity/proof of static field content.
- RU05 `CarInfoProxy.init` follows at `17:05:35.152`, after CarConfigUtil connect and config read.
- No `Resources$NotFoundException` occurs in the entire 182,058-line success capture. No `VDB-EXTRA-EolConfig` or explicit `isFragranceExist` log occurs, so direct measurement of static `mCarConfig1[12]` remains unproven.
- On-device APK hash and `pidof` prove this is RU05 PID `7601`; operator visually confirms Fragrance entry visible.

### Interpretation

**PROVEN:** complete HU reboot with already-installed signed RU05 produces successful Fragrance UI entry; this successful boot contains the in-HVAC CarConfigUtil initialization + VDBus `918905` subscription + C5 ext-config read sequence that was absent in the failed hot-install launch. This is the strongest **process-initialization-path contrast**, not yet proof of why the `CarConfigUtil.init` path was not logged during hot-install despite the static unconditional call. Full reboot is a **demonstrated successful procedure in this one live A/B test**, not yet validated as universally necessary or sufficient. Root cause and whether fragrance hardware actuates remain OPEN. Do not perturb the presently successful state with blind writes/restarts.

## NEW DIRECT USER EVIDENCE — 2026-11-12, Fragrance cartridges and controls working

- On official RU02/T1J vehicle running the SHA-confirmed signed RU05 HVAC after complete HU reboot (previously observed PID7601), user reports **"Картриджи видны, все управляется"**: cartridges are visible/recognized by the Fragrance UI and the controls are operational from the user's perspective.
- **PROVEN:** not merely exposed button; Fragrance panel is usable, cartridge information is displayed, and UI/control interactions respond in this current state.
- **NOT YET DIRECTLY CONFIRMED:** physically noticeable fragrance emission, hardware output/actuation telemetry, long-term persistence across repeated starts, or standalone success on the native RU02 HVAC.
- **Do not disrupt the working state.** No more APK changes, blind VDBus writes, reboot or uiMode testing. If the operator wishes to qualify physical operation, one natural, non-invasive check is whether scent is actually emitted when enabled; avoid asserting it was proven by visual controls alone.

## NEW LIVE EVIDENCE — 2026-11-12, fragrance visible after HU reboot

- The user directly reports that **Fragrance/Aromatization appeared in the vehicle UI after rebooting the head unit**. This supersedes the earlier observation of absence, but only for the *post-reboot* state.
- Before the reboot, the exact signed RU05 HVAC had been running from `/data/app` (SHA256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`) and Fragrance stayed absent after a true Android uiMode change.
- **PROVEN after reboot:** `pm path` identifies active `/data/app/~~ReYPfSxE7hU_esE2vR25pg==/com.desaysv.svhvac-CQIUkgXPxSaJCkFmnERMPA==/base.apk`; live `sha256sum` exactly `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`, identical to the signed, previously audited RU05 HVAC. `pidof com.desaysv.svhvac` returns **7601** (earlier pre-reboot RU05 process was 13304). The user directly observes **Fragrance/Aromatization entry visible** in this post-reboot state.
- **PROVEN experiment outcome:** RU05-on-RU02 did not show the Fragrance entry immediately after APK update or following an instrumented same-process Android day/night configuration change; it **does display the entry after a complete head-unit reboot**. This proves successful OEM *UI entry exposure* for the RU05 APK on the canonical RU02 vehicle, **not yet fragrance hardware activation/operation**, and does not identify the causal loading sequence.
- **LIKELY / HYPOTHESIS ONLY:** Complete HU boot changes VDBus/CarConfigUtil initialization timing compared with in-process night-mode binding refresh. The appearance could also be another mechanism. Preserve working live state; no reinstall, restart, configuration writes or UI toggling until active APK identity and boot logs are captured.
- **DONE:** Full post-reboot log captured and analyzed; see successful-boot forensic comparison above. **NEXT:** preserve the working state and confirm UI entry interaction/hardware functionality separately when appropriate, without treating GUI exposure as hardware actuation.

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
- Retrospective clarification: the user later confirmed Fragrance **remained absent** after the 16:26 rebuild, but **appeared** after full HU reboot with the same RU05 APK.

Interpretation: The old uncertainty "does day/night actually change Android uiMode on this car and invoke RU05 b()?" is **resolved positively for this 16:26 event**. Remaining blocker: **RU05 process-local config50 state and observed UI result**. Do not repeat blind night toggles to prove a trigger already observed.

### Live BOOTCLASSPATH evidence — 2026-11-12

- Direct ADB `printenv BOOTCLASSPATH` shows `/system/framework/vdbus_extra.jar:/system/framework/vdbus.jar:/system/framework/chery-platform-internal.jar` at the end of the boot class path (alongside standard Android framework jars).
- `dumpsys package com.desaysv.svhvac | Select-String 'usesLibraries|usesLibraryFiles|usesOptionalLibraries|codePath|sharedUserId'` returned only active `/data/app/.../com.desaysv.svhvac...` and original `/product/app/SVHvac` codePaths; no matching uses-library lines in that filtered output. This does **not** exclude BOOTCLASSPATH class loading.
- Normal Android parent-first class loading means any fully qualified class present in boot class path can shadow an APK-packaged class of the same name. **HYPOTHESIS:** RU05's packaged `com.desaysv.ivi.extra.project.carconfig.CarConfigUtil` and/or `EolConfig` is shadowed by classes in system `vdbus_extra.jar`; this could account for absent RU05 `CarConfigUtil.init` logging. The live framework JAR's class inventory has not yet been checked in this step; presence of the jar alone is **not proof** of any specific duplicate class or its runtime semantics.
- One next read-only operator step: `adb pull /system/framework/vdbus_extra.jar` to PC, hash and upload for offline DEX inspection; no device writes, reinstall, night-mode toggles or config changes.

### 2026-11-12 direct offline analysis: live RU02 `vdbus_extra.jar` — PROVEN

- Live pull `/system/framework/vdbus_extra.jar` has 111,020 bytes, SHA256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`; archive contains `classes.dex` with 74 class definitions.
- Actual parsed class-def table (not merely incidental strings) contains `Lcom/desaysv/ivi/extra/project/carconfig/CarConfigUtil;`, `EolConfig;`, `Constants;`, `ReserveConfigConstants;`, plus `Lcom/desaysv/ivi/extra/project/carinfo/proxy/CarInfoProxy;`. RU05 HVAC packaged the same fully qualified class definitions.
- The device's BOOTCLASSPATH includes this JAR. Under normal Android parent-first PathClassLoader delegation, the boot classes win over same-name RU05 APK classes, so replacing the HVAC APK by RU05 does not replace those carconfig implementations. This is a **PROVEN class-identity collision / boot precedence**, not proof of a different ID50 result.
- Previously retrieved RU02 framework decompile and the RU05 embedded smali both show `CarConfigUtil.init(Context)` begins with `Log.i("VDB-EXTRA-CarConfigUtil","init")`; their `getConfig(50)` paths retain `EolConfig` byte12 bit6. Therefore simply finding shadowed classes does **not** explain the missing HVAC PID13304 CarConfigUtil init log or prove config50=0.
- Cross-check of full 2026-11-12 live log: HVAC PID13304 has `VDB-EXTRA-CarInfoProxy [init]` and successful VDBus vehicle-device binding, but **no** `VDB-EXTRA-CarConfigUtil` info log or `addSubscribe event=918905` attributable to that PID. This supports a still-OPEN discrepancy in HVAC application config initialization; do not turn log absence into proof of no calls.
- A separate nonfatal `Resources$NotFoundException` occurs at RU05 `view/b.y0` after its Application onCreate began; its connection to Fragrance remains unproven.
- **Next approach:** offline compare runtime-relevant call path and system JAR method bodies against RU05/HvacApplication startup and decide a safe *read-only* discriminator. No blind in-car writes, APK patching, or repeat uiMode toggles.

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

The night-mode transition was a genuine Android `uiMode` change handled by RU05 HVAC PID13304, but Fragrance stayed absent. The 2026-11-12 boot-path JAR contains duplicate carconfig classes, so the RU05 embedded config stack is shadowed under standard Android class loading. Why are there no CarConfigUtil init/subscription traces from the HVAC PID, and what is the process-local value of EolConfig config50? Root cause remains OPEN.

## Next step

Prioritize offline decompile comparison of live RU02 boot JAR against RU05 startup/config methods, especially the apparent discrepancy between unconditional `CarConfigUtil.init` call/log and no corresponding HVAC PID13304 line. When needed, one focused read-only live probe, chosen from that comparison. No repeated blind uiMode toggles, config writes or APK patching.
