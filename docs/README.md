# Notes on Garmin Express Binary Analysis

This directory contains summarized notes about what we learned by inspecting the Garmin Express macOS app bundle and related binaries.

Key findings so far:
- The app is a compiled macOS bundle with AppKit UI and extensive nib resources.
- Obj‑C runtime inspection shows feature areas for device onboarding, maps, updates, Connect IQ, Wi‑Fi, music, and marine tooling.
- No plugin/extension folder was present in the bundle (no `PlugIns` directory).
- The bundle embeds internal frameworks: ApplicationInsightsOSX, ConnectIQSerialization, Telemetry, TrueTime, OAuthConsumer, Promises/FBLPromises.

Artifacts (full dumps, symbols, strings, and device files) live in `artifacts/`.
