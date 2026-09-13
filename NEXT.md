# RU02-HVAC NEXT

## Current objective

Live-validate the proven RU05 same-process configuration-change rebuild as a no-patch Fragrance visibility refresh after config50 has loaded.

## Current state

- Current RU02-generation T1J UI has no proven Fragrance visibility activation path; static HVAC-targeting RRO is absent.
- RU05 T1J has a valid visibility path: `OfflineConfigManager.e()` -> `getConfig(50)==1` -> `VISIBLE(0)` / `INVISIBLE(4)`.
- RU05 maps config50 exactly as `(mCarConfig1[12] >> 6) & 1` and loads the project ext-config through its embedded VDBus stack.
- Late config update does not automatically refresh the existing BottomLayout binding.
- RU05 `view/b.t1()` rebuilds the main/bottom DataBinding tree in the same process.
- Exact configuration-change path is now proven:
  `onConfigurationChanged()` -> private `b()` -> if shown `d1(false)` -> handler message -> `I0(context)` -> `L0()` -> `t1()` -> fresh BottomLayout binding.
- `onConfigurationChanged()` only enters this rebuild when language changes or `(uiMode & 0x30)` changes.
- Because the process remains alive, already-loaded static `EolConfig.mCarConfig1` can survive into the fresh binding evaluation.

## Next step

Vehicle-return live test, one safe step at a time.

First read-only step before any change:

1. install/start exact signed RU05 HVAC as previously validated;
2. verify from runtime evidence that RU05 has loaded Fragrance config50 as `1`;
3. capture current Android night/uiMode state with ADB and keep that value for rollback.

Only after the original state is recorded, toggle night mode to the opposite value once, observe the app logs/UI, and immediately restore the original value.

Expected proof chain in logs:

- `onConfigurationChanged currentNightMode...`
- `onConfigurationChanged destoryAndReshow isHvacShow=...`
- `destroyHvac isNeedRemoveRoot=false`
- `HvacContentView init`
- `isFragranceExist = true`

Success criterion: Fragrance entry becomes visible after the fresh BottomLayout binding is created.

If the rebuild occurs and `isFragranceExist=true` but the button still does not appear, reopen only the RU05 generated binding/runtime UI state. If config50 is not `1` at rebuild time, investigate runtime class loading / RU05 config stack instead.

Do not install additional RU05 components, patch Desay APKs, or blind-write VDBus/properties.
