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
