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
- RU06 vs RU02-labeled HVAC is extremely close: 2948 identical ZIP entries; the confirmed DEX delta is ionizer/config91-specific; decoded manifest is semantically identical; decoded resource changes are localization-only. Therefore the observed RU06 -> RU02 HVAC APK delta provides no mechanism explaining hidden Fragrance.
- Earlier live CarInfo `versionName` matches the local artifact labeled RU06, so live APK hash identity remains to be reconciled when the vehicle is available.
- RU05 HVAC/CarInfo expose a larger embedded VDBus/carconfig implementation set than RU06/RU02-generation artifacts, proving a packaging/architecture generation boundary.
- RU02 OTA payload contains Android `system`, `system_ext`, `product`, `vendor` and separate `system_qnx`.
- RU02 Android `system.img`: 967962624 bytes, SHA-256 `19ac46038cd8662891a8da6319844b31b0cadaf5e82e156108a91b474771265d`.
- `/system/framework/vdbus.jar`: SHA-256 `bdf017b219e4d940c17ea5d142bad7752bdf006bc28a82759259df2545c76de3`.
- `/system/framework/vdbus_extra.jar`: SHA-256 `48c9eae627c738a08ff06086d2784162ae3859ad644d6e9a401f42c266356bea`.
- `CarConfigUtil.init()` subscribes to `PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE` event `918905` from `ServiceType.VEHICLE_DEVICE` and loads EOL config.
- On event `918905`, `CarConfigUtil` decodes `VDVDeviceConfigStore`; key `vehicle.persist.project.ext.configs` calls `EolConfig.updateConfig(Utils.stringToByte(value), null, null, null, null)`.
- `CarConfigUtil.getConfig(int)` directly delegates to `EolConfig.getJetourEolConfig(int)`; no additional Fragrance-specific suppression exists there.
- `VDServiceDef` identifies the service as package `com.desaysv.ivi.vds.vdev`, class `com.desaysv.ivi.vds.vdev.service.VehicleDevice`, while `VehicleService` is separately `com.desaysv.ivi.vds.vehicle.service.VehicleService` under `android.hardware.automotive.vehicle@2.0-service`.
- Framework-level chain is PROVEN: `VehicleDevice` event 918905 -> `VDVDeviceConfigStore` -> `CarConfigUtil` -> `EolConfig.updateConfig()` -> `getConfig(50)` -> HVAC Fragrance predicate.
- Two candidate APKs were extracted and fingerprinted:
  - `DesaySVProjectService.apk` SHA-256 `2a172f9aa1a447df8ad32a733680843bfb5f131ca5c7db8dcbc507db49dd7282`, package `com.desaysv.ivi.vds.projection`.
  - `SVVDSCarStateService.apk` SHA-256 `52b3d5f63922031be273746cc3ac553c19a2d49a9508028aa81305c9988c0e1d`, package `com.desaysv.ivi.vds.carstate`.
- Exact decompiled source-tree verification proves neither candidate owns `com.desaysv.ivi.vds.vdev.service.VehicleDevice`:
  - `DesaySVProjectService.apk` contains no `com/desaysv/ivi/vds/vdev` source subtree and no `class VehicleDevice` definition.
  - `SVVDSCarStateService.apk` contains no `com/desaysv/ivi/vds/vdev` source subtree and no `class VehicleDevice` definition; it contains client-side `VehicleDeviceManager` only.

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
- Raw DEX presence of the string `com.desaysv.ivi.vds.vdev.service.VehicleDevice` identifies the implementation APK.
- `DesaySVProjectService.apk` is the VehicleDevice implementation package.
- `SVVDSCarStateService.apk` is the VehicleDevice implementation package.

## OPEN

- Which RU02 package/JAR actually defines `com.desaysv.ivi.vds.vdev.service.VehicleDevice`?
- Which RU02 system/backend condition prevents Fragrance from becoming available despite config50=1?
- Why does old RU05 HVAC also fail on the RU02 system base?
- Which local CarInfo/HVAC artifacts are byte-exact with the live canonical vehicle?
- Does RU02 runtime resolve VDBus/carconfig classes from system framework/shared libraries rather than bundled copies in old RU05 HVAC?
- How does the real `VehicleDevice` implementation obtain/publish `vehicle.persist.project.ext.configs` through event 918905, and are there project/market/telematics/capability gates there?
- Does `VehicleService` or another upstream component transform/filter the relevant capability before `VehicleDevice` publishes it?

## Constraints

- No Desay/platform private signing key is available; modified system APK deployment is not a valid primary route.
- Prefer read-only runtime/system comparison before any write/injection experiment.
