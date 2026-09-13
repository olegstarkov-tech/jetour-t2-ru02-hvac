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
- RU05 `getJetourEolConfig(50)` is exactly `(mCarConfig1[12] >> 6) & 1`, identical to current RU02.
- RU05 `EolConfig.loadConfig()` requests `vehicle.persist.project.ext.configs` through VDBus `getOnce(0xe0006)` and stores parsed response `value` into `mCarConfig1`; absent response/payload falls back to empty string.
- RU05 `CarConfigUtil.init(Context)` loads config immediately only if VehicleDevice is already connected; otherwise it starts `bindService()` and returns, with load deferred to `onVDConnected()`.
- Event `0xe0579` later updates `vehicle.persist.project.ext.configs` through `EolConfig.updateConfig(...)`.
- RU05 `HvacApplication.onCreate()` invokes `CarConfigUtil.init()` before later HVAC view initialization including `view/b.I0(context)`.
- RU05 manifest declares `HvacApplication` and exported `HvacService`, but no normal HVAC Activity.
- RU05 `BottomLayoutBindingImpl.onFieldChange()` always returns `false`.
- `BottomLayoutBindingImpl` does not register `EolConfig`/`CarConfigUtil` as observable dependencies.
- `EolConfig.updateConfig()` does not directly set BottomLayout dirty flags or request a rebind.
- RU05 Fragrance predicate is reevaluated only when BottomLayout binding itself is invalidated/reassigned, e.g. `invalidateAll()` or `setHvacContentView()`.
- Therefore a late config50 update does not automatically refresh existing RU05 Fragrance visibility.

## LIKELY

- Current RU02-generation root cause remains a T1J UI implementation omission/regression.
- Failed signed RU05-on-RU02 A/B test is now plausibly explained by stale initial binding state: first evaluation before async VehicleDevice/config load gives false/INVISIBLE; correct config arrives later; existing binding is not automatically reevaluated.
- This live sequence is not yet proven because `CarConfigUtil.init()` runs before view initialization and the VehicleDevice callback may have completed before first binding evaluation.
- If stock RU05 view/binding can be recreated after config load without process death, an ADB-assisted workaround may be possible without Desay signing keys.

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
- A late RU05 `EolConfig.updateConfig()` automatically refreshes BottomLayout via observable DataBinding registration.

## OPEN

- What exact RU05 service/view lifecycle creates/recreates BottomLayoutBinding?
- Can an existing stock/exported path trigger BottomLayout recreation or `setHvacContentView()` after config load while preserving the process/static `mCarConfig1`?
- Did runtime class loading use the embedded RU05 config/VDBus classes or a same-named parent/shared implementation? Investigate only if lifecycle timing fails to explain the live test.
- Runtime dynamic overlay and exact live APK hash reconciliation remain pending vehicle return.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only/static analysis and reversible runtime tests before writes or component replacement.
