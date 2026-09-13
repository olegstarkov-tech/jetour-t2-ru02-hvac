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
- Framework-level config update chain is PROVEN: `VehicleDevice` event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.
- Exact DEX `class_def` scans across ordinary RU02 system/product/system_ext Java archives found no Java definition of `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`; `DesaySVProjectService.apk` and `SVVDSCarStateService.apk` are DISPROVEN as owners.
- Full vendor inventory proves the relevant backend is native, not a hidden Java package.
- Vendor contains `/bin/hw/com.desaysv.vehicledevice@1.0-service`, `/lib64/com.desaysv.vehicledevice@1.0.so`, `/lib64/libdesaysv_vehicledevice.so`, `libdesaysv_vehiclebus.so`, and related backend libraries.
- `libdesaysv_vehicledevice.so` exports project-config handlers including `VehicleHal::setProjectExtConfigs`, `requestProjectConfigs`, `onVehiclePropertyConfigChange`, and `setVehiclePropertyConfigCallback`.
- `VehicleHal::setProjectExtConfigs(key,value)` suppresses empty/unchanged values, stores changed values, updates EOL cache state and forwards the same key/value to the registered callback.
- `VehicleHal::onVehiclePropertyConfigChange(key,value)` logs and forwards the same pair; no country/project/market/telematics gate is present in that forwarding function.
- Embedded `.gnu_debugdata` decompresses to an unstripped AArch64 mini-ELF with the same BuildID as the service.
- Mini-debug names `0x7b28` exactly as `VehicleDeviceVDS::onVehiclePropertyConfigChange(const std::string&, const std::string&)` and reports function size 252 bytes.
- Mini-debug names `0x7c24` exactly as a `non-virtual thunk to VehicleDeviceVDS::onVehiclePropertyConfigChange(...)`; it is not a second semantic callback implementation.
- Mini-debug identifies `0x11040` as `vtable for ...::VehicleDeviceVDS`, size 336 bytes.
- In the real callback `x20` is original `proKey`, `x19` is original `proValue`.
- The event object is built on the stack; event ID `918905` (`0x000E0579`) is stored at `sp+4`; a `VehicleBusBundle` is constructed at `sp+0x10`.
- First bundle insertion is `VehicleBusBundle::putString(global_0x135a0, proKey)`; second is `VehicleBusBundle::putString(global_0x135b8, proValue)`. Thus payload values are exactly the upstream callback key/value pair.
- Mini-debug did not expose names for global `std::string` bundle-key objects `0x135a0` and `0x135b8`; their literal field names remain unresolved but are no longer a root-cause blocker.
- Final publication call is PROVEN: callback loads the object's vptr, loads virtual slot `+0x38`, and executes `blr`; the primary address point is `0x11050`, so the slot is `0x11088`; ELF relocation `0x11088` is exactly `VehicleBusStub::publish(VehicleBusEvent const&)`.
- Therefore `blr` at `0x7bec` is definitively `VehicleBusStub::publish(event)`.
- PROVEN static transport chain: `VehicleHal(key,value) -> VehicleDeviceVDS::onVehiclePropertyConfigChange(key,value) -> event 918905 -> two-string key/value bundle -> VehicleBusStub::publish -> framework event 918905 -> VDVDeviceConfigStore -> CarConfigUtil -> EolConfig -> config50`.
- No country/project/market/telematics filter has been found in this traced transport path.

## LIKELY

- Global bundle-key strings `0x135a0` and `0x135b8` correspond to the framework bean's property-key and property-value field names. Their values are PROVEN; only literal key-name text remains unknown.
- The remaining Fragrance blocker is outside the now-closed project-config transport path and is more likely another capability/runtime state input consumed by HVAC/framework/UI.

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
- Event `918905` exists only in Java/framework space.
- The 918905 path should contain a direct `bl VehicleBusStub::publish@plt`; it publishes through vtable slot `+0x38` mapped by relocation `0x11088`.
- `0x7c24` is a second independent publication implementation; mini-debug proves it is only a non-virtual thunk to `0x7b28`.
- The traced VehicleDevice callback/publication bridge visibly applies a country/project/market/telematics filter before publishing the project config pair.
- Failure of project-config/event-918905 transport is the working explanation for hidden Fragrance despite current evidence showing config50=1 at HVAC startup.

## OPEN

- Which actual RU02 condition prevents Fragrance from becoming available despite config50=1?
- What Fragrance-specific runtime/capability input beyond config50 is consumed by HVAC or its backend?
- Do `AC_FRAGRANCE_DISPLAY`, `AC_FRAGRANCE_WARNING`, fragrance type/level/state IDs, or another event participate in visibility/activation rather than simple control state?
- Why does old RU05 HVAC also fail on the RU02 system base?
- Which local CarInfo/HVAC artifacts are byte-exact with the live canonical vehicle?
- Literal initialized names of bundle-key globals `0x135a0` / `0x135b8` remain unresolved, but this is documentation-only unless contradictory evidence appears.

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
