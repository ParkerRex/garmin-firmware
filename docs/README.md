# Garmin Express (macOS) — Binary Analysis Notes

This document is intended to be a durable, excruciatingly detailed map of what we can infer about Garmin Express from static analysis of the macOS app bundle and related artifacts. It is written to help future engineers navigate functionality even without source.

Scope:
- The analysis is static (Obj‑C runtime dump, symbols, strings, load commands).
- No dynamic tracing or runtime instrumentation was performed.
- We did not extract firmware source — only device metadata and Connect IQ config files from a connected Forerunner 735XT.

Where the raw artifacts live:
- Full dumps: `artifacts/` (symbols, strings, Obj‑C runtime, device files)
- Device capture: `artifacts/GarminDevice.xml`, `artifacts/OUT.BIN`, `artifacts/EPO.BIN`

---

## 1) High‑level architecture (what the app is doing)

### 1.1 App bundle structure

From `/Applications/Garmin Express.app/Contents`:
- `MacOS/Garmin Express` — main app binary (x86_64 Mach‑O)
- `Resources/*.nib` — AppKit UI, heavy use of nib-based view controllers
- `Frameworks/` — embedded/internal frameworks:
  - `ApplicationInsightsOSX.framework`
  - `ConnectIQSerialization.framework`
  - `Telemetry.framework`
  - `TrueTime.framework`
  - `OAuthConsumer.framework`
  - `Promises.framework`, `FBLPromises.framework`
- `Library/LoginItems/Garmin Express Service.app` — helper/background service

There is **no `PlugIns/` directory** in the app bundle (no obvious plugin system on macOS).

### 1.2 Core responsibilities inferred

From class names, selectors, and resource nibs we can confidently separate the app into these functional areas:

1) Device onboarding and management
- Add device wizard, choose device model, registration
- Device hub views, device settings, device actions

2) Updates and content delivery
- Device firmware updates
- App updates (Garmin Express self-update)
- Map updates, chart updates, universal map handling
- Connect IQ app updates

3) Connect IQ management
- Install/remove IQ apps, auto-update settings
- Serialization and config management for IQ apps

4) Wi‑Fi provisioning and management
- Wi‑Fi setup windows and utilities
- Write/read Wi‑Fi settings to device

5) Music management
- Music storage view, playlist management
- iTunesLibrary usage (see linked framework)

6) Marine workflows
- Marine device management, charts, downloads

7) UI flows and generic UI infrastructure
- Many AppKit view controllers and window controllers
- Reusable “pod” UI units (e.g., ActionPod, QueuePod, PrivacyPod, etc.)

8) Telemetry and analytics
- Telemetry framework
- ApplicationInsights framework

### 1.3 What the helper service likely does

The LoginItem app (`Garmin Express Service.app`) contains a full set of classes, selectors, and strings of its own. It likely:
- Runs in background for device detection, sync, or update prep
- Handles USB connection polling or data transfer
- Interfaces with device mount events and Disk Arbitration

See: `artifacts/garmin-express-service.objc-runtime.txt`, `artifacts/garmin-express-service.strings.txt`.

---

## 2) Feature‑by‑feature detail (what’s in the binary)

This section maps feature areas to class names inferred from Obj‑C runtime and selector lists. Use these as entry points when reconstructing the app.

### 2.1 Device onboarding & management

Examples from `artifacts/garmin-express.objc-classes.txt`:
- `AddDeviceHelpInstructionsViewController`
- `AddDeviceHelpItem`, `AddDeviceHelpItemFactory`
- `AddDeviceHelpViewController`
- `AddDeviceProfileViewController`
- `AddDeviceRegistrationViewController`
- `AddDeviceSettingsViewController`
- `AddDeviceWindowController`
- `AddDeviceWizardViewController`
- `ChooseDeviceViewController`, `ChooseDeviceModel`
- `ConnectDeviceWindowController`
- `DeviceHubView`, `DeviceHubViewController`
- `ManageDeviceViewController`
- `DeleteDevicePodViewController`

Relevant nibs (all in `Resources/`):
- `AddDevice*.nib`
- `ChooseDevice*.nib`
- `ConnectDeviceWindowController.nib`
- `DeviceHubViewController.nib`

Practical note: nib/controller names are closely aligned, so look for matching names in the nib list for UI entry points.

### 2.2 Device firmware updates & software updates

Classes from `artifacts/garmin-express.classes.Update.txt`:
- `DeviceSoftwareUpdate*` (installation, release notes, response)
- `FirmwareUpdate`, `FirmwareUpdateHelper`
- `AppUpdateKey`, `AppUpdateProgress` (Garmin Express self update)
- `GarageUpdateKey` (likely device inventory updates)

Map updates and content updates are also present (see 2.3).

### 2.3 Maps, charts, and navigation content

Classes from `artifacts/garmin-express.classes.Map.txt`:
- `ManageMapsViewController`
- `MapContent`, `MapContentEx`
- `MapOptionsViewController`
- `MapUpdateViewController`
- `GetMapUpdatesDelegate`, `GetMapDownloadDetailsDelegate`
- `GeminiMapUpdate`, `GeminiMapInstallRequest` (Gemini appears to be an internal update pipeline)
- `UniversalMap*` classes (universal map options & progress)

Marine chart updates appear separately in marine classes.

### 2.4 Connect IQ app management

Classes from `artifacts/garmin-express.classes.IQ.txt` and Connect classes:
- `IQApp`, `IQAppDetails`, `IQSSerialization`
- `GetIQAppUpdateDelegate`
- `GetAutoUpdateConnectIQAppsDelegate`
- `InstallAppsToConnectIQDelegate`
- `RemoveConnectIQAppsDelegate`
- `SetAutoUpdateConnectIQAppsDelegate`

Expect flows for listing apps, applying updates, and toggling auto‑update settings.

### 2.5 Wi‑Fi

Classes from `artifacts/garmin-express.classes.WiFi.txt`:
- `WifiSetupWindowController`
- `WifiViewController`
- `WifiUtilities`
- `WriteWifiSettingsRequest`
- `ReadWifiSettingsResponse`
- `SetDeviceWifiStatusDelegate`

### 2.6 Music management

Classes from `artifacts/garmin-express.classes.Music.txt`:
- `MusicManagementViewController`
- `MusicManagementHelpWindowController`
- `MusicAppsViewController`
- `DeviceMusicStorageViewController`

The binary links against `iTunesLibrary.framework` (see `artifacts/garmin-express.otool-L.txt`).

### 2.7 Marine

Classes from `artifacts/garmin-express.classes.Marine.txt`:
- `ManageMarineViewController`
- `MarineDevice*` classes
- `MarineDownloadViewController`
- `MarineQueuePodViewController`
- `DDMarine*` (marine views/providers)

### 2.8 Privacy, consent, and account

Relevant classes:
- `PrivacyViewController`, `PrivacyPodViewController`
- `ConsentModalWindowController`, `MultipleConsentWindowController`
- `RevokeConsent*` controllers

Account/SSO:
- `ConnectSsoWindowController` (Connect account workflows)
- `ConnectTicketExchanger`, `ConnectTokenToITTokenDelegate` (token exchange flow)

### 2.9 UI infrastructure and “Pod” system

Many nibs and controllers use a pod metaphor (action pods, queue pods, etc.). Examples:
- `ActionPod*ViewController`
- `QueuePodViewController`, `OptionalUpdatesPodViewController`
- `PrivacyPodViewController`, `StoragePodViewController`, `WiFiPodViewController`

This suggests a reusable UI system used across features.

---

## 3) Deep dive — how to follow flows in the artifacts

This section explains how to interpret the dumps.

### 3.1 Obj‑C runtime dump

Files:
- `artifacts/garmin-express.objc-runtime.txt`
- `artifacts/garmin-express-service.objc-runtime.txt`

These include:
- class lists
- method/selector names
- Obj‑C metadata for each class

Useful extraction files:
- `artifacts/garmin-express.objc-classes.txt`
- `artifacts/garmin-express.objc-selectors.txt`

Search strategy:
1) Start with a known UI nib or class (e.g., `ManageMapsViewController`).
2) Grep the runtime dump for that class to see its method list.
3) Use selector names to find related delegate protocols and data fetch flows.

### 3.2 Strings dump

Files:
- `artifacts/garmin-express.strings.txt`
- `artifacts/garmin-express-service.strings.txt`

Use to:
- find error messages and config keys
- find file paths and update locations
- discover hidden preferences or feature flags

### 3.3 Symbol dumps

Files:
- `artifacts/garmin-express.nm-m.txt` (mangled + demangled symbols)
- `artifacts/garmin-express.nm-gU.txt` (global symbols only)

Use to:
- identify C/C++ functions, Swift symbols, and bridged code
- map functional clusters (e.g., serialization, update, device I/O)

### 3.4 Linked frameworks

File: `artifacts/garmin-express.otool-L.txt`

Key takeaways:
- `DiskArbitration`, `IOKit` — likely device mount detection and USB device handling
- `ServiceManagement` — login item management
- `WebKit` — embedded web content in UI
- `iTunesLibrary` — music sync integration

---

## 4) Device‑side artifacts (Forerunner 735XT)

Captured from `/Volumes/GARMIN` and stored in `artifacts/`.

### 4.1 GarminDevice.xml

Location: `artifacts/GarminDevice.xml`

Key info:
- Model: Forerunner 735XT
- PartNumber: 006‑B2158‑00
- SoftwareVersion: 9.80
- Update file expectations:
  - Main firmware: `GARMIN/GUPDATE.GCD` (not present on device storage at capture)
  - RemoteSW updates: `GARMIN/REMOTESW/GUPDATE.GCD`, `GUP2511.GCD`, `GUP2423.GCD`, `GUP2510.GCD`
  - Text packs: `GARMIN/TEXT/*.ln2`

### 4.2 OUT.BIN (Connect IQ configuration/state)

Location: `artifacts/OUT.BIN`

Observations:
- Small binary (661 bytes)
- Contains strings that match widget/data‑field names:
  - HRV Stress Test, Heart Rate, My Day, Last Run/Ride/Swim, Steps, Intensity Minutes, Calories
  - VIRB, Calendar, Weather, Controls, Notifications
  - Strava Suffer Score

This aligns with the IQAppsConfiguration data type in GarminDevice.xml.

### 4.3 EPO.BIN (ephemeris)

Location: `artifacts/EPO.BIN`

Observations:
- 63 KB binary blob, no readable strings
- Likely ephemeris payload for faster satellite acquisition

---

## 5) What we still *don’t* have (important gaps)

- Firmware source code is not retrievable via update files; firmware updates are compiled/signed binaries.
- No `GUPDATE.GCD` firmware image present on the device at capture time.
- No dynamic tracing logs or runtime introspection from the app.

---

## 6) Suggested next steps

1) Capture firmware update binaries
- Trigger Garmin Express to download firmware for the 735XT.
- Copy `GUPDATE.GCD` from device or local update cache.
- Add to `artifacts/` with a corresponding analysis file.

2) Map UI to controllers
- Cross-reference `Resources/*.nib` names with classes to map UI flows.
- Build a “nib → controller → feature area” table.

3) Add a “feature map” appendix
- For each feature, list main controllers, delegate classes, and relevant selectors.

---

## 7) Index of key artifacts

- `artifacts/garmin-express.objc-classes.txt`
- `artifacts/garmin-express.objc-selectors.txt`
- `artifacts/garmin-express.objc-runtime.txt`
- `artifacts/garmin-express.strings.txt`
- `artifacts/garmin-express.nm-m.txt`
- `artifacts/garmin-express.otool-L.txt`
- `artifacts/garmin-express-service.objc-runtime.txt`
- `artifacts/garmin-express-service.strings.txt`
- `artifacts/GarminDevice.xml`
- `artifacts/OUT.BIN`
- `artifacts/EPO.BIN`

