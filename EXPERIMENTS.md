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

## E7 — Signed RU05 HVAC: hot update vs full HU reboot (2026-11-12)

Result: **PROVEN positive Fragrance UI after full HU reboot**, negative after hot-update/day-night change.

- Both runs: official T1J/RU 00.00.02, exact signed RU05 HVAC SHA256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`.
- **Hot-update PID13304, 16:20:53:** UI lacked Fragrance. Runtime logs showed `CarInfoProxy` and vehicle binding, but **no** HVAC process-local `VDB-EXTRA-CarConfigUtil.init`, `[onVDConnected]`, or own event918905 subscription. VehicleDevice's early get(e0006) read `vehicle.persist.project.pn`. Nonfatal Resources$NotFoundException 0x7f0f00a0 at 16:20:53.317. Genuine Android night-mode rebuild at 16:26 also did not expose button.
- **Full HU reboot PID7601, 17:05:35:** user directly observes Fragrance button VISIBLE. HVAC's own `CarConfigUtil.init` at .048, `onVDConnected` at .113, event918905 subscribe at .114 and backend subscription by PID7601 at .118, ext.configs C5 returned by VehicleDevice at .126. CarInfoProxy.init at .152. No resource exception in successful captured boot.
- Original RU02 HVAC remains preinstalled under `/product/app/SVHvac`; active code after reboot was RU05 `/data/app/.../base.apk` with matching SHA. No other device changes are part of this successful observation.
- **Inference/OPEN:** successful boot executes a full carconfig initialization path absent from the failed run's log; direct runtime verification of in-process static config and causal reason for hot-install discrepancy still open. Hardware Fragrance operation is not established by a visible menu.

Artifacts: user-uploaded `RU05_FRAGRANCE_CONFIG_LOG.txt` (failed) and `RU05_FRAGRANCE_SUCCESS_BOOT.txt` (success), checked in this investigation. Successful-state rollback remains the stock RU02 system APK, but **do not perform rollback while testing/using the working RU05 UI**.

## E8 — Fragrance cartridge enumeration and controls (2026-11-12)

Result: **PROVEN positive user-visible UI/control operation** on signed RU05 HVAC running on dealer RU02/T1J after HU reboot.

- User's direct live observation: "Картриджи видны, все управляется".
- The Fragrance interface not only appears but shows cartridges and permits normal control interactions.
- Physical scent emission, backend action confirmation, and persistence across future reboots are not independently tested/reported.
- No intervention or extra write was performed to achieve this observation beyond the previously recorded full HU reboot; preserve state.

## E9 — End-to-end fragrance emission verified (2026-11-13)

Result: **PROVEN user-observed positive physical operation**.

- Tested on the canonical official T1J dealer RU02/00.00.02 with exact signed RU05 HVAC SHA256 `f7dd31844be3fab910d191cca8545391d5cb213c416950707c3d75f184e13522`, active after a complete HU reboot.
- Previous operator observation: cartridge information displayed, controls functioning.
- New direct operator confirmation: "пахнет" — fragrance scent is physically noticeable during use.
- Distinguish confirmed physical/user-level result from unmeasured electrical telemetry, cartridge-level sensor protocol or consistency after future cold boots; those were not instrumented.
- Preserve stable working installation; no diagnostic action taken to obtain this confirmation.

## Experiment discipline

For future tests record:
- exact firmware/package version;
- command/test performed;
- before/after state;
- rollback;
- raw evidence location where practical;
- classification: PROVEN / NEGATIVE / INCONCLUSIVE.
