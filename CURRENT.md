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

Do not mix this bench with Desay 00.00.08, AutoLink, T1H, Chinese firmware, or non-telematics cars without explicit verification.

## Mission

Determine why OEM Fragrance / Aromatization remains hidden/non-working even though vehicle configuration enables it, and identify the real RU02 activation mechanism.

## PROVEN checkpoint

### Config path

- Engineering Fragrance 0/1 changes `config1` byte12 `0x85 -> 0xC5`; delta `0x40` = bit6.
- `ID_CAR_CONFIG_FRAGRANCE = 50`.
- `EolConfig.getJetourEolConfig(50)` maps `(mCarConfig1[12] >> 6) & 1`.
- Source property is `vehicle.persist.project.ext.configs`.
- Current HVAC startup has been observed with byte12=`C5`.
- HVAC `OfflineConfigManager.f()` is exactly `getConfig(50)==1 && !isT1H_PHEV()`.
- Vehicle is T1J.

### Event-918905 transport — CLOSED

- Framework receives VehicleDevice event `918905` (`PROJECT_VEHICLE_PROPERTY_CONFIG_UPDATE`) and feeds `VDVDeviceConfigStore -> CarConfigUtil -> EolConfig`.
- RU02 backend owner is native, not a hidden Java APK/JAR.
- Vendor contains `/bin/hw/com.desaysv.vehicledevice@1.0-service` and `libdesaysv_vehicledevice.so` / VehicleBus libraries.
- Mini-debug/disassembly proves `VehicleDeviceVDS::onVehiclePropertyConfigChange(key,value)` builds event `918905`, inserts original key/value, and calls `VehicleBusStub::publish(event)`.
- No country/project/market/telematics filter was found in the traced transport path.
- Therefore missing Fragrance should not be attributed to failed `vehicle.persist.project.ext.configs` / event-918905 delivery without contradictory evidence.

### HVAC artifact / UI anchors

Static audit uses local `SVHvac_RU02_2026.apk` SHA-256:

`2819ccafd364fb46bf06c492932fdc6fb4768c75705d243ef66b0c6518ce765a`

This matches the established local RU02-labeled hash, but still requires fresh live reconciliation when the car returns.

- HVAC contains Fragrance code/resources/dialog/UI.
- Older signed RU05 HVAC actually ran on the RU02 system base but did not restore Fragrance.
- Base `bottom_layout.xml` declares `fragrance_btn` as `GONE`.
- Investigated `view/b.java` and `BottomLayoutBindingImpl` have no proven path setting that button VISIBLE.
- `a2(fragranceBtn,z)` only controls enabled/clickable state.
- Fragrance power callbacks update `fragranceBtn.setSelected(...)`, not visibility.

### FragrancePresenter / FragranceModel — CLOSED as hidden gate

`b.a.d.a.b.x0` is exactly `FragrancePresenter`.

- It contains no CarInfo/VDBus property IDs and no visibility/capability condition.
- It obtains its model from `b.a.b.a.c.e.b()` and directly delegates/fans out callbacks.

`b.a.b.a.c.e` is `ModelFactory`; `e.b()` returns `new b()` where `b.a.b.a.c.b` is compiled as `FragranceModel`.

`FragranceModel`:

- implements `IFragranceModel` and `CarInfoHelper.ISpiListener`;
- listens only to module `327690`;
- exact ID set is `{58,59,60,61,62,71,92,93,94}`;
- mapping is:
  - 58/59/60 = fragrance type 1/2/3;
  - 61 = power;
  - 62 = level;
  - 71 = position/channel;
  - 92/93/94 = cartridge remain 1/2/3;
- initial reads use exactly the same nine IDs;
- writes are only 61, 62 and 71.

Therefore IDs 64 (`AC_FRAGRANCE_DISPLAY`), 66 (`AC_FRAGRANCE_WARNING`), 76 (welcome fragrance), and 88 (happy-egg fragrance) are **not part of the actual RU02 HVAC FragranceModel subscription/read/write path**.

The traced presenter/model path contains ordinary operational state only and no second display/availability/capability input.

## DISPROVEN / closed unless new evidence

- Engineering writes the wrong Fragrance bit.
- Current HVAC uses another config ID instead of 50.
- Fragrance code/resources were removed.
- Installing old HVAC alone solves the issue.
- T1H is the active branch for this car.
- `a2(fragranceBtn,z)` controls visibility.
- `OfflineConfigManager.h()` is Fragrance; it is ionizer/config91.
- `CarConfigUtil` applies another hidden Fragrance-specific gate after config50.
- `DesaySVProjectService.apk` / `SVVDSCarStateService.apk` owns VehicleDevice.
- Project-config/event-918905 transport failure explains the missing UI.
- `AC_FRAGRANCE_DISPLAY` is already proven as the missing gate.
- `FragrancePresenter` contains the hidden visibility gate.
- The actual RU02 `FragranceModel` consumes IDs 64/66/76/88 as a second availability gate.

## Current open question

**What UI/resource activation mechanism is supposed to make `fragrance_btn` visible when config50=1, given that the base layout declares it `GONE` and the traced presenter/model path has no visibility signal?**

Primary candidates now are UI/resource-side:

- an unexamined call-site of `OfflineConfigManager.f()` controlling construction/selection;
- a qualified alternate layout/resource;
- a product/vendor/system RRO/resource overlay targeting `com.desaysv.svhvac`;
- another resource-selection path outside the already inspected view/binding code.

## Next step

Run one deterministic UI/resource activation audit:

1. enumerate every call-site of `OfflineConfigManager.f()`;
2. enumerate every `fragranceBtn` / `fragrance_btn` source reference and visibility operation;
3. decode/list every qualified `res/layout*` resource containing `fragrance_btn` and its initial visibility;
4. identify which bottom-layout resource T1J actually inflates;
5. scan extracted RU02 system/product/system_ext/vendor overlay packages for targets/resources affecting `com.desaysv.svhvac`;
6. choose between in-APK UI construction and external resource overlay only from evidence.

Do not return to broad backend-ID guessing, VehicleDevice transport, OAT/VDEX hunting, blind writes, or unsigned HVAC patching.
