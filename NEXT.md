# RU02-HVAC NEXT

## Current objective

Find a no-patch way to make signed RU05 HVAC reevaluate Fragrance visibility after its embedded config stack has loaded config50 on the RU02 system base.

## Current state

- Current RU02-generation T1J UI has no Fragrance visibility activation path; static HVAC-targeting RRO is absent.
- RU05 T1J has a valid visibility path: `OfflineConfigManager.e()` -> `getConfig(50)==1` -> `VISIBLE(0)` / `INVISIBLE(4)`.
- RU05 `getJetourEolConfig(50)` maps exactly to `(mCarConfig1[12] >> 6) & 1`, same as RU02.
- RU05 `EolConfig.loadConfig()` fetches `vehicle.persist.project.ext.configs` through VDBus `getOnce(0xe0006)` and stores it into static `mCarConfig1`.
- If VehicleDevice is already connected during `CarConfigUtil.init()`, loadConfig runs immediately. Otherwise init only starts `bindService()` and returns; `onVDConnected()` later subscribes and calls loadConfig.
- Event `0xe0579` can later update `mCarConfig1` through `EolConfig.updateConfig()`.
- `HvacApplication.onCreate()` calls `CarConfigUtil.init()` before `view/b.I0(context)`.
- RU05 has no normal HVAC Activity in its manifest; it has `HvacApplication` and exported `HvacService`.
- `BottomLayoutBindingImpl.onFieldChange()` always returns false.
- `EolConfig`/`CarConfigUtil` are not observable dependencies of the bottom binding.
- A late `EolConfig.updateConfig()` does not automatically request a rebind.
- Fragrance predicate is reevaluated only when the bottom binding's own dirty flags are set, e.g. by `invalidateAll()` or `setHvacContentView()`.

## Working hypothesis

If RU05 first evaluated Fragrance before VDBus had populated `mCarConfig1`, it could set the button `INVISIBLE`. A later successful config load would not automatically refresh that visibility. This is technically supported but not yet proven to be the exact live failure sequence.

## Next step

One narrow static lifecycle trace:

1. find where RU05 `BottomLayoutBinding` / `BottomLayoutBindingImpl` is inflated or created;
2. find where `setHvacContentView` / variable ID 2 is assigned;
3. trace the owning `view/b` lifecycle methods that create/remove/recreate the bottom view;
4. inspect exported `HvacService` actions/commands only for a safe existing way to trigger that recreation without killing the process.

If such a path exists, the live test after vehicle return is: install signed RU05 HVAC, wait until its VDBus config is populated, trigger only the stock view/binding recreation path, and check whether Fragrance becomes visible.

Do not install additional RU05 components yet. Do not patch signed Desay APKs. Do not perform blind VDBus/property writes.
