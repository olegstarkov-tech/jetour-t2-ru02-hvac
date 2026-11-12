# RU02-HVAC NEXT

## Current objective

Discriminate whether the previously observed RU05 day/night/auto no-effect result actually exercised the proven Android configuration-change rebuild after config50 was already loaded.

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
