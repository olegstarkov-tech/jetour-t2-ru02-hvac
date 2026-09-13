# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- `vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` returns `(mCarConfig1[12] >> 6) & 1` from `vehicle.persist.project.ext.configs`.
- Current HVAC has been observed starting with ext config containing `C5` at byte12.
- Current HVAC Fragrance predicate is `getConfig(50)==1 && !isT1H_PHEV()`.
- Older signed RU05 HVAC executed on RU02 but did not restore Fragrance; replacing HVAC alone is not a solution.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`; investigated binding/code has no proven visibility path; `a2(fragranceBtn,z)` only controls enabled/clickable.
- Framework chain is PROVEN: VehicleDevice event `918905` -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC predicate.
- Ordinary Java owner search is negative across scanned `system`, `product`, `system_ext`, and vendor APK/JAR containers; the relevant RU02 VehicleDevice backend is native.
- RU02 vendor contains `/bin/hw/com.desaysv.vehicledevice@1.0-service`, `/lib64/libdesaysv_vehicledevice.so`, HIDL `com.desaysv.vehicledevice@1.0.so`, and Desay VehicleBus libraries.
- The service executable imports `VehicleBusStub::get/set/bulkGet/publish/subscribe/unsubscribe` and contains `vdev.service.VehicleDevice|vehiclebus`.
- The service registers `SVPVehicleDevice::setVehiclePropertyConfigCallback(ISVPVehiclePropertyConfigCallback*)` and contains log string `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s`.
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, `setVehiclePropertyConfigCallback`, and related project-config handlers.
- `VehicleHal::setProjectExtConfigs(key,value)` reads the old value, ignores empty input, ignores unchanged values, otherwise updates `VehicleConfigStore::setProjectConfig(key,value)`, updates EOL config slots 1-4 for the first four `vehicle.persist.project.ext.configs*` keys, then invokes the registered callback with `(key,value)`.
- `VehicleHal::onVehiclePropertyConfigChange(key,value)` logs and directly forwards `(key,value)` to the registered callback virtual method when present.
- No country/project/market/telematics/Fragrance-specific branch exists in those two traced HAL functions.
- `countryCode` and `ro.sys.ivi.eol.country.code` do exist elsewhere in `libdesaysv_vehicledevice.so`, so country handling exists in the native layer but is not the gate in these two functions.
- Raw 32-bit searches for `918905` and `917510` in the first native targets were negative; this does not disprove indirect event construction.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC APK alone solves the problem.
- T1H is the active vehicle branch.
- `a2(fragranceBtn,z)` is the visibility gate.
- `OfflineConfigManager.h()` is Fragrance; it is ionizer/config91.
- RU06 -> RU02 HVAC DEX/manifest/resource deltas contain the missing-Fragrance gate.
- `CarConfigUtil.getConfig(50)` applies another hidden Fragrance-specific gate.
- Raw DEX string presence identifies a VehicleDevice APK owner.
- `DesaySVProjectService.apk` or `SVVDSCarStateService.apk` implements VehicleDevice.
- Any scanned ordinary APK/JAR implements the RU02 VehicleDevice backend.
- `VehicleHal::setProjectExtConfigs()` or `VehicleHal::onVehiclePropertyConfigChange()` applies a country/project/market/telematics filter before forwarding a changed non-empty config value.

## OPEN

- What does service-side `VehicleDeviceVDS::onVehiclePropertyConfigChange(key,value)` put into `VehicleBusEvent` and how does it publish it?
- Where is logical event `918905` assigned/constructed: service executable, `libdesaysv_vehiclebus.so`, backend library, or table-driven service metadata?
- Is any remaining project/market/telematics/capability filter applied in the service/VehicleBus glue layer?
- Why does old RU05 HVAC also fail on RU02 system base?
- Which local HVAC/CarInfo artifacts are byte-exact with the live canonical vehicle?

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
