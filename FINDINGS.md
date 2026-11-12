# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / LIKELY / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes config1 byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- Current RU02 Fragrance config is ID50 from `vehicle.persist.project.ext.configs`; current startup has been observed with byte12=`C5`.
- Current RU02-generation T1J `bottom_layout.xml` sets `fragrance_btn` `GONE`; current T1J generated binding has no Fragrance visibility setter/predicate, and no scanned static RU02 HVAC RRO compensates.
- Exact signed RU05 HVAC is `SVHvac_RU05.apk`, SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.
- RU05 has a valid T1J Fragrance visibility path: `OfflineConfigManager.e()` controls `fragranceBtn.setVisibility(true?0:4)`.
- RU05 `e()` is exactly `isFragranceExist` and computes only `CarConfigUtil.getConfig(50)==1`.
- RU05 packages its own `CarConfigUtil`, `EolConfig`, `ReserveConfigConstants`, VDBus and VehicleDevice event classes.
- RU05 `getJetourEolConfig(50)` is exactly `(mCarConfig1[12] >> 6) & 1`, identical to current RU02.
- RU05 `EolConfig.loadConfig()` requests `vehicle.persist.project.ext.configs` through VDBus `getOnce(0xe0006)`; absent response/payload can yield empty/default config.
- RU05 `CarConfigUtil.init(Context)` loads immediately only if VehicleDevice is already connected; otherwise load is deferred to `onVDConnected()`.
- Event `0xe0579` can later update `mCarConfig1` through `EolConfig.updateConfig()`.
- `HvacApplication.onCreate()` invokes `CarConfigUtil.init()` before later `view/b.I0(context)` initialization.
- RU05 manifest declares exported `HvacService` and no normal HVAC Activity.
- RU05 `BottomLayoutBindingImpl.onFieldChange()` always returns false; EolConfig/CarConfig are not observable binding dependencies.
- Late `EolConfig.updateConfig()` does not automatically recompute existing Fragrance visibility.
- RU05 `view/b.t1()` is a stock same-process main-view reinflate primitive: it removes the old main view, inflates a new `HvacMainViewBinding`, assigns the new `bottomLayout`, and adds the new root back into the existing HVAC root.
- `view/b.I0(context)` calls `L0()` and that path reaches `t1()`.
- `HvacApplication.onConfigurationChanged()` explicitly compares current language and `(uiMode & 0x30)` against cached values and calls private rebuild method `b()` when either changes.
- `HvacApplication.b()` snapshots `HvacService.c()` and chooses handler message 2 if HVAC was shown, otherwise message 1.
- If `view/b.R0()` is true, `b()` calls `view/b.d1(false)` plus `HvacService.d(false)` before immediately posting the selected handler message.
- `R0()` is exactly field `K`.
- `d1(false)` is `destroyHvac`: it preserves the outer `HvacContentViewBinding`/process but removes the old `HvacMainViewBinding` root and tears down the old UI state, ending with `K=false`.
- Handler message 1 or 2 then calls `view/b.I0(applicationContext)` when `R0()==false`.
- Therefore RU05 has a fully proven OEM same-process chain: `configuration change -> d1(false) -> handler -> I0 -> L0 -> t1 -> fresh BottomLayoutBinding`.
- This rebuild does not kill the process, so process-static config state such as `EolConfig.mCarConfig1` can survive and be reevaluated by the new RU05 binding.
- Exported `HvacService.onStartCommand()` accepts string extra `type` and exposes stock commands including `OPEN_PANEL`, `CLOSE_PANEL`, `CONTROL_PANEL`, `SSS`, `HHH`, and VR open/close-fragment commands; these are not currently proven to be the full reinflate trigger themselves.
- During an earlier live run with signed RU05 HVAC installed on the running canonical vehicle, the user manually switched HU quick-shade day/night/auto modes and Fragrance still did not appear.
- That historical live test did not capture whether the shade control changed Android `uiMode & 0x30`, triggered the exact `onConfigurationChanged -> destroy/rebuild` path, preserved the same process, or happened after RU05 config50 had become `1`.

### New live identity evidence (2026-11-12)

- User's canonical RU02/T1J car currently runs the exact signed RU05 HVAC, device-side SHA256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`, active in `/data/app/.../base.apk`; PID at the observation `13304`.
- Stock 2026 HVAC remains installed in `/product/app/SVHvac` but is not the active APK.
- The live RU05 config50 value is **not yet observed**; identity/PID alone do not prove Fragrance config loaded.

### 2026-11-12 live logcat — new proven boundaries

- The exact signed RU05 process was PID `13304` starting at 16:20:53, and the same PID handled a real Android `uiMode & 0x30` change from day (16) to night (32) at 16:26:00: `HvacApplication.onConfigurationChanged -> b()` followed by `HvacApplication$a.handleMessage`. Thus the real OS configuration-change trigger and RU05 rebuild coordinator executed, not just a cosmetic quick-shade theme change.
- Backend `vehicle.persist.project.ext.configs` read at 16:20:59 contains byte12 `C5`, config50=1, but the requester was `com.desaysv.svuploadlogapp` PID 13373, not RU05 HVAC. No proof of RU05 process-local `mCarConfig1` / `getConfig(50)` yet.
- No `VDB-EXTRA-EolConfig` or `isFragranceExist` logs anywhere in capture; no `VDB-EXTRA-CarConfigUtil` log from RU05 PID 13304. These absences are NOT proof of no calls and not proof of false gate.
- RU05 startup logged `Resources$NotFoundException: String resource ID #0x7f0f00a0` inside `view/b.y0 -> K0 -> O0 -> I0`; the process continued. Treat as an independent runtime anomaly, not Fragrance root cause without evidence.

### RU05 night-mode test outcome (2026-11-12 direct operator observation)

- With verified signed RU05 APK active (PID13304), a real Android day->night configuration change at 16:26 caused RU05 `HvacApplication.onConfigurationChanged` and handler execution, and the operator confirms **Fragrance did NOT become visible** afterward.
- Direct runtime evidence of unsubscribing/re-subscribing CarInfo listeners before/after handler supports UI teardown/rebuild; runtime `t1()` invocation not independently logged.
- Simple day/night UI refresh as a successful workaround is ruled out for the observed test. In-process RU05 config50 remains unknown; backend config50=1 was read by *another* app.
- RU05 embedded `CarConfigUtil.init` has an unconditional `Log.i("VDB-EXTRA-CarConfigUtil", "init")` before VDBus initialization; PID13304 emitted no such message, even though `VDB-EXTRA-CarInfoProxy init` occurred. Treat runtime class source / system jar shadowing as a **hypothesis**, not proven.

### BOOTCLASSPATH / duplicate-class hypothesis (2026-11-12)

- Live RU02 environment includes `/system/framework/vdbus_extra.jar`, `/system/framework/vdbus.jar`, and `/system/framework/chery-platform-internal.jar` in `BOOTCLASSPATH`. A filtered `dumpsys package` shows data-app active and product-app preinstalled HVAC but no matching optional/shared-library lines; boot class path entries do not require an app manifest `uses-library` declaration.
- If `com.desaysv.ivi.extra.project.carconfig.CarConfigUtil` or `EolConfig` appears in the boot-path jar, parent-first lookup is a technically grounded mechanism for shadowing packaged RU05 implementations.
- The duplicated class descriptors and their exact runtime behavior are **unverified** pending offline extraction of the system jar. The previous absence of expected logging remains evidence worth investigating but does not prove class substitution by itself.

## LIKELY

- Current RU02-generation root cause remains a T1J UI implementation omission/regression.
- Failed signed RU05-on-RU02 A/B test may still involve stale initial binding, but confidence is reduced by the historical quick-shade day/night/auto no-effect observation.
- A controlled ADB/logcat test is required; simply repeating a user-visible day/night toggle without proving runtime preconditions is not sufficient.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit in current RU02 path.
- Current HVAC uses a different Fragrance config ID.
- Fragrance was removed from current HVAC.
- Old HVAC APK alone always solves the problem.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` controls visibility.
- Current hidden suppression is in VehicleDevice transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- Current T1J binding contains a hidden Fragrance visibility setter.
- A scanned static RU02 HVAC overlay unhides the entry.
- RU05 has the same T1J GONE/missing-binding defect.
- RU05 uses another Fragrance config ID or another byte/bit.
- RU05 adds a second T1H/PHEV/market/telematics/project gate after config50.
- A late RU05 config update automatically refreshes BottomLayout via observable DataBinding.
- `OPEN_PANEL` is already proven to be the BottomLayout-reinflate trigger.
- RU05 configuration-change handling is merely hide/show without binding reconstruction.
- Any arbitrary quick-shade day/night/auto switch is already proven sufficient to expose Fragrance.

## OPEN

- Did the prior quick-shade day/night/auto action actually alter Android `uiMode & 0x30` and execute the exact RU05 rebuild path?
- Was RU05 config50 already `1` when that prior visual toggle occurred?
- If a controlled rebuild is proven with same PID and `isFragranceExist=true` yet the button remains absent, investigate runtime class loading or another RU05 UI runtime effect.
- Runtime dynamic overlay and exact live APK hash reconciliation remain pending vehicle return.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only/static analysis and reversible runtime tests before writes or component replacement.
