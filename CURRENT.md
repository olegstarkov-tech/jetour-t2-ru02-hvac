# RU02-HVAC CURRENT

Status: active
Canonical branch: `main`
Recovery marker: `RU02-HVAC-CONTINUE`

## Bench

- Jetour T2 / T1J
- Official Russian dealer vehicle
- Desay SV 8155
- Dealer RU firmware 00.00.02
- Telematics present

This bench is distinct from Desay 00.00.08 / D08 work.

## Current mission

Determine why OEM Fragrance / Aromatization does not appear/work even when vehicle configuration enables it, and identify the actual RU02 system mechanism controlling availability.

## Latest PROVEN state

### Config / HVAC path

- Engineering Fragrance 0/1 changes `config1` byte12: `0x85` OFF -> `0xC5` ON; delta `0x40` = bit6.
- `/system/framework/vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` maps Fragrance as `(mCarConfig1[12] >> 6) & 1`.
- `mCarConfig1` source is `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed reading ext config with byte12=`C5`.
- Current HVAC `OfflineConfigManager.f()` is `getConfig(50)==1 && !isT1H_PHEV()`.
- Current HVAC contains Fragrance classes/resources/UI logic. RU06 vs RU02-labeled HVAC DEX/manifest/resource differences do not explain missing Fragrance.
- Older correctly signed RU05 HVAC executed on RU02 system base but still did not restore Fragrance; APK replacement alone is not a solution.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`; investigated binding/code has no proven visibility path. `a2(fragranceBtn,z)` controls enabled/clickable only.
- The local RU02-labeled HVAC artifact used for the current static runtime-input audit is verified by SHA-256: `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a` (`SVHvac_RU02_2026.apk`). This verifies artifact identity against the existing local hash ledger, not yet against a fresh live pull from the canonical vehicle.

### Framework backend path

- `vdbus.jar` SHA-256 `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`.
- `vdbus_extra.jar` SHA-256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`.
- `CarConfigUtil.init()` subscribes to VehicleDevice event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`).
- Event `918905` payload is decoded via `VDVDeviceConfigStore`; key `vehicle.persist.project.ext.configs` causes `EolConfig.updateConfig(...)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; there is no extra Fragrance-specific gate there.
- `VDServiceDef` identifies producer/service `com.desaysv.ivi.vds.vdev.service.VehicleDevice` in package `com.desaysv.ivi.vds.vdev`.
- `VehicleService` is separately HAL-facing: `com.desaysv.ivi.vds.vehicle.service.VehicleService` under `android.hardware.automotive.vehicle@2.0-service`.
- PROVEN framework chain: VehicleDevice event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.

### Ownership localization

- Exact DEX `class_def` scan across 88 APKs in `system/app`, `system/priv-app`, `product/app`, `product/priv-app` found zero definitions of `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Expanded exact scan across `system/framework`, `product/framework`, `system_ext/framework`, and `system_ext` app/priv-app locations checked 102 archives and also found zero exact owners.
- `DesaySVProjectService.apk` and `SVVDSCarStateService.apk` were explicitly decompiled and DISPROVEN as VehicleDevice owners.
- Full recursive `vendor.img` inventory proves the relevant backend is native, not a hidden Java package.

### Native RU02 VehicleDevice path

RU02 vendor contains a dedicated native Desay VehicleDevice stack:

- `/etc/init/com.desaysv.vehicledevice@1.0-service.rc`;
- `/bin/hw/com.desaysv.vehicledevice@1.0-service`;
- `/lib64/com.desaysv.vehicledevice@1.0.so`;
- `/lib64/libdesaysv_vehicledevice.so`;
- related `libdesaysv_vehiclebus.so`, `libdesaysv_vehiclebus_backend_aidl.so`, `libdesaysv_vehiclebus_backend_aosp.so`, `libproperties_vehicle.so`, and Android Automotive Vehicle HAL libraries.

Static inspection proves:

- native service links the Vehicle HAL and Desay VehicleBus layers;
- service imports `VehicleBusStub::get`, `set`, `bulkGet`, `publish`, `subscribe`, `unsubscribe` and `VehicleBusBundle` accessors;
- `libdesaysv_vehicledevice.so` exports `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, and `setVehiclePropertyConfigCallback`;
- `VehicleHal::setProjectExtConfigs(key,value)` compares against the stored value, ignores empty/unchanged updates, writes changed project config to `VehicleConfigStore`, updates EOL caches for ext-config keys, then invokes the registered property-config callback with the same key/value;
- `VehicleHal::onVehiclePropertyConfigChange(key,value)` only logs and forwards the same key/value through the registered callback; no country/project/market/telematics condition was observed in that forwarding function;
- `libdesaysv_vehicledevice.so` contains `vehicle.persist.project.ext.configs` through `ext.configs9`, project code/PN/phonelink keys, `countryCode`, and `ro.sys.ivi.eol.country.code`.

### Event 918905 native publication bridge — CLOSED

Targeted disassembly plus the embedded `.gnu_debugdata` mini-ELF closes the native project-config publication path.

Mini-debug symbols prove:

- `0x7b28` = `com::desaysv::vehicledevice::V1_0::implementation::VehicleDeviceVDS::onVehiclePropertyConfigChange(const std::string&, const std::string&)`;
- `0x7c24` = `non-virtual thunk to VehicleDeviceVDS::onVehiclePropertyConfigChange(...)`, so it is not a second semantic implementation;
- `0x11040` = `vtable for ...::VehicleDeviceVDS` (336 bytes).

Inside the real callback at `0x7b28`:

- callback arguments are preserved as `x20 = proKey` and `x19 = proValue`;
- a `VehicleBusEvent`/embedded `VehicleBusBundle` is built on the stack;
- event ID `918905` (`0x000E0579`) is written at `sp+4`;
- first insertion is `VehicleBusBundle::putString(global_0x135a0, proKey)`;
- second insertion is `VehicleBusBundle::putString(global_0x135b8, proValue)`;
- then the object vptr is loaded, virtual slot `+0x38` is loaded, and `blr` is executed with the event pointer.

Publication target is exact:

- Itanium-style vtable symbol starts at `0x11040`; primary address point used by the object is `0x11050`;
- `0x11050 + 0x38 = 0x11088`;
- ELF relocation at `0x11088` is exactly `R_AARCH64_ABS64 VehicleBusStub::publish(VehicleBusEvent const&)`;
- therefore the indirect `blr` at `0x7bec` is definitively `VehicleBusStub::publish(event)`.

The mini-debug pass did not expose symbol names for global `std::string` bundle-key objects at `0x135a0` and `0x135b8`. Their payload values are nevertheless PROVEN to be the original callback property key and value. Resolving the literal bundle field names is now documentation-only, not a root-cause blocker.

PROVEN end-to-end static transport chain:

`VehicleHal::onVehiclePropertyConfigChange(key,value) -> VehicleDeviceVDS::onVehiclePropertyConfigChange(key,value) -> VehicleBusEvent.id=918905 -> bundle(globalA,key)+bundle(globalB,value) -> VehicleBusStub::publish(event) -> framework VDBus event 918905 -> VDVDeviceConfigStore -> CarConfigUtil -> EolConfig -> config50`.

No country/project/market/telematics filter has been found in this traced transport path itself.

### Current HVAC runtime-input audit

The first broad static audit of the verified RU02-labeled HVAC artifact produced these narrow observations:

- no semantic reference named `FragranceDisplay` or `FragranceWarning` was found in the generated audit, and the explicit numeric candidate section for IDs 58/59/60/61/62/64/66/71/76/88 was empty;
- this does NOT prove those states are unused because the real presenter/backend classes are obfuscated and the grep-oriented audit was biased toward files already containing Fragrance text;
- visible Fragrance UI callbacks found in `HvacContentView` / `T1HHvacActivity` handle power state by selecting `fragranceBtn` (`setSelected`) rather than changing its visibility;
- `FragranceDialog` callbacks handle power, level, type, remaining amount and position/installed-state behavior; these are operational state handlers and no evidence from this audit ties them to making the main `fragrance_btn` VISIBLE;
- `FragranceDialog` uses presenter type `b.a.d.a.b.x0`; activity/view code labels the same object `mFragrancePresenter` and calls methods such as `l()`, `n()`, `o()`, `p()`, `q()`, `u()`, `w()`, `x()`.

Therefore the next static target is the obfuscated `b.a.d.a.b.x0` FragrancePresenter and its callback/subscription interfaces, not a preselected assumption that `AC_FRAGRANCE_DISPLAY` is the missing gate.

## DISPROVEN / closed unless new evidence

- Engineering writes the wrong Fragrance bit.
- New HVAC uses another config ID instead of 50.
- Fragrance code/resources were removed from current HVAC.
- Installing old HVAC alone solves the issue.
- T1H is the active branch for this T1J bench.
- `a2(fragranceBtn,z)` controls visibility.
- RU06 -> RU02 HVAC DEX/manifest/resource differences contain the missing-Fragrance gate.
- `OfflineConfigManager.h()` is a Fragrance predicate; it is ionizer/config91.
- `CarConfigUtil` contains a separate Fragrance-specific suppression after `getConfig(50)`.
- Raw DEX string presence identifies the VehicleDevice implementation APK.
- `DesaySVProjectService.apk` or `SVVDSCarStateService.apk` implements VehicleDevice.
- Any scanned ordinary APK/JAR in RU02 `system`, `product`, or `system_ext` defines VehicleDevice.
- `/vehicle` contains the Java VehicleDevice implementation.
- Vendor needs another broad APK/JAR search; the relevant backend is native.
- Event `918905` exists only in the Java framework.
- The 918905 path should contain a direct `bl VehicleBusStub::publish@plt`; publish is reached through vtable slot `+0x38` / relocation `0x11088`.
- `0x7c24` is a second independent callback implementation; mini-debug proves it is a non-virtual thunk to `0x7b28`.
- The traced VehicleDevice callback/publication bridge applies a country/project/market/telematics filter before publishing changed project config; traced code is direct packaging and publish.
- Missing Fragrance should be attributed to failure of the `vehicle.persist.project.ext.configs` / event-918905 transport path without new contradictory evidence.

## Current open question

The project-config transport mechanism is statically closed. The active question is now:

**What other RU02 Fragrance availability/capability input keeps the OEM UI hidden even though config50 is correctly delivered and readable as 1?**

The immediate sub-question is: what does `b.a.d.a.b.x0` subscribe to/read, and does it expose an additional availability/display state separate from ordinary fragrance power/type/level/remain state?

## Next step

Trace the current RU02 HVAC `b.a.d.a.b.x0` FragrancePresenter end-to-end:

1. inspect the complete `x0.java` source and identify superclass/interfaces/callback registrations;
2. map public methods `l()/n()/o()/p()/q()/u()/w()/x()` and any other methods to underlying CarInfo/HVAC/VDBus reads/writes;
3. enumerate every event/property ID or proxy method used by x0, including registrations and initial reads;
4. inspect x0 callback interface implementations/consumers to determine whether any state controls availability/visibility rather than operational status;
5. only after a concrete candidate predicate is found, trace its producer into CarInfo/VDBus/native backend.

Do not return to broad APK guessing, OAT/VDEX hunting, blind property writes, or unsigned HVAC patching. Do not treat `AC_FRAGRANCE_DISPLAY` as root cause without a concrete x0/callback consumer path.

When the vehicle becomes available again, pull/hash live HVAC/CarInfo APKs and reconcile artifact identity before any runtime experiment that depends on exact APK identity.
