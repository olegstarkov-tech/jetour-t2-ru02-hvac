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

### RU05 embedded config stack — CLOSED further

- RU05 APK packages its own `CarConfigUtil`, `EolConfig`, `ReserveConfigConstants`, VDBus classes and VehicleDevice-event classes.
- RU05 `CarConfigUtil.getConfig(int)` directly calls packaged `EolConfig.getJetourEolConfig(int)`.
- RU05 event constants include `PROJECT_RESERVE_CONFIGS=0xe0006` and `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE=0xe0579`.

Exact RU05 `getJetourEolConfig(50)` mapping is now proven identical to current RU02 semantics:

- packed-switch input `50` maps to `:pswitch_3e`;
- that branch reads `mCarConfig1[0x0c]` = byte/index 12;
- it follows `:goto_7`, whose shift operand is constant `6`, then masks with `1`;
- therefore RU05 config50 is exactly `(mCarConfig1[12] >> 6) & 1`.

Exact RU05 startup load path is also proven:

- `EolConfig.loadConfig()` creates a VDBus request with bundle key `type="vehicle.persist.project.ext.configs"`;
- request event is `0xe0006` (`PROJECT_RESERVE_CONFIGS`);
- it calls `VDBus.getOnce(event)`;
- if response/payload is missing, it substitutes empty string;
- `Utils.stringToByte(value)` becomes `mCarConfig1`.
- Therefore a failed/early `getOnce()` can leave `mCarConfig1` empty/default and make `getConfig(50)` return 0.

Exact RU05 `CarConfigUtil.init(Context)` ordering is proven:

- initializes VDBus and registers a bind listener;
- if `VEHICLE_DEVICE` is already connected, it subscribes to event `0xe0579`, registers notify listener, commits subscriptions, then immediately calls `EolConfig.loadConfig()`;
- if `VEHICLE_DEVICE` is not connected, it only calls `bindService(VEHICLE_DEVICE)` and returns;
- later `onVDConnected(VEHICLE_DEVICE)` performs the subscription and then calls `EolConfig.loadConfig()`;
- later event `0xe0579` updates `vehicle.persist.project.ext.configs` through `EolConfig.updateConfig(...)`.

Thus RU05 and current RU02 agree on Fragrance ID, byte, bit and property key. The remaining RU05-on-RU02 discrepancy is no longer a mapping mismatch.

## LIKELY

### Current RU02-generation root cause

The leading static explanation remains a **T1J UI implementation omission/regression**: current config50 is valid and operational Fragrance backend exists, but the current T1J entry is hardcoded `GONE` and no current T1J visibility activation path exists.

### RU05-on-RU02 failed A/B test

A strong new candidate is **startup timing / one-shot binding evaluation**:

1. RU05 `CarConfigUtil.init()` can return before `EolConfig.loadConfig()` if VehicleDevice is not yet connected.
2. Before load, `mCarConfig1` is null/empty, so RU05 `isFragranceExist -> getConfig(50)` returns false.
3. RU05 generated binding may therefore set `fragranceBtn` to `INVISIBLE` on its first evaluation.
4. `onVDConnected()` or later event 918905 can populate the correct C5/config50 state afterward.
5. If the Fragrance existence expression is only evaluated during initial binding and is not observable/rebound on EolConfig updates, the button stays hidden despite correct data arriving later.

This timing explanation is **LIKELY, not yet PROVEN**. It requires exact call-order and generated-binding dirty/rebind proof.

## DISPROVEN / closed without new evidence

- Wrong Engineering Fragrance bit or wrong current config ID.
- Fragrance removed from current HVAC.
- Old HVAC APK alone solves the problem.
- T1H is the active branch for this car.
- `a2()` controls visibility.
- Hidden current suppression is in VehicleDevice transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- Current T1J binding contains a hidden Fragrance visibility setter.
- A scanned static RU02 HVAC-targeting overlay unhides the entry.
- RU05 has the same T1J GONE/missing-binding defect.
- RU05 uses another Fragrance config ID instead of 50.
- RU05 uses another config1 byte/bit for Fragrance; exact mapping is also byte12 bit6.
- RU05 adds a second T1H/PHEV/market/telematics/project gate after config50.
- A simple RU05-vs-RU02 EOL mapping difference explains the failed old-HVAC test.

## Current open question

Is RU05 Fragrance visibility evaluated once before its asynchronous VDBus config load completes, with no automatic rebind after `mCarConfig1` is updated?

## Next step

One narrow read-only RU05 lifecycle/binding-order trace:

1. locate every call-site of `CarConfigUtil.init(Context)` and identify the containing Application/Activity/service lifecycle method;
2. inspect RU05 `BottomLayoutBindingImpl.invalidateAll()`, `onFieldChange()`, `executeBindings()` and the exact dirty-flag guard around `OfflineConfigManager.e()`;
3. prove whether any CarConfig/EolConfig update can mark that binding dirty or request a rebind;
4. if the predicate is one-shot, determine a non-patching runtime test: keep RU05 process alive until VDBus/config load completes, then recreate/reinflate the HVAC Activity/task without killing the process and see whether Fragrance appears.

When vehicle access returns: reconcile live hashes/runtime overlay state and, if needed, perform the focused RU05 timing test.

Do not install additional RU05 system components yet, do not blind-write VDBus/properties, and do not pursue unsigned HVAC patching.
