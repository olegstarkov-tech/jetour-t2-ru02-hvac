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
14. A three-generation signed APK comparison set is available locally: paired `SVHvac` + `SVVDSCarInfo` for RU05, RU06 and current-labeled RU02-TEL-2026.
15. RU05 is a distinct application/package generation. RU06 and RU02-labeled artifacts are structurally much closer.
16. Internal ZIP-entry hash comparison is complete for the key pairs.
17. HVAC RU06 -> RU02:
   - 2948 entries are byte-identical;
   - only 6 entries differ;
   - no entries are added or removed;
   - differing entries are `classes.dex`, `AndroidManifest.xml`, `resources.arsc`, and the three v1 signature files;
   - `res/layout/bottom_layout.xml` is byte-identical;
   - `res/layout/dialog_fragrance_layout.xml` is byte-identical;
   - `res/layout/fragrance_type_item.xml` is byte-identical.
18. `classes.dex` differs only slightly in size for HVAC RU06 vs RU02: `6031076` -> `6031108` bytes.
19. Focused JADX source-tree diff of RU06 vs RU02-labeled HVAC found exactly one differing generated Java file: `com/desaysv/svhvac/view/b.java`.
20. The only generated-Java delta in that file is method `K1(boolean)`: RU02-labeled code wraps the existing `ionAnimation` update block in an additional guard `com.desaysv.svhvac.h.a.a().h()`. The actual ion-animation operations are otherwise unchanged (`setVisibilityAndRunOn`, `setDrawables(R.array.blow_mode_ion_array)`).
21. `com.desaysv.svhvac.h.a` is `OfflineConfigManager`. The RU06 and RU02-labeled decompiled `OfflineConfigManager.java` files are byte-identical (SHA-256 `d4419f35e81def8fcd7739f927e46a115e366b32d0b54d1b3b0b42365fa047c1`).
22. `OfflineConfigManager.h()` is definitively the ionizer-existence gate: it returns `CarConfigUtil.getDefault().getConfig(91) == 1` and logs `isIonExist`.
23. `OfflineConfigManager.f()` remains the distinct Fragrance gate: `getConfig(50)==1 && !isT1H_PHEV()`.
24. The call sites reinforce this semantic classification: `h()` is used by `onIonClick`, logs `ion is no exist`, and by `isIonExistShow` UI logic together with PM2.5 support checks.
25. Direct apktool/smali diff independently confirms the same `K1(boolean)` delta: RU02 adds only `OfflineConfigManager.a().h()` followed by early `return-void` when false, before the unchanged `ionAnimation` block. The extracted `h()Z` smali itself has no RU06/RU02 diff. Therefore the only confirmed HVAC DEX behavior change between these two artifacts is ionizer/config91-specific and is not the Fragrance/config50 gate.
26. Both HVAC JADX runs processed the same 1449-class workload and reported `22` errors. The captured console logs provide only the error count, not per-error identities, so equality of the 22 error sets is not yet proven.
27. CarInfo RU06 -> RU02-labeled artifact:
   - 593 entries are byte-identical;
   - only 5 entries differ;
   - no entries are added or removed;
   - `resources.arsc` is byte-identical;
   - differing entries are `classes.dex`, `AndroidManifest.xml`, and the three v1 signature files.
28. RU06 and RU02-labeled CarInfo `classes.dex` have exactly the same size (`4210872`) but different SHA-256 values.
29. Focused JADX source-tree diff of RU06 vs RU02-labeled CarInfo found exactly one differing generated Java file: `com/desaysv/ivi/vds/carinfo/BuildConfig.java`.
30. The only decompiled Java difference in that file is `VERSION_NAME`:
   - RU06 artifact: `Chery-8155_11_e5fc2d4_2025-09-26_2509261457_R`
   - RU02-labeled artifact: `Chery-8155_11_8f9dc0d_2026-04-10_2604101647_R`
   `BUILD_TYPE`, `DEBUG` and `VERSION_CODE` are unchanged.
31. The collected JADX WARN/ERROR reports for those two CarInfo decompiles contain the same 662 normalized lines as a multiset. The visible hard error is the same AndroidX `DiffUtil.java:107` `RegionMakerVisitor` failure in both.
32. An artifact-identity inconsistency is important: earlier live `dumpsys package com.desaysv.ivi.vds.carinfo` on the current vehicle reported versionName `Chery-8155_11_e5fc2d4_2025-09-26_2509261457_R`, which matches the artifact currently labeled RU06, not the artifact currently labeled `RU02_TEL_2026`. The live APK hash has not yet been compared, so do not assume the RU02-labeled CarInfo artifact is the byte-exact APK currently installed on the canonical car.
33. CarInfo RU05 -> RU06 changes materially in DEX size (`4926660` -> `4210872`), reinforcing the RU05 -> RU06 generation boundary.
34. Quick DEX string scan shows RU05 HVAC and RU05 CarInfo embed/expose many VDBus-extra/carconfig implementation symbols and constants (`VehicleDevice`, `VehicleService`, `EolConfig`, `CarConfigUtil`, `ID_CAR_CONFIG_FRAGRANCE`, multiple `ID_AC_FRAGRANCE*`, Binder interface classes). RU06/RU02-generation artifacts mainly expose client references. This proves a packaging/architecture generation change; it does NOT yet prove Fragrance root cause.
35. Decoded `AndroidManifest.xml` comparison for RU06 vs RU02-labeled HVAC is semantically identical: normalized manifest diff is empty.
36. Decoded `resources.arsc` comparison yields only string/localization deltas: 13 differing common `strings.xml` files and one RU02-only `values-ms-rMY/strings.xml`.
37. No decoded layout, bool, integer, style, id, array, drawable, alias or visibility resource differs between RU06 and RU02-labeled HVAC.
38. The 14 decoded resource deltas are localization maintenance: additions/changes for French, Kazakh, Uzbek, Thai and other locales, Malay locale relocation/retargeting, and small wording fixes in several languages.
39. Fragrance-related resource differences are text-only. The default English `fragrance_too_little_tips` fixes `fragnance` -> `fragrance`; Thai/Malay/French/Kazakh/Uzbek differences are translation/locale changes. Russian Fragrance strings are unchanged.
40. Therefore the observed RU06 -> RU02 HVAC APK delta (DEX + manifest + decoded resources) provides no mechanism that explains Fragrance being hidden on the canonical RU02 bench.

## Important interpretation

The config-side theory "Engineering writes the wrong Fragrance bit" is closed: the observed engineering bit and the current framework mapping match exactly.

The UI `GONE` observation is important, but is NOT by itself accepted as the root cause of the whole RU02 problem, because the older HVAC APK also failed on the current system base.

The RU06 vs RU02-labeled CarInfo decompiled application code is functionally indistinguishable except for build/version metadata. Artifact identity still has to be reconciled against the live canonical car.

The RU06 vs RU02-labeled HVAC application delta is now exhausted at the useful semantic level: the only DEX behavior change is ionizer/config91-specific, the decoded manifest is unchanged, and the decoded resource changes are localization-only. This strongly redirects the investigation away from HVAC APK differences and toward the firmware-specific system/framework/backend path.

## Current open question

Which system/framework/backend condition on the actual live RU02 stack suppresses Fragrance despite config50=1 and despite the HVAC application containing the expected Fragrance predicate/resources?

## Working direction

The canonical vehicle is temporarily unavailable, so live APK hash identity is deferred without blocking offline work.

Immediate read-only offline step: inventory and extract the RU02 system/framework/backend components responsible for vehicle configuration and VDBus (`vdbus_extra.jar`, VehicleDevice, VehicleService and related Desay vehicle services). Map the Fragrance/config50 flow end-to-end from persistent config through EolConfig/CarConfigUtil and VDBus/service events into HVAC. Look specifically for firmware/project/market/telematics/capability/event-update gates that can leave config50 readable as 1 while preventing the Fragrance feature from becoming available.

When the vehicle is available again, pull/hash the live CarInfo/HVAC APKs and reconcile local artifact labels.

Do not install more packages and do not return to unsigned HVAC APK patching until a concrete mechanism is identified.
