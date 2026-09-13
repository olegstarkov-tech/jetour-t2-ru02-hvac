# RU02-HVAC NEXT

## Current objective

Determine whether RU05's stock configuration-change handling can be used as a no-patch, same-process Fragrance visibility refresh after config50 has loaded.

## Current state

- Current RU02-generation T1J UI still has no Fragrance visibility activation path; static HVAC-targeting RRO is absent.
- RU05 T1J has a valid visibility path: `OfflineConfigManager.e()` -> `getConfig(50)==1` -> `VISIBLE(0)` / `INVISIBLE(4)`.
- RU05 maps config50 exactly as `(mCarConfig1[12] >> 6) & 1` and loads the same project ext-config through its embedded VDBus stack.
- Async config load/update does not automatically rebind existing BottomLayout.
- RU05 `view/b.t1()` is now proven to remove the existing main HVAC child, inflate a new `HvacMainViewBinding`, capture its new `bottomLayout`, and add the new root back while staying in the same process.
- `view/b.I0(context)` calls `L0()`, and the traced initialization path reaches `t1()`.
- Exported `HvacService` has stock `type` commands (`OPEN_PANEL`, `CLOSE_PANEL`, `CONTROL_PANEL`, `SSS`, `HHH`, VR open/close fragment), but none is yet proven to call `t1()` directly.
- `HvacApplication` contains a private method with log string `onConfigurationChanged destoryAndReshow isHvacShow=`; the configuration-change path calls this method, making it the strongest stock rebuild candidate.
- The current lifecycle report omitted the middle of that private method, so the exact destroy/re-show sequence is still unknown.

## Next step

From the already decoded RU05 APK, extract only:

1. full `HvacApplication.smali` private method `b()V`;
2. full configuration-change callback that invokes `b()`;
3. full `HvacApplication$a.smali` Handler class, because the application constructor creates this handler and the rebuild method may defer work through it.

Do not run more broad grep/audit scripts.

If those exact bodies prove a same-process destroy + `I0/L0/t1` reinitialization, prepare a reversible live test for vehicle return:

- install/start signed RU05 HVAC;
- verify/wait for RU05 config load (`mCarConfig1` / config50=1) from logs;
- capture the current Android configuration value to be changed;
- trigger one benign configuration change through ADB;
- verify the stock `destoryAndReshow` path and Fragrance visibility;
- restore the original configuration value.

Do not yet choose or execute the ADB configuration command until the exact application code shows which configuration fields it reacts to and how the rebuild works.

Do not install additional RU05 components, patch Desay APKs, or blind-write VDBus/properties.
