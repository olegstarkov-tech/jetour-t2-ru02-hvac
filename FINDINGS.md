# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / LIKELY / DISPROVEN / OPEN.

## PROVEN

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

### 2026-11-12 direct offline analysis: live RU02 `vdbus_extra.jar` — PROVEN

- Live pull `/system/framework/vdbus_extra.jar` has 111,020 bytes, SHA256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`; archive contains `classes.dex` with 74 class definitions.
- Actual parsed class-def table (not merely incidental strings) contains `Lcom/desaysv/ivi/extra/project/carconfig/CarConfigUtil;`, `EolConfig;`, `Constants;`, `ReserveConfigConstants;`, plus `Lcom/desaysv/ivi/extra/project/carinfo/proxy/CarInfoProxy;`. RU05 HVAC packaged the same fully qualified class definitions.
- The device's BOOTCLASSPATH includes this JAR. Under normal Android parent-first PathClassLoader delegation, the boot classes win over same-name RU05 APK classes, so replacing the HVAC APK by RU05 does not replace those carconfig implementations. This is a **PROVEN class-identity collision / boot precedence**, not proof of a different ID50 result.
- Previously retrieved RU02 framework decompile and the RU05 embedded smali both show `CarConfigUtil.init(Context)` begins with `Log.i("VDB-EXTRA-CarConfigUtil","init")`; their `getConfig(50)` paths retain `EolConfig` byte12 bit6. Therefore simply finding shadowed classes does **not** explain the missing HVAC PID13304 CarConfigUtil init log or prove config50=0.
- Cross-check of full 2026-11-12 live log: HVAC PID13304 has `VDB-EXTRA-CarInfoProxy [init]` and successful VDBus vehicle-device binding, but **no** `VDB-EXTRA-CarConfigUtil` info log or `addSubscribe event=918905` attributable to that PID. This supports a still-OPEN discrepancy in HVAC application config initialization; do not turn log absence into proof of no calls.
- A separate nonfatal `Resources$NotFoundException` occurs at RU05 `view/b.y0` after its Application onCreate began; its connection to Fragrance remains unproven.
- **Next approach:** offline compare runtime-relevant call path and system JAR method bodies against RU05/HvacApplication startup and decide a safe *read-only* discriminator. No blind in-car writes, APK patching, or repeat uiMode toggles.

### 2026-11-12 new post-reboot Fragrance visibility observation

- Direct operator evidence: Fragrance appears in the GUI **after a full HU reboot**, whereas it was absent before with signed RU05 HVAC and after an instrumented true Android day-to-night configuration change in the same process.
- **PROVEN:** Post-reboot `pm path` points to `/data/app/.../com.desaysv.svhvac-.../base.apk`, and **live SHA256 is `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`**, exactly matching the audited signed RU05 HVAC. PID post reboot is `7601` versus pre-reboot `13304`. Thus the visible Fragrance entry belongs to active **RU05**, not the preinstalled RU02 package.
- Result establishes a **successful post-reboot RU05 Fragrance UI appearance** after a negative same-process Android night-mode reconstruction test. Physical fragrance operation is not yet verified, and startup-order causation is a hypothesis, not proof.
- This distinguishes a complete boot from in-process UI refresh and suggests an initialization-order-sensitive mechanism, but the causal process-local config50 pathway is still **OPEN**. Do not claim root cause.

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
