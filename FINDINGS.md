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
- Embedded `.gnu_debugdata` proves the native callback/publication bridge: `VehicleDeviceVDS::onVehiclePropertyConfigChange(...)` builds event `918905`, inserts original key/value, and publishes through `VehicleBusStub::publish(event)`.
- PROVEN static transport chain: `VehicleHal(key,value) -> VehicleDeviceVDS callback -> event 918905 -> VehicleBusStub::publish -> framework event 918905 -> VDVDeviceConfigStore -> CarConfigUtil -> EolConfig -> config50`.
- Current static HVAC audit uses local `SVHvac_RU02_2026.apk` SHA-256 `2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`, matching the established local RU02-labeled artifact hash.
- `HvacContentView` Fragrance power callback changes `fragranceBtn` selected state via `setSelected(z)`; it does not make the button visible.
- `FragranceDialog` callbacks handle power, level, type, remaining amount and position/installed-state behavior; no direct main-button visibility change was identified there.
- `b.a.d.a.b.x0` is exactly `FragrancePresenter`; it contains no CarInfo/VDBus IDs and no visibility/capability branch. It directly delegates to `IFragranceModel` and forwards callbacks.
- `ModelFactory.b()` (`b.a.b.a.c.e.b()`) returns singleton `new b()` where `b.a.b.a.c.b` is compiled as `FragranceModel`.
- `FragranceModel` implements `IFragranceModel` and `CarInfoHelper.ISpiListener`.
- `FragranceModel` listens only to CarInfo module `327690` with exact command array `{58,59,60,61,62,71,92,93,94}`.
- `J0()` registers that exact set with `CarInfoHelper.listen(327690, ids)`; `K0()` initial-reads exactly the same set when CarInfo is connected.
- Exact callback mapping: 58=type1, 59=type2, 60=type3, 61=power, 62=level, 71=position/channel, 92=remain1, 93=remain2, 94=remain3.
- Exact getter mapping uses the same IDs: `h=58`, `z=59`, `P=60`, `F0=61`, `r=62`, `m0=71`, `E=92`, `H=93`, `f0=94`.
- FragranceModel writes only IDs 61 (power), 62 (level), and 71 (position/channel) with `CarInfoProxy.sendItemValue(327690, id, value)`.
- IDs 64 (`AC_FRAGRANCE_DISPLAY`), 66 (`AC_FRAGRANCE_WARNING`), 76 (welcome fragrance), and 88 (happy-egg fragrance) are not part of the actual FragranceModel subscription/read/write path.
- Therefore the traced FragrancePresenter/FragranceModel path contains ordinary operational state only and no second display/availability/capability input.

## LIKELY

- The remaining Fragrance blocker is outside the closed project-config transport and presenter/model operational-state path.
- Because the base `bottom_layout.xml` declares `fragrance_btn` as `GONE` and no proven in-code visibility setter has been found, the next high-value area is UI/resource activation: qualified layout selection, resource overlays, or an unexamined construction path using `OfflineConfigManager.f()`.

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
- Failure of project-config/event-918905 transport is the working explanation for hidden Fragrance despite config50=1.
- `AC_FRAGRANCE_DISPLAY` is already proven to be the missing visibility gate.
- `FragrancePresenter` contains the hidden visibility/capability gate.
- The actual RU02 HVAC `FragranceModel` consumes ID64/66/76/88 as a second availability/display gate.

## OPEN

- Which actual RU02 UI/resource mechanism makes or should make `fragrance_btn` visible when config50=1?
- Is there an unexamined `OfflineConfigManager.f()` call-site controlling layout construction or resource selection?
- Are there qualified alternate `bottom_layout` resources with different initial visibility?
- Is a runtime resource overlay / product/vendor overlay expected to target `com.desaysv.svhvac` or the relevant layout/resource?
- Why does old RU05 HVAC also fail on the RU02 system base?
- Which local CarInfo/HVAC artifacts are byte-exact with the live canonical vehicle?

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
