# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- `ID_CAR_CONFIG_FRAGRANCE = 50`; current RU02 `EolConfig` maps it from `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- Current RU02-generation HVAC Fragrance predicate is `getConfig(50)==1 && !isT1H_PHEV()`.
- VehicleDevice/event-918905 transport is statically closed end-to-end to `CarConfigUtil -> EolConfig -> config50`.
- Current RU02-generation T1J `bottom_layout.xml` declares `fragrance_btn` `GONE`; its `BottomLayoutBindingImpl` has no Fragrance visibility setter and no Fragrance predicate call.
- Current T1J `view/b` uses the Fragrance predicate only for FragranceDialog initialization.
- No scanned static RU02 overlay targets `com.desaysv.svhvac`.
- Therefore the current RU02-generation T1J path has no proven `config50 -> fragrance_btn VISIBLE` path.
- Exact signed RU05 HVAC artifact is `SVHvac_RU05.apk`, SHA-256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.
- RU05 T1J `bottom_layout.xml` does not initially hide `fragrance_btn`.
- RU05 `BottomLayoutBindingImpl` explicitly controls `fragranceBtn.setVisibility(...)` from `OfflineConfigManager.e()`.
- RU05 `OfflineConfigManager.e()` is exactly `isFragranceExist` and computes only `CarConfigUtil.getConfig(50)==1`; no secondary gate exists.
- RU05 APK itself contains `CarConfigUtil`, `EolConfig`, `ReserveConfigConstants`, related carconfig classes, VDBus classes, and `VDEventVehicleDevice`.
- RU05 packaged `CarConfigUtil.getConfig(int)` directly calls packaged `EolConfig.getJetourEolConfig(int)`.
- RU05 packaged config stack contains `vehicle.persist.project.ext.configs`, `...configs2`, `...configs3`, combo config, and callback handling for project vehicle-property config updates.
- RU05 packaged `VDEventVehicleDevice` defines `PROJECT_RESERVE_CONFIGS=0xe0006` and `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE=0xe0579` (918905).

## LIKELY

- Current RU02-generation root cause remains a T1J UI implementation omission/regression.
- RU05-on-RU02 failure is a separate older config-stack initialization/class-resolution issue, not the same UI defect.
- Because RU05 carries its own complete carconfig/VDBus implementation, the next likely discriminator is RU05 `EolConfig` decoding and startup-load timing rather than another APK component replacement.

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
- RU05 adds a second T1H/PHEV/market/telematics/project gate after config50.

## OPEN

- Does RU05 `EolConfig.getJetourEolConfig(50)` map to config1 byte12 bit6 exactly like RU02?
- How exactly does RU05 `EolConfig.loadConfig()` obtain/populate config1 at startup, and can timing/default data explain a false `getConfig(50)` on RU02?
- Did runtime class loading use the embedded RU05 classes or a same-named parent/shared implementation? Only investigate if EolConfig semantics otherwise match.
- Is any runtime/dynamic overlay active on the live vehicle? Pending vehicle return.
- Which local HVAC/CarInfo artifacts are byte-exact with the live canonical vehicle? Pending vehicle return.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only firmware/runtime comparison before any write/injection experiment.
