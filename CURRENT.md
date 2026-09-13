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
10. An older correctly signed RU05 HVAC APK was installed over the current RU02 system and actually ran from `/data/app`, but Fragrance still did not appear. Therefore replacing only HVAC APK is not a solution.
11. Modified system HVAC APK is not a practical path at present because the Desay/platform private signing key is unavailable.
12. In current `bottom_layout.xml`, `fragrance_btn` is declared `android:visibility="gone"`.
13. In investigated `view/b.java` and `BottomLayoutBindingImpl.executeBindings()` smali, no path was found that changes `fragranceBtn` to VISIBLE. The binding assigns its click listener, but does not prove a visibility change.
14. A three-generation signed APK comparison set is now available locally:
   - `SVHvac_RU05.apk` + `SVVDSCarInfo_RU05.apk` (older/2024 generation; RU05 HVAC is the one already tested live on RU02),
   - `SVHvac_RU06.apk` + `SVVDSCarInfo_RU06.apk`,
   - `SVHvac_RU02_2026.apk` + `SVVDSCarInfo_RU02_TEL_2026.apk` (current problematic 2026 telematics generation).
15. Static inventory shows a clear generation split:
   - RU02 HVAC: APK `209865320` bytes, DEX `6031108` bytes.
   - RU06 HVAC: APK `209832552` bytes, DEX `6031076` bytes.
   - RU05 HVAC: APK `93028104` bytes, DEX `6275412` bytes.
   - RU02 and RU06 HVAC have the same manifest size (`7188`) and `bottom_layout.xml` size (`12156`), while RU05 differs (`6456` manifest, `7620` bottom layout).
16. RU02 and RU06 `SVVDSCarInfo` have exactly the same APK size (`2789840`), DEX size (`4210872`), `resources.arsc` size (`666316`) and manifest size (`3436`), but different whole-APK SHA-256 values. RU05 CarInfo is materially different (`3019216` APK, `4926660` DEX).
17. Quick DEX string scan shows RU05 HVAC and RU05 CarInfo embed/expose many VDBus-extra/carconfig implementation symbols and constants (`VehicleDevice`, `VehicleService`, `EolConfig`, `CarConfigUtil`, `ID_CAR_CONFIG_FRAGRANCE`, multiple `ID_AC_FRAGRANCE*`, Binder interface classes). The same quick scan on RU06/RU02 mainly exposes client references such as `VDBus`/`CarConfigUtil`, not that larger embedded implementation symbol set. This proves a packaging/architecture generation change; it does NOT yet prove which runtime class implementation wins or that this is the Fragrance root cause.

## Important interpretation

The config-side theory "Engineering writes the wrong Fragrance bit" is closed: the observed engineering bit and the current framework mapping match exactly.

The UI `GONE` observation is important, but is NOT by itself accepted as the root cause of the whole RU02 problem, because the older HVAC APK also failed on the current system base.

The new RU05/RU06/RU02 triad strongly suggests an architecture/package-boundary change occurred between RU05 and RU06: RU05 APKs contain more VDBus/carconfig implementation material, while RU06 and RU02 look like the newer generation and are very close to each other structurally. This makes the system/framework/backend generation boundary a higher-priority target than further HVAC-only replacement.

## Current open question

What exact code/resource/backend difference between RU05, RU06 and RU02-TEL prevents the OEM Fragrance function from becoming available on RU02 even though config50=1 is correctly present?

## Working direction

Next read-only step: perform internal-entry hashes and focused decompile/diff of the six APKs before any live replacement experiment. In particular determine whether RU06 and RU02 CarInfo actually have identical `classes.dex`, and localize any HVAC code/resource delta touching Fragrance, VDBus, car config or availability gating.

Do not spend the next phase on unsigned HVAC APK patching.
