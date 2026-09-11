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

Determine why OEM Fragrance / Aromatization does not appear/work even when the vehicle configuration enables it, and identify the actual system mechanism controlling availability on this RU02 bench.

## Latest PROVEN state

1. Engineering Fragrance 0/1 really changes car configuration.
2. Observed `config1` byte 12:
   - `0x85` = OFF
   - `0xC5` = ON
   - delta `0x40` = bit 6.
3. In `/system/framework/vdbus_extra.jar`:
   - `ID_CAR_CONFIG_FRAGRANCE = 50`.
4. `EolConfig.getJetourEolConfig(50)` maps Fragrance as:
   `Fragrance = (mCarConfig1[12] >> 6) & 1`.
5. `mCarConfig1` is loaded from `vehicle.persist.project.ext.configs`.
6. Current HVAC startup has been observed reading `vehicle.persist.project.ext.configs` with byte12=`C5`; therefore the Fragrance config bit reaches the current app/backend path.
7. Current HVAC contains Fragrance classes/resources/UI logic; feature code was not removed.
8. Current HVAC `OfflineConfigManager.f()` checks:
   `getConfig(50) == 1 && !isT1H_PHEV()`.
9. Vehicle is T1J; do not treat T1H paths as the active branch without evidence.
10. An older correctly signed HVAC APK was installed over the current system and actually ran from `/data/app`, but Fragrance still did not appear. Therefore replacing only HVAC APK is not a solution.
11. Modified system HVAC APK is not a practical path at present because the Desay/platform private signing key is unavailable.
12. In current `bottom_layout.xml`, `fragrance_btn` is declared `android:visibility="gone"`.
13. In investigated `view/b.java` and `BottomLayoutBindingImpl.executeBindings()` smali, no path was found that changes `fragranceBtn` to VISIBLE. The binding assigns its click listener, but does not prove a visibility change.

## Important interpretation

The config-side theory "Engineering writes the wrong Fragrance bit" is closed: the observed engineering bit and the current framework mapping match exactly.

The UI `GONE` observation is important, but is NOT by itself accepted as the root cause of the whole RU02 problem, because the older HVAC APK also failed on the current system base.

## Current open question

What system/backend or firmware-specific condition on RU 00.00.02 prevents the OEM Fragrance function from becoming available even though config50=1 is correctly present?

## Working direction

Prioritize read-only investigation of system/backend differences and runtime state:
- VehicleDevice
- VehicleService
- VDBus / vdbus_extra
- CarConfigUtil / EolConfig update semantics
- persistent vehicle properties
- firmware-specific services/framework components

Do not spend the next phase on unsigned HVAC APK patching.
