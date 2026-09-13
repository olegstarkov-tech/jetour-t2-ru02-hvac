# RU02-HVAC CURRENT

Status: active
Canonical branch: `main`
Recovery marker: `RU02-HVAC-CONTINUE`

## Bench

Jetour T2 / T1J, official RU, Desay SV 8155, dealer firmware 00.00.02, telematics present.

## Mission

Determine why OEM Fragrance remains hidden despite enabled vehicle config and identify the real RU02 activation mechanism.

## PROVEN

### Config and transport

- Engineering Fragrance changes config1 byte12 `0x85 -> 0xC5`; bit6 is Fragrance.
- `ID_CAR_CONFIG_FRAGRANCE = 50`; current RU02 `EolConfig` maps it from `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- Current RU02-generation HVAC `OfflineConfigManager.f()` is exactly `getConfig(50)==1 && !isT1H_PHEV()`; canonical car is T1J.
- Native VehicleDevice/event-918905 transport is closed end-to-end and contains no observed country/project/market/telematics suppression.

### HVAC operational path

Static audit artifact: `SVHvac_RU02_2026.apk`, SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`.

- FragrancePresenter is a thin delegate with no visibility/capability gate.
- `ModelFactory.b()` returns `FragranceModel`.
- FragranceModel uses only module `327690`, IDs `{58,59,60,61,62,71,92,93,94}`: type1/2/3, power, level, position/channel, remain1/2/3.
- Initial reads use the same set; writes are only 61/62/71.
- IDs 64/66/76/88 are not in the real FragranceModel subscription/read/write path.

### RU02-generation T1J UI visibility path — CLOSED inside current APK

- T1J `res/layout/bottom_layout.xml` declares `fragrance_btn` `GONE`.
- T1J `BottomLayoutBindingImpl.executeBindings()` assigns the Fragrance click listener but contains no `fragranceBtn.setVisibility(...)` and no `OfflineConfigManager.f()` call-site.
- T1J `view/b` uses `OfflineConfigManager.f()` to initialize FragranceDialog, not to expose the main button.
- No scanned static RU02 overlay targets `com.desaysv.svhvac`.
- Therefore the current RU02-generation T1J APK has no proven in-APK or static-RRO `config50 -> fragrance_btn VISIBLE` path.

### RU05 T1J UI + predicate comparison — CLOSED

Exact signed old HVAC: `SVHvac_RU05.apk`, SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.

- RU05 T1J `bottom_layout.xml` has `fragrance_btn` without initial `GONE`.
- RU05 `BottomLayoutBindingImpl` explicitly applies `fragranceBtn.setVisibility(...)`.
- Exact flow is `OfflineConfigManager.e() -> (true ? VISIBLE(0) : INVISIBLE(4)) -> fragranceBtn`.
- RU05 `OfflineConfigManager.e()` is exactly `isFragranceExist`.
- Exact `e()` body calls `CarConfigUtil.getConfig(0x32)` = `getConfig(50)`, compares only with `1`, logs, and returns; no second T1H/PHEV/market/telematics/project condition exists.

### RU05 embedded config stack — NEW PROVEN

Static class-resolution/config-source audit of the exact RU05 APK proves:

- RU05 APK itself contains `com/desaysv/ivi/extra/project/carconfig/CarConfigUtil.smali`.
- RU05 APK itself contains `EolConfig.smali`, `ReserveConfigConstants.smali`, related carconfig classes, VDBus classes, and `VDEventVehicleDevice`.
- RU05 `CarConfigUtil.getConfig(int)` directly calls its packaged `EolConfig.getJetourEolConfig(int)`.
- RU05 embedded `EolConfig` contains and loads the same vehicle property keys, including `vehicle.persist.project.ext.configs`, `...configs2`, `...configs3`, and combo config.
- RU05 `ReserveConfigConstants.KEY_CAR_CONFIG_1` is `vehicle.persist.project.ext.configs`.
- RU05 packaged callback code contains `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE` and handles `vehicle.persist.project.ext.configs*` updates.
- RU05 packaged `VDEventVehicleDevice` defines `PROJECT_RESERVE_CONFIGS = 0xe0006` and `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE = 0xe0579` (918905), matching the event family already traced in RU02.

This substantially weakens the idea that RU05 failed merely because its Fragrance predicate transparently reused the newer RU02 `CarConfigUtil/EolConfig`: RU05 carries a complete older config/VDBus implementation inside its own APK.

What is not yet proven from the quick report alone:

- whether Android runtime class loading selected the embedded RU05 carconfig classes versus an identically named parent/shared implementation;
- the exact RU05 `getJetourEolConfig(50)` bit mapping;
- the exact RU05 `EolConfig.loadConfig()` behavior and timing on startup.

## DISPROVEN / closed without new evidence

- Wrong Engineering bit or wrong config ID in the current RU02-generation path.
- Fragrance removed from current HVAC.
- Old HVAC APK alone solves it.
- T1H is the active branch for this car.
- `a2()` controls visibility.
- Hidden gate in current CarConfigUtil, VehicleDevice/event-918905 transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- The current RU02-generation T1J binding contains a hidden config50-to-Fragrance visibility setter.
- A scanned RU02 static HVAC-targeting overlay fixes/unhides `fragrance_btn`.
- RU05 shares the same T1J `GONE` + missing-binding omission as RU02-generation HVAC.
- RU05 uses another Fragrance config ID instead of 50.
- RU05 Fragrance visibility contains a second T1H/PHEV/market/telematics/project gate after config50.

## Leading explanation

For the current RU02-generation HVAC, the strongest static explanation remains a **T1J UI implementation omission/regression**.

The signed RU05 failure is now a separate compatibility/config-initialization question. Because RU05 has its own packaged CarConfig/EOL/VDBus stack and a correct visibility path, the highest-value discriminator is no longer broad class ownership; it is whether RU05 decodes config50 from the same byte/bit and whether its startup load/callback path actually populated that config before binding visibility was evaluated on the RU02 system base.

## Current open question

Does RU05 `EolConfig.getJetourEolConfig(50)` map to config1 byte12 bit6 exactly like RU02, and does RU05 `EolConfig.loadConfig()` obtain `vehicle.persist.project.ext.configs` in a way that could yield empty/default/old data on RU02 at binding time?

## Next step

One narrow read-only RU05 EolConfig trace:

1. extract exact `getJetourEolConfig(I)I` case handling for ID 50;
2. extract exact `loadConfig()` / static initialization around `vehicle.persist.project.ext.configs`;
3. extract the corresponding `CarConfigUtil.init`/service-connected path that invokes load/update;
4. only if those match RU02 semantically, investigate runtime class-loader precedence or initialization timing.

When vehicle access returns: reconcile live hashes and runtime overlay state; if still needed, log the RU05 predicate/config load directly.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H assumptions, blind writes, or unsigned HVAC patching.
