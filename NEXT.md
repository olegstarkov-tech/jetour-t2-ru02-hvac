# RU02-HVAC NEXT

## Current objective

Explain why signed RU05 HVAC hid Fragrance on the RU02 system base even though its T1J visibility predicate is exactly `CarConfigUtil.getConfig(50)==1`.

## Current state

- Current RU02-generation Fragrance config transport and backend/model path are closed.
- Current RU02-generation T1J UI still has no visibility activation path; static HVAC-targeting RRO is absent.
- RU05 has a valid T1J visibility path and uses only `getConfig(50)==1` for Fragrance visibility.
- Exact RU05 APK contains its own `CarConfigUtil`, `EolConfig`, `ReserveConfigConstants`, VDBus and VehicleDevice-event classes.
- RU05 `CarConfigUtil.getConfig(int)` directly calls its packaged `EolConfig.getJetourEolConfig(int)`.
- RU05 packaged stack uses `vehicle.persist.project.ext.configs*` and event `0xe0579` / 918905.

## Next step

Do one narrow read-only RU05 EolConfig trace from the already decoded RU05 APK:

1. print the exact `getJetourEolConfig(I)I` handling for input 50 and prove which byte/bit it reads;
2. print the exact `loadConfig()` body around `vehicle.persist.project.ext.configs` and determine how the property value is fetched/parsed and what happens if it is empty/missing;
3. print the exact `CarConfigUtil.init` / VDBus-connected path that calls `EolConfig.loadConfig()` and registers event 918905;
4. compare only these semantics with current RU02 framework.

If RU05 also maps config50 to byte12 bit6 and directly loads the same `vehicle.persist.project.ext.configs`, then class-loader precedence/startup timing becomes the next discriminator. If RU05 mapping/load semantics differ, that difference is the candidate explanation for the failed old-HVAC A/B test.

Do not rerun broad APK scans or install additional RU05 system components yet.

When the vehicle returns:

- reconcile live HVAC/CarInfo hashes;
- run runtime overlay check;
- if static analysis still leaves ambiguity, log RU05 config load/predicate directly.

Do not return to broad backend-ID guessing, VehicleDevice transport, T1H assumptions, blind writes, or unsigned HVAC patching.
