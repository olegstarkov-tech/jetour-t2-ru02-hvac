# RU02-HVAC EXPERIMENTS

This file records real runtime/A-B tests on the canonical RU02 bench.

## E1 — Engineering Fragrance toggle

Result: PROVEN.

Observed full vehicle config changes include byte12 transition:

- OFF: `0x85`
- ON: `0xC5`

Delta: `0x40` (bit6).

This matches the framework mapping for config ID 50.

## E2 — Framework mapping from vdbus_extra.jar

Result: PROVEN by decompile.

Relevant facts:

- `ID_CAR_CONFIG_FRAGRANCE = 50`
- `CarConfigUtil.getConfig(50)` delegates to `EolConfig.getJetourEolConfig(50)`
- mapping uses `(mCarConfig1[12] >> 6) & 1`
- config source key: `vehicle.persist.project.ext.configs`

## E3 — Current HVAC startup with Fragrance enabled

Result: PROVEN runtime evidence.

With Engineering Fragrance left ON, current HVAC startup triggered loading of `vehicle.persist.project.ext.configs` containing byte12=`C5`.

Interpretation: current HVAC/system path receives the correct configured Fragrance bit.

## E4 — Install older signed HVAC over current RU02 system

Result: PROVEN negative A/B test.

- Older signed HVAC package installed successfully.
- Package actually executed from `/data/app` rather than the stock `/product/app` copy.
- Fragrance still did not appear.

Interpretation:
- simple HVAC APK replacement is insufficient;
- the problem is not explained solely by the current HVAC APK version.

## E5 — Return to stock HVAC

Result: successful rollback.

After uninstalling the data-app override, package resolution returned to stock `/product/app/SVHvac/SVHvac.apk`.

## E6 — Current UI static inspection

Result: PROVEN static observation.

- current `bottom_layout.xml` sets `fragrance_btn` visibility to `gone`;
- investigated `view/b.java` and `BottomLayoutBindingImpl.executeBindings()` did not reveal a transition of this button to VISIBLE;
- click listener wiring exists.

Interpretation: relevant UI evidence, but not sufficient alone to explain why old HVAC also fails on the same system base.

## Experiment discipline

For future tests record:
- exact firmware/package version;
- command/test performed;
- before/after state;
- rollback;
- raw evidence location where practical;
- classification: PROVEN / NEGATIVE / INCONCLUSIVE.
