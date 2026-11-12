# RU02-HVAC NEXT

## Live operator confirmation: cartridges and Fragrance controls — 2026-11-12

- **PROVEN:** After full HU reboot with verified signed RU05 HVAC, user directly confirms `Картриджи видны, все управляется`. This verifies the displayed cartridge interface and responsiveness of controls, beyond simply making the Fragrance button visible.
- **OPEN:** Whether physical scent is emitted; user has not explicitly confirmed that, nor the repeatability of this result on subsequent boots.
- Preserve the working installation. No reinstall, reboot, blind VDBus writes or more night-mode toggles.
- **One optional natural next step:** confirm whether scent actually comes from the diffuser during standard UI operation. Otherwise pursue only offline root-cause comparison between failed hot-install and successful cold-boot CarConfigUtil initialization.

## Updated priority after SUCCESS log analysis — 2026-11-12

- **COMPLETE/PROVEN:** RU05 confirmed running by SHA and PID7601 after full HU reboot; Fragrance button appears.
- **COMPLETE/PROVEN:** Successful boot captured and compared with hot-install failure. At 17:05:35.048 RU05 PID7601 logged `CarConfigUtil.init`; at .113 `onVDConnected`; at .114 subscribed to 918905; at .123–.126 VehicleDevice read `vehicle.persist.project.ext.configs` ending in C5 at byte12 (config50=1). `CarInfoProxy.init` happened at .152.
- Failed PID13304 launched UI and `CarInfoProxy` but did not log CarConfigUtil init, onVDConnected or own 918905 subscription; its early vehicle `get(e0006)` at 16:20:53.233 read `vehicle.persist.project.pn` only. A post-install `Resources$NotFoundException` was absent from success boot.
- **LIKELY:** config initialization ordering/path explains visible/hidden difference. **OPEN:** exact reason hot-install run lacked CarConfigUtil init trace even though static onCreate calls it; no direct measurement of in-process `mCarConfig1[12]`; no proof of actual fragrance hardware operation.
- **ONE NEXT SAFE STEP:** preserve the successful state; when the user is ready, verify from the existing exposed Fragrance panel whether it opens/responds, without additional reboots, installs, VDBus writes or configuration changes. Treat any test of fragrance hardware separately and avoid assuming equipment is fitted.

## Current objective

SUCCESS UI exposure is proven after HU reboot with signed RU05. Narrow the remaining open question to initialization-path causality and actual fragrance hardware behavior; preserve working state.

## Current state

- Current RU02-generation T1J UI has no proven Fragrance visibility activation path; static HVAC-targeting RRO is absent.
- RU05 T1J has a valid visibility path: `OfflineConfigManager.e()` -> `getConfig(50)==1` -> `VISIBLE(0)` / `INVISIBLE(4)`.
- RU05 maps config50 exactly as `(mCarConfig1[12] >> 6) & 1` and loads the project ext-config through its embedded VDBus stack.
- Late config update does not automatically refresh the existing BottomLayout binding.
- RU05 `view/b.t1()` rebuilds the main/bottom DataBinding tree in the same process.
- Exact configuration-change path is proven:
  `onConfigurationChanged()` -> private `b()` -> if shown `d1(false)` -> handler message -> `I0(context)` -> `L0()` -> `t1()` -> fresh BottomLayout binding.
- `onConfigurationChanged()` enters this rebuild when language changes or `(uiMode & 0x30)` changes.
- Historical direct live evidence: while signed RU05 HVAC was installed on the running canonical vehicle, the user manually switched HU quick-shade day/night/auto modes and Fragrance did not appear.
- That historical test did not capture whether the shade switch actually changed Android `uiMode & 0x30`, triggered the exact rebuild, preserved the same PID, or occurred after RU05 `getConfig(50)` became `1`.

### Updated live state, 2026-11-12

- DONE: exact RU05 APK identity (SHA), PID 13304, backend ext config byte12=C5/ID50=1; 297,718-line logcat analyzed.
- DONE: at 16:26:00 Android night-mode changed from 16 to 32; same RU05 PID ran `HvacApplication.onConfigurationChanged -> b()` and its Handler branch. Repeating an uninstrumented night toggle is unnecessary.
- NOT YET PROVEN: RU05's own in-process `mCarConfig1[12]` / `OfflineConfigManager.e()=true`. C5 backend request at 16:20:59 belongs to unrelated log uploader process PID 13373.
- NOT YET OBSERVED: UI Fragrance visibility after the 16:26 refresh. Ask for a direct visual observation first; do not assume it stayed hidden or appeared from log alone.
- If hidden, shift next targeted diagnostic toward RU05 `CarConfigUtil/EolConfig` class-loading and actual getConfig(50) runtime path before any additional UI/theme tests. Static source says init should happen; `VDB-EXTRA-CarConfigUtil` lines absent specifically from PID 13304 while other clients log them, but absence is inconclusive.
- Also record nonfatal RU05 `Resources$NotFoundException` 0x7f0f00a0 in `view/b.I0` as separate runtime anomaly; do not assign root cause yet.

### Live result update — 2026-11-12

- Operator confirms **Fragrance still absent** after the real 16:26:00 Android day/night event handled by signed RU05 HVAC PID13304. No more blind uiMode toggling; it did not solve the problem.
- Most promising new distinction: exact RU05 APK `CarConfigUtil.init` unconditionally logs tag `VDB-EXTRA-CarConfigUtil`, but PID13304 did not emit it at startup, whereas CarInfoProxy and other VDBus logs are present. Investigate shared framework class loading rather than assuming process-local config50=1.
- **One next read-only step:** retrieve `BOOTCLASSPATH` and `dumpsys package com.desaysv.svhvac` library details to assess whether framework `vdbus_extra.jar` can supersede APK-embedded classes. Then decide any next probe based on evidence; no writes/reboots/config mutations.

### Next distinction — duplicate-class collision PROVEN 2026-11-12

- Pulled live boot JAR and parsed 74 DEX class definitions. `CarConfigUtil`, `EolConfig`, `CarInfoProxy`, and related classes duplicate RU05 APK FQCNs; under standard Android boot-class resolution the framework implementation wins.
- Both available RU02 decompile and RU05 smali begin `CarConfigUtil.init` with the same info log and keep config50 byte12 bit6; class overlap alone is not the cause of missing log or missing UI.
- **NEXT:** offline inspect the exact application init and class-resolution path, and seek a safe process-local config50 read; do not repeat night-mode/reinstall experiments.

### 2026-11-12 direct offline analysis: live RU02 `vdbus_extra.jar` — PROVEN

- Live pull `/system/framework/vdbus_extra.jar` has 111,020 bytes, SHA256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`; archive contains `classes.dex` with 74 class definitions.
- Actual parsed class-def table (not merely incidental strings) contains `Lcom/desaysv/ivi/extra/project/carconfig/CarConfigUtil;`, `EolConfig;`, `Constants;`, `ReserveConfigConstants;`, plus `Lcom/desaysv/ivi/extra/project/carinfo/proxy/CarInfoProxy;`. RU05 HVAC packaged the same fully qualified class definitions.
- The device's BOOTCLASSPATH includes this JAR. Under normal Android parent-first PathClassLoader delegation, the boot classes win over same-name RU05 APK classes, so replacing the HVAC APK by RU05 does not replace those carconfig implementations. This is a **PROVEN class-identity collision / boot precedence**, not proof of a different ID50 result.
- Previously retrieved RU02 framework decompile and the RU05 embedded smali both show `CarConfigUtil.init(Context)` begins with `Log.i("VDB-EXTRA-CarConfigUtil","init")`; their `getConfig(50)` paths retain `EolConfig` byte12 bit6. Therefore simply finding shadowed classes does **not** explain the missing HVAC PID13304 CarConfigUtil init log or prove config50=0.
- Cross-check of full 2026-11-12 live log: HVAC PID13304 has `VDB-EXTRA-CarInfoProxy [init]` and successful VDBus vehicle-device binding, but **no** `VDB-EXTRA-CarConfigUtil` info log or `addSubscribe event=918905` attributable to that PID. This supports a still-OPEN discrepancy in HVAC application config initialization; do not turn log absence into proof of no calls.
- A separate nonfatal `Resources$NotFoundException` occurs at RU05 `view/b.y0` after its Application onCreate began; its connection to Fragrance remains unproven.
- **Next approach:** offline compare runtime-relevant call path and system JAR method bodies against RU05/HvacApplication startup and decide a safe *read-only* discriminator. No blind in-car writes, APK patching, or repeat uiMode toggles.

## Next step

Vehicle is now available. As of 2026-11-12 live evidence, exact RU05 signed APK SHA matches `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`, active `/data/app/.../base.apk`, PID snapshot `13304`. Stock 2026 HVAC stays at `/product/app/SVHvac`. **Identity/PID step complete.**

Current one read-only step: capture existing `adb logcat -d -v time` without clearing buffers, killing process, or changing uiMode. Filter `VDB-EXTRA-EolConfig`, `[loadConfig] carConfig1`, `VDB-EXTRA-CarConfigUtil`, `isFragranceExist`, `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`, and `onVDConnected`; then interpret RU05 `config1[12]` bit6 if present. Missing historical log lines are inconclusive.

After confirming config, capture current Android night/uiMode state and focused logcat filters for `onConfigurationChanged`, `destoryAndReshow`, `destroyHvac`, `HvacContentView init`, and `isFragranceExist`.

Only after that evidence is live, toggle Android night mode once through ADB and observe:

- whether PID remains unchanged;
- whether the exact configuration rebuild chain runs;
- whether fresh binding logs `isFragranceExist=true`;
- whether the Fragrance entry becomes visible.

Interpretation:

- rebuild executes + same PID + `isFragranceExist=true` + button appears -> no-patch RU05 workaround proven;
- rebuild executes + same PID + `isFragranceExist=true` + button absent -> stale-binding workaround disproven; reopen only RU05 runtime binding/UI path;
- rebuild executes but `isFragranceExist=false` -> investigate RU05 runtime config/class-loading path;
- shade control does not produce the same Android configuration-change logs -> prior manual day/night/auto observation does not test this mechanism.

Do not install additional RU05 components, patch Desay APKs, or blind-write VDBus/properties.
