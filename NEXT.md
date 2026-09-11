# RU02-HVAC NEXT

## Current objective

Identify the system/backend condition on dealer RU firmware 00.00.02 that blocks OEM Fragrance availability despite config50=1 being correctly present.

## Next step

Do not return to APK patching.

First collect read-only runtime evidence from the active stock HVAC while Fragrance is ON:

1. confirm current stock package/version;
2. capture focused logcat around HVAC start and VehicleDevice/VehicleService/CarConfigUtil/EolConfig activity;
3. look specifically for project-config update/load events and any Fragrance-related runtime branch or suppression condition.

Only after that decide whether the next comparison target should be VehicleDevice/VehicleService framework components or another known-good firmware base.

## Guardrails

- Canonical bench: T1J RU dealer 00.00.02 with telematics.
- No blind VDBus/property writes.
- No unsigned system APK patch path.
- Do not import D08/00.00.08 conclusions as facts.
- Do not reopen DISPROVEN hypotheses without new evidence.
