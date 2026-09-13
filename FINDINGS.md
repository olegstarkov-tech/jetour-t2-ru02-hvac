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
- Therefore a newly executed `t1()` creates a fresh BottomLayout binding that can reevaluate the RU05 Fragrance predicate without process death.
- `view/b.I0(context)` calls `L0()` during initialization; the traced path shows `L0()` calling `t1()`.
- Exported `HvacService.onStartCommand()` accepts string extra `type` and dispatches stock commands including `OPEN_PANEL`, `CLOSE_PANEL`, `CONTROL_PANEL`, `SSS`, `HHH`, and VR open/close-fragment commands.
- Current trace does not prove these service commands directly invoke `t1()`; they mainly operate on the existing singleton `view/b` state.
- `HvacApplication` contains private `b()` with log text `onConfigurationChanged destoryAndReshow isHvacShow=` and the configuration-change path calls this method after additional UI/config handling.
- The report did not capture the middle of private `b()`, so exact destroy/reinitialize calls inside it remain unresolved.

## LIKELY

- Current RU02-generation root cause remains a T1J UI implementation omission/regression.
- Failed signed RU05-on-RU02 A/B test is plausibly explained by stale initial binding: first evaluation before async config load gives false/INVISIBLE; correct config arrives later; existing binding is not automatically reevaluated.
- Because RU05 contains a stock same-process `t1()` reinflate, this stale state is potentially recoverable without APK patching.
- The strongest no-patch candidate is a stock `HvacApplication` configuration-change destroy/re-show path that eventually reaches a fresh main/bottom binding after config50 is already loaded.

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

## OPEN

- What exactly does `HvacApplication.b()` do in the configuration-change destroy/re-show path?
- Does that path keep the process alive and reach `view/b.I0 -> L0 -> t1()` or another equivalent fresh BottomLayout creation?
- Which benign reversible Android configuration change can safely trigger that path through ADB on the live RU05 process?
- Did runtime class loading use embedded RU05 config/VDBus classes or a same-named parent/shared implementation? Investigate only if lifecycle timing fails.
- Runtime dynamic overlay and exact live APK hash reconciliation remain pending vehicle return.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only/static analysis and reversible runtime tests before writes or component replacement.
