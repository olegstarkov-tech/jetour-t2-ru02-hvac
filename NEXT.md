# RU02-HVAC NEXT

## Current objective

Prove or disprove the RU05 startup-timing / one-shot-binding explanation for the failed signed RU05-on-RU02 Fragrance A/B test.

## Current state

- Current RU02-generation Fragrance config/backend path is closed; current T1J UI has no visibility activation path and remains the leading current-generation defect.
- RU05 has a valid T1J visibility path controlled only by `CarConfigUtil.getConfig(50)==1`.
- RU05 config50 mapping is now proven identical to RU02: `(mCarConfig1[12] >> 6) & 1`.
- RU05 loads the same `vehicle.persist.project.ext.configs` key, but through VDBus `getOnce()` event `0xe0006`.
- Missing/early getOnce response can produce empty/default `mCarConfig1`.
- RU05 `CarConfigUtil.init()` loads config immediately only if VehicleDevice is already connected; otherwise it binds and returns, with `loadConfig()` deferred to `onVDConnected()`.
- Later event `0xe0579` can update the config arrays.

## Next step

Do one narrow read-only RU05 lifecycle/binding trace from the existing decoded APK:

1. find every exact call-site of `CarConfigUtil.init(Context)` and identify whether it runs from Application, Activity, service, or view lifecycle;
2. inspect `BottomLayoutBindingImpl.invalidateAll()`, `onFieldChange()`, `executeBindings()` and the dirty-flag guard around `OfflineConfigManager.e()`;
3. prove whether EolConfig/CarConfig updates have any route to `requestRebind()` / dirty flags for the Fragrance visibility expression;
4. if the predicate is one-shot, identify the safest no-patch runtime experiment for vehicle return: initialize/keep RU05 process alive until VDBus config is loaded, then recreate or reinflate the HVAC Activity/task without killing the process.

Potential payoff: if second inflation sees config50=1 and shows the stock RU05 Fragrance button, then an ADB-assisted RU05 workaround becomes possible without Desay private signing keys.

Do not install additional RU05 system components yet, do not blind-write properties/VDBus, and do not patch system APKs.
