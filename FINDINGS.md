# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- `vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` returns `(mCarConfig1[12] >> 6) & 1`.
- `vehicle.persist.project.ext.configs` is the source for `mCarConfig1`.
- Current HVAC has been observed starting with ext config containing `C5` at byte12.
- Current HVAC Fragrance gate is `getConfig(50)==1 && !isT1H_PHEV()`.
- Current HVAC contains Fragrance code/resources; RU06 -> RU02-labeled HVAC deltas do not explain hidden Fragrance.
- Older signed RU05 HVAC executed on RU02 system but did not restore Fragrance; replacing HVAC alone is not a solution.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`; investigated binding/code did not reveal a visibility path, and `a2(fragranceBtn,z)` only controls enabled/clickable state.
- RU05 -> RU06 is a real packaging/architecture generation boundary; older apps expose more embedded VDBus/carconfig implementation symbols.
- RU02 `system.img` SHA-256 is `19ac46038cd8662891a8da6319844b31b0cadaf5e82e156108a91b474771265d`.
- RU02 `system_ext.img` SHA-256 is `1b9902976b45277875d4a9b79d49fa4a26599dc9cdb682459c5fa1acfe47582b`.
- RU02 `vendor.img` SHA-256 is `b1e7e189033a7d4347b2c730263d535955b783cb48a1a3226b6f0e8c4a9ef283`.
- `/system/framework/vdbus.jar` SHA-256 is `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`; `/system/framework/vdbus_extra.jar` SHA-256 is `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`.
- `CarConfigUtil.init()` subscribes to VehicleDevice event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`).
- On event `918905`, payload is decoded through `VDVDeviceConfigStore`; key `vehicle.persist.project.ext.configs` updates EOL config via `EolConfig.updateConfig(...)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; no extra Fragrance-specific gate exists there.
- `VDServiceDef` identifies event producer `com.desaysv.ivi.vds.vdev.service.VehicleDevice`; `VehicleService` is separately HAL-facing.
- Framework-level config update chain is PROVEN: `VehicleDevice` event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.
- Exact DEX `class_def` scan checked 88 RU02 APKs across `system/app`, `system/priv-app`, `product/app`, and `product/priv-app`; zero exact VehicleDevice owners found.
- Expanded exact scan across `system/framework`, `product/framework`, `system_ext/framework`, and `system_ext` app/priv-app paths checked 102 Java archives; zero exact owners found.
- Full recursive vendor inventory shows only 2 APKs, 0 JARs, 1 ODEX, 1 VDEX, 0 OAT and 0 APEX; there is no hidden vendor Java implementation population to explain VehicleDevice ownership.
- Vendor contains a dedicated native VehicleDevice stack: `/bin/hw/com.desaysv.vehicledevice@1.0-service`, `/etc/init/com.desaysv.vehicledevice@1.0-service.rc`, `/lib64/com.desaysv.vehicledevice@1.0.so`, `/lib64/libdesaysv_vehicledevice.so`, plus `libdesaysv_vehiclebus.so` and backend libraries.
- The native VehicleDevice service executable directly links `libdesaysv_vehiclebus_backend_aidl.so`, `libdesaysv_vehiclebus_backend_aosp.so`, `libdesaysv_vehiclebus.so`, `libdesaysv_vehicledevice.so`, `com.desaysv.vehicledevice@1.0.so`, and `android.hardware.automotive.vehicle@2.0.so`.
- The service executable imports `VehicleBusStub::get/set/bulkGet/publish/subscribe/unsubscribe` and `VehicleBusBundle` accessors, proving the native service directly participates in the VehicleBus publish/subscribe layer.
- The service binary contains `vdev.service.VehicleDevice|vehiclebus` and `VehicleDeviceVDS::onVehiclePropertyConfigChange proKey = %s, proValue = %s`.
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, `setVehiclePropertyConfigCallback`, and related project-config handlers.
- `VehicleHal::setProjectExtConfigs(key,value)` compares against the old value, ignores empty/unchanged updates, writes changed values to `VehicleConfigStore`, updates EOL cache state for ext-config keys, then forwards the same key/value through the registered callback.
- `VehicleHal::onVehiclePropertyConfigChange(key,value)` only logs and forwards the same key/value to the callback; no country/project/market/telematics gate is present in that forwarding function.
- `libdesaysv_vehicledevice.so` contains the persistent project keys `vehicle.persist.project.ext.configs` through `ext.configs9`, project code/PN/phonelink keys, `countryCode`, and `ro.sys.ivi.eol.country.code`.
- Targeted service disassembly proves native construction of event ID `918905` (`0x000E0579`) at two sites: `0x7ba8/0x7bb0` and `0x7ca4/0x7cac`.
- In the first 918905 block, `VehicleBusBundle::putString(...)` is called at `0x7bc4` and `0x7bd8`; in the second it is called at `0x7cc0` and `0x7cd4`. Therefore the service-side 918905 path constructs a two-string bundle.
- The two 918905 sites are in the same local service region as xrefs to the exact `VehicleDeviceVDS::onVehiclePropertyConfigChange` log string. Therefore event `918905` is constructed in native VehicleDevice service glue, not only represented as a Java framework constant.
- `readelf -rW` shows `VehicleBusStub::publish(VehicleBusEvent const&)` through an `R_AARCH64_ABS64` relocation at `0x11088`, while `VehicleBusStub::set(...)` has an ordinary `R_AARCH64_JUMP_SLOT` relocation. This explains why no direct `publish@plt` call appears in the narrow disassembly and strongly indicates an indirect function-pointer/table call path.

## LIKELY

- The two strings inserted into the 918905 bundle are the upstream property key and property value, matching the `ISVPVehiclePropertyConfigCallback(key,value)` input and the framework-side `VDVDeviceConfigStore` consumer. Exact argument/string-key decoding is still required before upgrading this to PROVEN.
- The final `VehicleBusStub::publish()` call is indirect through a data/function-pointer slot associated with relocation address `0x11088`; exact code xref is not yet proven.

## DISPROVEN

Do not reopen without new contradictory evidence:

- Engineering writes the wrong Fragrance bit.
- New HVAC uses a different config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC APK alone solves the problem.
- T1H is the active vehicle branch for this T1J bench.
- `a2(fragranceBtn,z)` is the visibility gate.
- `OfflineConfigManager.h()` is a Fragrance predicate; it is ionizer/config91.
- RU06 -> RU02 HVAC DEX/manifest/resource deltas contain the missing-Fragrance gate.
- `CarConfigUtil.getConfig(50)` applies another hidden Fragrance-specific gate after `EolConfig`.
- Raw DEX string presence is enough to identify the VehicleDevice implementation APK.
- `DesaySVProjectService.apk` implements VehicleDevice.
- `SVVDSCarStateService.apk` implements VehicleDevice.
- Any scanned ordinary APK/JAR in RU02 `system`, `product`, or `system_ext` defines VehicleDevice.
- `/vehicle` contains the Java VehicleDevice implementation.
- Vendor needs another broad APK/JAR search; recursive inventory shows the backend is native.
- Event `918905` exists only in Java/framework space; native service code explicitly materializes the same numeric ID.
- The 918905 path should contain a direct `bl VehicleBusStub::publish@plt`; relocation evidence instead points to an indirect reference.

## OPEN

- Which RU02 native condition prevents Fragrance from becoming available despite config50=1?
- What exact service function/basic block owns the 918905/two-`putString` path?
- What are the two bundle field names and which arguments are the property key/value?
- Which code xref uses the `VehicleBusStub::publish` ABS64 relocation at `0x11088`, and does the 918905 callback reach it directly or through a wrapper/table?
- Are country/project/product IDs, telematics, capability, or another condition used later in this exact service-side publication path?
- Why does old RU05 HVAC also fail on the RU02 system base?
- Which local CarInfo/HVAC artifacts are byte-exact with the live canonical vehicle?

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
