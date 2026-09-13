# RU02-HVAC FINDINGS

Only durable findings belong here. Labels: PROVEN / DISPROVEN / OPEN.

## PROVEN

- Bench is Jetour T2 / T1J, official RU dealer vehicle, Desay SV 8155, firmware 00.00.02, telematics present.
- Engineering Fragrance toggle changes `config1` byte12 between `0x85` and `0xC5`; delta `0x40` = bit6.
- `vdbus_extra.jar` defines `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` returns `(mCarConfig1[12] >> 6) & 1`.
- `vehicle.persist.project.ext.configs` is the source for `mCarConfig1`.
- Current HVAC has been observed starting with ext config containing `C5` at byte12.
- Current HVAC contains Fragrance code/resources.
- Current HVAC Fragrance gate is `getConfig(50)==1 && !isT1H_PHEV()`.
- Older signed RU05 HVAC really executed on the current RU02 system but did not restore Fragrance; replacing HVAC APK alone is not a solution.
- Current `bottom_layout.xml` declares `fragrance_btn` as `GONE`; investigated binding/code did not reveal a visibility path, and `a2(fragranceBtn,z)` only controls enabled/clickable state.
- RU06 vs RU02-labeled HVAC is extremely close; the only confirmed DEX behavior delta is ionizer/config91-specific, decoded manifest is semantically identical, and decoded resource changes are localization-only. No observed application delta explains missing Fragrance.
- RU06 vs RU02-labeled CarInfo generated Java differs only in `BuildConfig.VERSION_NAME`; live artifact identity remains to be reconciled when the vehicle is available.
- RU05 -> RU06 is a real packaging/architecture generation boundary; RU05 HVAC/CarInfo expose more embedded VDBus/carconfig implementation symbols than newer artifacts.
- RU02 Android `system.img` has been extracted read-only: 967962624 bytes, SHA-256 `19ac46038cd8662891a8da6319844b31b0cadaf5e82e156108a91b474771265d`.
- `/system/framework/vdbus.jar` SHA-256 is `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`; `/system/framework/vdbus_extra.jar` SHA-256 is `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`.
- `CarConfigUtil.init()` subscribes to VehicleDevice event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`).
- On event `918905`, payload is decoded through `VDVDeviceConfigStore`; key `vehicle.persist.project.ext.configs` updates EOL config via `EolConfig.updateConfig(...)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; no extra Fragrance-specific gate exists there.
- `VDServiceDef` identifies the event producer as package `com.desaysv.ivi.vds.vdev`, class `com.desaysv.ivi.vds.vdev.service.VehicleDevice`.
- `VehicleService` is separately identified as HAL-facing `com.desaysv.ivi.vds.vehicle.service.VehicleService` under package `android.hardware.automotive.vehicle@2.0-service`.
- Framework-level config update chain is PROVEN: `VehicleDevice` event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.
- Exact candidate APK extraction/decompile proved `DesaySVProjectService.apk` and `SVVDSCarStateService.apk` do not define VehicleDevice. `SVVDSCarStateService` contains only client-side `VehicleDeviceManager`.
- An exact DEX `class_def` scan checked 88 RU02 APKs across `system/app`, `system/priv-app`, `product/app`, and `product/priv-app` for descriptor `Lcom/desaysv/ivi/vds/vdev/service/VehicleDevice;`.
- Result of that exact owner scan: zero matches. Therefore none of those 88 APKs implements VehicleDevice; previous raw DEX string hits were references/shared tables, not ownership evidence.

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
- Any APK in scanned RU02 `system/app`, `system/priv-app`, `product/app`, or `product/priv-app` defines VehicleDevice; exact class-table scan across 88 APKs found none.

## OPEN

- Which RU02 system/backend condition prevents Fragrance from becoming available despite config50=1?
- Why does old RU05 HVAC also fail on the RU02 system base?
- Which local CarInfo/HVAC artifacts are byte-exact with the live canonical vehicle?
- Does RU02 runtime resolve VDBus/carconfig classes from system framework/shared libraries rather than bundled copies in old RU05 HVAC?
- Where is the actual RU02 class definition for `com.desaysv.ivi.vds.vdev.service.VehicleDevice`?
- How does VehicleDevice obtain/publish `vehicle.persist.project.ext.configs` through event 918905, and are there project/market/telematics/capability gates there?
- Does VehicleService or another upstream component transform/filter the relevant capability before VehicleDevice publishes it?

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
