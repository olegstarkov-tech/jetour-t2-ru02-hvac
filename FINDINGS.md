# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes config1 byte12 `0x85 -> 0xC5`; delta `0x40` = bit6.
- Current RU02 Fragrance config is ID50 from `vehicle.persist.project.ext.configs`; current startup has been observed with byte12=`C5`.
- Current RU02-generation T1J `bottom_layout.xml` sets `fragrance_btn` `GONE`; current T1J generated binding has no Fragrance visibility setter/predicate, and no scanned static RU02 HVAC RRO compensates.
- Exact signed RU05 HVAC is `SVHvac_RU05.apk`, SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.
- RU05 has a valid T1J Fragrance visibility path: `OfflineConfigManager.e()` controls `fragranceBtn.setVisibility(true?0:4)`.
- RU05 `e()` is exactly `isFragranceExist` and computes only `CarConfigUtil.getConfig(50)==1`.
- RU05 packages its own `CarConfigUtil`, `EolConfig`, `ReserveConfigConstants`, VDBus and VehicleDevice event classes.
- RU05 `CarConfigUtil.getConfig(int)` directly delegates to packaged `EolConfig.getJetourEolConfig(int)`.
- RU05 `getJetourEolConfig(50)` is exactly `(mCarConfig1[12] >> 6) & 1`, identical to the current RU02 Fragrance bit mapping.
- RU05 `EolConfig.loadConfig()` requests `vehicle.persist.project.ext.configs` through VDBus event `0xe0006` / `PROJECT_RESERVE_CONFIGS` using `VDBus.getOnce()` and parses response field `value` into `mCarConfig1`.
- If the `getOnce()` response or payload is absent, RU05 substitutes an empty string before `stringToByte`, allowing config50 to resolve as default/0.
- RU05 `CarConfigUtil.init(Context)` has two startup branches: when VehicleDevice is already connected it subscribes to `0xe0579` and immediately calls `EolConfig.loadConfig()`; otherwise it calls `bindService(VEHICLE_DEVICE)` and returns.
- RU05 `onVDConnected(VEHICLE_DEVICE)` later subscribes/registers and calls `EolConfig.loadConfig()`.
- RU05 event `0xe0579` later updates `vehicle.persist.project.ext.configs` through `EolConfig.updateConfig(...)`.

## LIKELY

- Current RU02-generation root cause remains a T1J UI implementation omission/regression.
- The failed signed RU05-on-RU02 A/B test is now most plausibly an initialization/timing issue rather than an ID/bit/property mismatch.
- Concrete candidate sequence: first RU05 binding evaluation occurs before asynchronous VehicleDevice connection/load completes; `mCarConfig1` is empty so `getConfig(50)==0`; binding sets Fragrance `INVISIBLE`; correct config arrives later but may not trigger a generated-binding re-evaluation.
- If this one-shot-binding hypothesis is proven, a non-patching workaround may be possible by keeping the RU05 process alive until config load completes and then recreating/reinflating the HVAC UI without killing the process.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit in the current RU02-generation path.
- Current HVAC uses a different Fragrance config ID.
- Fragrance was removed from current HVAC.
- Old HVAC APK alone solves the problem.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` controls visibility.
- Current hidden suppression is in VehicleDevice transport, FragrancePresenter, or FragranceModel.
- ID64 is already proven as the missing gate.
- Current T1J binding contains a hidden Fragrance visibility setter.
- A scanned static RU02 HVAC overlay unhides the entry.
- RU05 has the same T1J GONE/missing-binding defect.
- RU05 uses another Fragrance config ID instead of 50.
- RU05 uses a different config1 byte/bit for Fragrance; it also uses byte12 bit6.
- RU05 adds a second T1H/PHEV/market/telematics/project gate after config50.
- A RU05-vs-RU02 Fragrance mapping mismatch explains the old A/B failure.

## OPEN

- Is RU05 `OfflineConfigManager.e()` evaluated only during initial generated-binding dirty state, with no rebind trigger from later `EolConfig.updateConfig()`?
- Where exactly is RU05 `CarConfigUtil.init(Context)` called relative to HVAC Activity/view creation?
- Can RU05 process be prewarmed or HVAC UI/task recreated after config load, without process death, to expose the OEM Fragrance button via ADB only?
- Did runtime class loading use the embedded RU05 config/VDBus classes or an identically named parent/shared implementation? Investigate only if lifecycle/binding timing does not explain the behavior.
- Runtime dynamic overlay and exact live APK hash reconciliation remain pending vehicle return.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only/static analysis and reversible runtime tests before writes or component replacement.
