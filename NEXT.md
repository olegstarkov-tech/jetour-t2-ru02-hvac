# RU02-HVAC NEXT

## Current objective

Discriminate whether the previously observed RU05 day/night/auto no-effect result actually exercised the proven Android configuration-change rebuild after config50 was already loaded.

## Current state

- Current RU02-generation T1J UI has no proven Fragrance visibility activation path; static HVAC-targeting RRO is absent.
- RU05 T1J has a valid visibility path: `OfflineConfigManager.e()` -> `getConfig(50)==1` -> `VISIBLE(0)` / `INVISIBLE(4)`.
- RU05 maps config50 exactly as `(mCarConfig1[12] >> 6) & 1` and loads the project ext-config through its embedded VDBus stack.
- Late config update does not automatically refresh the existing BottomLayout binding.
- RU05 `view/b.t1()` rebuilds the main/bottom DataBinding tree in the same process.
- Exact configuration-change path is proven:
  `onConfigurationChanged()` -> private `b()` -> if shown `d1(false)` -> handler message -> `I0(context)` -> `L0()` -> `t1()` -> fresh BottomLayout binding.
- `onConfigurationChanged()` enters this rebuild when language changes or `(uiMode & 0x30)` changes.
- Historical direct live evidence: while signed RU05 HVAC was installed on the running canonical vehicle, the user manually switched HU quick-shade day/night/auto modes and Fragrance did not appear.
- That historical test did not capture whether the shade switch actually changed Android `uiMode & 0x30`, triggered the exact rebuild, preserved the same PID, or occurred after RU05 `getConfig(50)` became `1`.

## Next step

When the vehicle returns, do not blindly repeat day/night switching.

First capture the runtime preconditions read-only:

1. exact signed RU05 HVAC running;
2. PID of `com.desaysv.svhvac`;
3. focused logcat proving RU05 `isFragranceExist` / config50 state;
4. current Android night/uiMode state;
5. focused logcat filters for `onConfigurationChanged`, `destoryAndReshow`, `destroyHvac`, `HvacContentView init`, and `isFragranceExist`.

Only after that evidence is live, toggle Android night mode once through ADB and observe:

- whether PID remains unchanged;
- whether the exact configuration rebuild chain runs;
- whether fresh binding logs `isFragranceExist=true`;
- whether the Fragrance entry becomes visible.

Interpretation:

- rebuild executes + same PID + `isFragranceExist=true` + button appears -> no-patch RU05 workaround proven;
- rebuild executes + same PID + `isFragranceExist=true` + button absent -> stale-binding workaround disproven; reopen only RU05 runtime binding/UI path;
- rebuild executes but `isFragranceExist=false` -> investigate RU05 runtime config/class-loading path;
- shade control does not produce the same Android configuration-change logs -> prior manual day/night/auto observation does not test this mechanism.

Do not install additional RU05 components, patch Desay APKs, or blind-write VDBus/properties.
