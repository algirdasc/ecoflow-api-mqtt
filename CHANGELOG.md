# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.10.17-alpha.1] - 2026-06-01

### Experimental

- **Rebased Stream Microinverter support onto the current release line
  (issue #53)** - adds the dedicated `Stream Microinverter` device type on top
  of `v1.10.15`, so testers no longer need to roll back to the older
  `v1.10.8-alpha.1` build and lose the Base Load Power fixes.
- Adds the initial telemetry mapping for PV string power (`powGetPv`,
  `powGetPv2`, `powGetPv3`, `powGetPv4`), grid power/voltage/frequency/status,
  feed-in power limit fields, inverter temperature, and Wi-Fi RSSI.
- Registers empty control maps for switch/number/select/button platforms so the
  device type does not fall back to unrelated Delta or PowerStream controls.

### Notes

- This is still a validation alpha. The field names came from issue #53
  reports and still need confirmation from a real `/quota/all` or DEBUG/MQTT
  payload on the latest line.

---

## [1.10.15] - 2026-05-30

### 🐛 Bug Fixes

- **Fixed Base Load Power showing "unknown" in the entity pop-out / dashboard
  (issue #49)** — `EcoFlowStreamNumber.native_value` tried to `float()` the
  `dayResidentLoadList` schedule dict, which raised and left the state as
  `unknown` (so the slider only worked from the device page, not the more-info
  dialog or a dashboard card). The read path now decodes the configured power
  via `_extract_resident_load_power`, the same way the write path and
  `EcoFlowNumber` already did. This also fixes the related "slider value doesn't
  match the EcoFlow app" symptom.

---

## [1.10.14] - 2026-05-30

### 🎉 New Features

- **Per-AC-plug live consumption for Stream Ultra X / AC Pro (issue #52)** —
  Added two power sensors exposing the live consumption of each AC output
  (Schuko) port, matching the values shown in the official EcoFlow app:
  - **AC Plug 1 Power** — `powGetSchuko1`
  - **AC Plug 2 Power** — `powGetSchuko2`

  Both are reported in plain watts (no scaling), enabling automations such as
  detecting when an appliance on a specific plug has finished running. Confirmed
  working by the reporter on both Stream Ultra and Stream AC Pro (graduated from
  the v1.10.10 pre-release).

---

## [1.10.12] - 2026-05-29

### 🔧 Consolidation

- Consolidated `main` onto a single release line. This build brings together
  the previously branch-only fixes: Stream AC relay command values (#50),
  Delta Pro telemetry scaling (#54), Stream Base Load Power control (#49), and
  the Delta Pro Ultra system mode fix (#51, detailed under 1.10.7 below).

---

## [1.10.11] - 2026-05-29

### 🎉 New Features

- **Base Load Power control for Stream Ultra X / AC Pro (issue #49)** —
  Added a **Base Load Power** number entity (0–800 W) that sets the feed-in /
  base-load setpoint via the device's `dayResidentLoadList` schedule
  (`cfgDayResidentLoadList`), replacing the need to manage it through the
  EcoFlow app. Confirmed working by community testers (graduated from the
  `v1.10.6-alpha.x` builds).
  - Reads the configured setpoint from `dayResidentLoadList.load[].loadPower`
    (`powGetSysLoad` is live output, not the setpoint).
  - Writing preserves the existing schedule windows and only updates their
    `loadPower` values.
  - **Disabled by default** — it writes to the device and only applies when a
    base-load schedule exists, so enable it explicitly under the device's
    entity list.

  > ℹ️ **Multi-period limitation:** a single Home Assistant number cannot
  > represent a multi-period schedule, so changing it applies the same value to
  > **all** configured periods. To use different values per time window, edit the
  > schedule in the EcoFlow app.

### 🔧 Technical

- Added DEBUG-level "changed fields" MQTT logging in the hybrid coordinator to
  make future field reverse-engineering easier.

This release also includes the Delta Pro telemetry scaling fix from v1.10.9
(issue #54).

---

## [1.10.9] - 2026-05-29

### 🐛 Bug Fixes

- **Fixed unscaled milli-unit Delta Pro telemetry (issue #54)** —
  Several original Delta Pro sensors exposed the raw fixed-point values from
  the EcoFlow API instead of scaling them into standard units, producing
  impossible readings and corrupting Recorder/Energy Dashboard statistics:
  - `bmsMaster.vol` (Battery Voltage) and `bmsMaster.amp` (Battery Current)
    are reported in milli-units (factor `0.001`) and are now divided by 1000
    (e.g. `49305` → `49.305 V`, `-3443` → `-3.443 A`).
  - The MPPT power family — `mppt.inWatts` (Solar Input Power),
    `mppt.outWatts`, `mppt.dcdc12vWatts`, `mppt.carOutWatts` — is reported in
    deciwatts (factor `0.1`) and is now divided by 10.

  `inv.*` / `pd.*` watt fields are plain watts and were left unchanged.
  Field factors were cross-verified against the EcoFlow Developer API and the
  foxthefox/ioBroker and tolwi/hassio-ecoflow-cloud reference implementations.

  > ⚠️ Existing long-term statistics for these sensors recorded before this
  > release contain a 1000× (voltage/current) or 10× (solar power)
  > discontinuity and may need a manual statistics purge.

---

## [1.10.7] - 2026-05-19

### Bug Fixes

- **Fixed Delta Pro Ultra System Mode enum state after toggling AC Output
  (issue #51)** - `sysWordMode` values now map numeric API states
  (`0` / `1` / `2` / `3`) to valid Home Assistant enum options instead of
  exposing raw `0`, which Home Assistant rejected as an invalid enum state.
- **Improved enum value mapping for numeric strings** - sensor `value_map`
  definitions now also handle API values such as `"0"` in addition to numeric
  `0`.

---

## [1.10.6] - 2026-05-14

### 🐛 Bug Fixes

- **Fixed Stream AC1/AC2 switch command values (PR #50)** —
  Stream BKW AC relay commands now send boolean `true` / `false` values for
  `cfgRelay2Onoff` and `cfgRelay3Onoff` instead of integer `1` / `0`,
  matching the EcoFlow Developer API and avoiding validation error 8524.

---

## [1.10.5] - 2026-04-20

### 🐛 Bug Fixes

- **Fixed AC2 switch validation error on Stream Ultra X (follow-up to issue #45)** —
  EcoFlow's BKW docs specify that in multi-device BKW systems AC1 and AC2
  relays can live on different physical devices, so sending
  `cfgRelay{2,3}Onoff` to a device that does not own the corresponding relay
  is rejected by the REST API with validation error 8524. The integration
  now skips creating a Stream Ultra X switch entity if the device's
  `/quota/all` response does not include the corresponding `relay2Onoff` /
  `relay3Onoff` / `feedGridMode` key, so no-op switches never reach the UI.
  A full multi-device BKW topology (main SN + relay routing) will follow as
  a separate enhancement.

---

## [1.10.4] - 2026-04-20

### 🐛 Bug Fixes

- **Fixed Stream Ultra X "Unknown" entities after EcoFlow maintenance (issue #45)** —
  MQTT credentials are now automatically re-fetched from the EcoFlow REST API
  when the broker rejects the stored credentials (CONNACK rc=4 / rc=5), with a
  5-minute cooldown to avoid hammering the API. Previously the integration
  stayed stuck on stale credentials after EcoFlow rotated them during their
  maintenance window.
- **Fixed AC1 Switch silent failure on Stream Ultra X** — switch commands with
  `needAck` now wait up to 5s for a matching `set_reply` MQTT message. If no
  ACK arrives (broker silently dropped the publish), the hybrid coordinator
  falls back to the REST API instead of reporting success while the device
  ignored the command.
- **Faster broker-stall recovery for Stream series** — MQTT silence threshold
  lowered from 180s to 90s specifically for Stream Ultra X, matching the
  broker's observed stall behaviour during EcoFlow maintenance windows.

---

## [1.8.4] - 2026-02-08

### 🐛 Bug Fixes

- **Reverted switch value type** - EcoFlow API accepts boolean `true`/`false` values (not integers) for switch commands
- **Removed unsupported controls** - Removed AC HV Always On, AC LV Always On switches and AC Always On Min SOC number as they have no documented SET commands in EcoFlow API (read-only fields)

---

## [1.8.3] - 2026-02-08

### 🐛 Bug Fixes

- **Fixed Delta Pro 3 switch commands** - (Reverted in 1.8.4)

---

## [1.8.2] - 2026-02-08

### 🐛 Bug Fixes

- **Fixed asyncio.CancelledError handling** - Properly handle task cancellation during coordinator shutdown to prevent initialization errors
- **Fixed Delta Pro 3 temperature sensors** - Added 17 temperature sensors with correct API keys (without `bms` prefix) to match actual API response

### 🔧 Technical

- CancelledError now properly re-raised to allow graceful shutdown (Python 3.8+ compatibility)
- Temperature sensor keys: `maxCellTemp`, `minCellTemp`, `maxMosTemp`, `minMosTemp`, `maxEnvTemp`, `minEnvTemp`, `maxCurSensorTemp`, `minCurSensorTemp`, `temp`, `invNtcTemp2`, `invNtcTemp3`, `adsNtcTemp`, `llcNtcTemp`, `tempPvH`, `tempPvL`, `tempPcsAc`, `tempPcsDc`

---

## [1.8.1] - 2026-02-08

### 🎉 New Features

#### Stream Ultra X Support (Issue #16)

Full support for EcoFlow Stream Ultra X (BK61 series) home battery/balcony power system:

**Sensors:**
- Battery Level (`cmsBattSoc`)
- Solar Input Power (`powGetPvSum`)
- System Load Power (`powGetSysLoad`)
- Grid Power (`powGetSysGrid`)
- Grid Connection Power (`gridConnectionPower`) - positive = consuming, negative = feed-in
- Battery Power (`powGetBpCms`) - positive = charging, negative = discharging
- Backup Reserve Level, Max/Min Charge Levels

**Switches:**
- AC1 Output (`relay2Onoff`)
- AC2 Output (`relay3Onoff`)
- Feed-in Control (`feedGridMode`)

**Numbers:**
- Backup Reserve Level (3-95%)
- Max Charge Level (50-100%)
- Min Discharge Level (0-30%)

**Select:**
- Operating Mode (Self-Powered / AI Mode)

**Binary Sensors:**
- Battery Charging/Discharging
- Solar Generating
- Grid Feed-in/Consuming
- AC1/AC2 Status

**Supported Stream Models:**
- STREAM Ultra X (BK61)
- STREAM Ultra, STREAM Pro, STREAM AC Pro
- STREAM Max, STREAM AC
- STREAM Ultra (US)

#### Delta Pro 3 Enhancements

**New Controls:**
- AC HV Always On switch (`cfgAcHvAlwaysOn`)
- AC LV Always On switch (`cfgAcLvAlwaysOn`)
- AC Always On Min SOC number (`cfgAcAlwaysOnMinSoc`)
- Power Off button (`cfgPowerOff`)

**New Temperature Sensors (17 total):**
- BMS detailed temps: `maxCellTemp`, `minCellTemp`, `maxMosTemp`, `minMosTemp`
- Environment temps: `maxEnvTemp`, `minEnvTemp`
- Current sensor temps: `maxCurSensorTemp`, `minCurSensorTemp`
- BMS Temperature: `temp`
- Inverter temps: `invNtcTemp2`, `invNtcTemp3`
- Component temps: `adsNtcTemp`, `llcNtcTemp`
- Solar temps: `tempPvH`, `tempPvL`
- PCS temps: `tempPcsAc`, `tempPcsDc`

### 🔧 Technical Improvements

- New `EcoFlowStreamSwitch`, `EcoFlowStreamNumber`, `EcoFlowStreamSelect` classes
- Stream API format with `cmdId`, `cmdFunc`, `dirDest`, `dirSrc`, `dest` parameters
- Proper handling of nested params for operating mode
- New `Platform.BUTTON` support with `EcoFlowButton` class

## [1.7.0] - 2026-02-08

### 🎉 New Features

#### Extended Delta 2 Sensors with Value Mapping

Added ~80 additional sensors for comprehensive Delta 2 monitoring:

**ENUM Value Mapping:**
Sensors now display human-readable text instead of raw numbers:
- `pd.ext4p8Port`: "none" / "extra_battery" / "smart_generator"
- `pd.chgDsgState`: "idle" / "discharging" / "charging"
- `inv.fanState`: "off" / "low" / "medium" / "high"
- `mppt.chgType`: "null" / "adapter" / "solar" / "car" / "generator"
- `mppt.chgState`: "off" / "charging" / "full" / "paused"
- And 30+ more ENUM sensors

**Extra Battery Support:**
- Extra Battery 1 & 2 connection status via `bms_kitInfo.watts`
- Battery level (SOC) for each connected extra battery
- Power output for each extra battery

**New Sensor Categories:**
- **BMS** - cycles, SOH, temperatures, voltages, capacities
- **EMS** - charge/discharge state, fan level, parallel voltages
- **PD** - USB/Type-C power, car power, system settings
- **INV** - AC frequencies, voltages, temperatures
- **MPPT** - solar charging, DC output, standby settings

### 🔧 Technical Improvements

- Generic `value_map` handling for ENUM device class sensors
- Array parsing for `bms_kitInfo.watts` with `kit_index` and `kit_field` support
- Proper handling of unavailable extra batteries (returns None)

## [1.6.0] - 2026-01-25

### 🎉 New Features

#### Delta 2 Device Support (Issue #12)

Full support for EcoFlow Delta 2 portable power station using the unique `moduleType`/`operateType` API format.

**Sensors (40+):**
- 🔋 **Battery** - SOC, SOC (precise), voltage, current, temperature, cycles, design capacity, full capacity, remaining capacity
- ⚡ **Power** - Total input/output, AC input/output, solar input, USB-A/USB-C outputs, car output
- 🌡️ **Temperature** - Battery temp, inverter temp, MPPT temp
- ⚙️ **Settings** - Max charge SOC, min discharge SOC, charging power, standby times, LCD settings

**Switches:**
- AC Output
- X-Boost
- DC/USB Output
- Car Charger
- Beeper (Silent Mode)

**Number Controls:**
- Max Charge Level (50-100%)
- Min Discharge Level (0-30%)
- AC Charging Power (100-1200W)
- DC Charging Current (4000-10000mA)
- Device Standby Time (0-720 min)
- AC Standby Time (0-720 min)
- Car Standby Time (0-720 min)
- Screen Timeout (0-300 sec)
- Screen Brightness (0-3)

**Select Controls:**
- AC Output Frequency (50Hz / 60Hz)
- Update Interval (5s / 10s / 15s / 30s / 60s)

**Binary Sensors:**
- AC Charging
- Solar Charging
- Discharging
- DC Output Enabled
- AC Output Enabled

### 🔧 Technical Details

- Delta 2 uses unique API format: `moduleType` (1=PD, 2=BMS, 5=MPPT) + `operateType`
- New entity classes: `EcoFlowDelta2Switch`, `EcoFlowDelta2Number`, `EcoFlowDelta2Select`
- Data keys use flat dot-notation: `pd.wattsInSum`, `bms_bmsStatus.soc`, `mppt.inWatts`, etc.
- Tested with real Delta 2 device (SN: R331ZEB4ZECD0090)

## [1.5.1] - 2026-01-21

### 🗑️ Removed

#### River 3 and River 3 Plus Support Removed

- **River 3** and **River 3 Plus** device support has been removed from the codebase
- These devices are **not supported** by EcoFlow Developer REST API (returns error 1006)
- This is a limitation on EcoFlow's side - these devices are not in the official REST API supported devices list
- All River 3 device definitions, constants, and mappings have been removed
- Updated error messages to clarify this limitation
- Updated README.md to reflect current supported devices

### 📝 Documentation

- Updated README.md to remove River 3/River 3 Plus from supported devices list
- Added clarification about EcoFlow REST API limitations

## [1.5.0] - 2026-01-21

### ✨ New Features

#### Extra Battery Sensors for Delta Pro 3

- **SOC** - State of Charge (%)
- **SOH** - State of Health (%)
- **Design Capacity** - Original battery capacity (Ah)
- **Full Capacity** - Current full capacity (Ah)
- **Remain Capacity** - Remaining capacity (Ah)

Data is decoded from `plugInInfo4p8xResv.resvInfo` array:

- SOC/SOH decoded as IEEE 754 float
- Capacity values converted from mAh to Ah
- Automatic fallback: uses port 4p82 as primary, falls back to 4p81

#### River 3 Plus Device Support

- Full support using same API as River 3
- All sensors, switches, numbers, selects, binary sensors

#### Region Selection

- **EU** - api-e.ecoflow.com
- **US** - api.ecoflow.com
- Select region during setup

### 🐛 Fixes

- Fixed sensor naming - descriptive names now shown in Home Assistant
- Fixed Extra Battery sensors showing Unknown when battery connected to port 2

### 📚 Documentation

- Updated README with new devices
- Added region support documentation
- Updated supported devices table

### 🧪 Testing

- Tested with real Delta Pro 3 device
- Verified Extra Battery data decoding
- Confirmed River 3 Plus compatibility

## [1.3.1] - 2025-12-11

### Added
- Clear connection status logs at startup:
  - ✅ REST API connected
  - ✅ MQTT connected (hybrid mode)
  - 🔋 EcoFlow API integration ready

## [1.3.0] - 2025-12-11

### 🚀 Major Features

- **Hybrid REST API + MQTT Support** - Real-time updates via MQTT with REST API for device control
  - ⚡ Real-time sensor updates (no polling delay)
  - 🔧 Reliable device control via REST API
  - 🔄 Automatic fallback to REST-only if MQTT unavailable
  - 📉 Reduced API calls when MQTT is active

- **Battery Cycles Sensor** - Track battery charge/discharge cycles via MQTT
  - Uses `cycles` field from BMS data
  - StateClass.TOTAL_INCREASING for proper statistics

- **Energy Dashboard Integration** - Automatic kWh sensors from power sensors
  - Total Input/Output Energy sensors (enabled by default)
  - AC Input Energy sensor (disabled by default)
  - Compatible with HA Energy Dashboard

### 🔧 Improvements

- **HACS Icon** - Custom SVG icon for HACS integration display
- **Cleaned up logging** - Removed verbose debug logs, only important messages remain
- **Thread-safe MQTT** - Fixed callback thread safety for proper HA event loop integration
- **SSL async initialization** - Moved blocking SSL context creation to executor
- **IntegrationSensor fix** - Fixed missing `hass` argument for energy sensors
- **HA 2025.4 compatibility** - Removed deprecated `async_add_job`

### 📁 Home Assistant Automations (in `automations/` folder)

- **Smart Charging** - Adaptive charging based on Yasno power outage schedule
- **Power Switch** - Grid/battery switch notifications
- **Battery Alerts** - Low/critical battery, high temp, full charge notifications

### 📝 Other Changes

- Non-Commercial License
- Repository renamed to `ecoflow-api-mqtt`
- Direct HACS installation link in README
- Hybrid Mode documentation
- Removed development helper scripts

## [1.3.0-beta12] - 2025-12-10

### Fixed
- 🔋 **Battery Cycles Sensor** - Added missing "key" field for cycles mapping
  - MQTT sends `cycles` field (not `bmsCycles`)
  - Now correctly maps `cycles` from MQTT to Battery Cycles sensor
  - Cycles sensor now shows data from MQTT (e.g., 26, 30 cycles detected)

### Note
Delta Pro 3 has multiple batteries (extra batteries), each with its own cycles count:
- Battery 1: `bmsSn: MR52Z1S5PG8R0374` - cycles: 26
- Battery 2: `bmsSn: MR51PA08PG830151` - cycles: 30
Currently shows the last received cycles value. Future enhancement: separate sensors per battery.

## [1.3.0-beta11] - 2025-12-10

### Fixed
- 🐛 **Thread Safety** - Fix async_write_ha_state called from wrong thread
  - MQTT callback runs in different thread than Home Assistant event loop
  - Use hass.async_add_job() to schedule updates in correct event loop
  - Prevents Home Assistant crashes and data corruption
  - Fixes "calls async_write_ha_state from a thread other than the event loop" warning

## [1.3.0-beta10] - 2025-12-10

### Fixed
- 🐛 **MQTT Message Parsing** - Handle MQTT messages without 'params' wrapper
  - EcoFlow MQTT sends data directly (not wrapped in params)
  - Now correctly processes both wrapped and unwrapped formats
  - Fixes "Quota message missing 'params'" warnings in logs
  - MQTT real-time updates now working correctly! ✅

## [1.3.0-beta9] - 2025-12-10

### Added
- ⚡ **Automatic Energy Sensors** - Full integration with Home Assistant Energy Dashboard
  - Automatically creates kWh sensors from power (W) sensors
  - Total Input Energy sensor (enabled by default)
  - Total Output Energy sensor (enabled by default)
  - AC Input Energy sensor (disabled by default)
  - Compatible with HA Energy Dashboard for tracking consumption and generation
- 📊 **Power Difference Sensor** - Shows net power flow (Input - Output)
  - Positive value = charging/receiving power
  - Negative value = discharging/consuming power
  - Useful for Energy Dashboard "Now" tab
- 🗄️ **Recorder Exclusions** - Database optimization
  - Technical attributes excluded from database history
  - Reduces database size and improves performance
  - Excludes: mqtt_connected, last_update_time, device_info, etc.

### Changed
- 📦 **Energy Dashboard Integration** - Power sensors now automatically integrate to energy
- 🔧 **Sensor Architecture** - Added base classes for energy and power difference sensors

## [1.3.0-beta8] - 2025-12-10

### Fixed
- 🔐 **MQTT Authentication** - Now automatically fetches `certificateAccount` and `certificatePassword` from EcoFlow API
  - Added `get_mqtt_credentials()` method to retrieve proper MQTT credentials
  - MQTT topics now use correct `certificateAccount` instead of email
  - Fixes "Connection Refused - not authorized (code 5)" error
- 📡 **MQTT Topics** - Proper certificateAccount used in all MQTT topics
  - `/open/{certificateAccount}/{sn}/quota` - Uses API-provided certificateAccount
  - `/open/{certificateAccount}/{sn}/status` - Device online/offline status
  - `/open/{certificateAccount}/{sn}/set` - Send commands
  - `/open/{certificateAccount}/{sn}/set_reply` - Command responses

### Changed
- 🔧 **MQTT Setup** - Integration now fetches MQTT credentials automatically on startup
- 📝 **Options Flow** - MQTT username/password fields now optional (auto-fetched from API)

## [1.3.0-beta7] - 2025-12-10

### Added
- 📊 **Complete GetAllQuotaResponse field mapping** - All fields from API documentation now mapped
  - Device status fields: errcode, devSleepState, devStandbyTime, bleStandbyTime
  - Battery status: bmsChgDsgState, cmsChgDsgState, cmsBmsRunState
  - Generator settings: cmsOilSelfStart, cmsOilOffSoc, cmsOilOnSoc
  - Power flow: powGet5p8, powGet4p81, powGet4p82, powGetAcLvTt30Out
  - Plug-in info: All plugInInfo* numeric/string fields (50+ fields)
  - Flow info: All flowInfo* enum sensors (17 fields)
  - Settings: fastChargeSwitch, energyBackupEn, llcHvLvFlag, acLvAlwaysOn, acHvAlwaysOn, etc.
- 🔧 **Fixed MQTT topics** - Corrected topic format from `/app/...` to `/open/{certificateAccount}/{sn}/...`
  - `/open/{certificateAccount}/{sn}/quota` - Device quota updates
  - `/open/{certificateAccount}/{sn}/status` - Device online/offline status
  - `/open/{certificateAccount}/{sn}/set` - Send commands
  - `/open/{certificateAccount}/{sn}/set_reply` - Command responses
- 📡 **Improved MQTT message handling** - Proper parsing for quota, status, and set_reply topics

### Changed
- 🔄 **MQTT client** - Updated to use correct EcoFlow MQTT protocol format
- 📝 **Documentation** - Updated MQTT protocol comments with correct topic structure

## [1.3.0-beta1] - 2025-12-10

### Added
- 🚀 **Hybrid REST API + MQTT Support** - Best of both worlds!
  - ⚡ **Real-time updates via MQTT** - Instant sensor updates without polling
  - 🔧 **Device control via REST API** - Reliable command execution
  - 🔄 **Automatic fallback** - Seamlessly falls back to REST if MQTT unavailable
  - 📊 **Battery Cycles sensor** - Now available via MQTT (`bmsCycles`)
  - 🎛️ **MQTT configuration** - Enable/disable MQTT through Settings → Configure
- 📡 **MQTT Client** - Full WebSocket-based MQTT implementation
  - Broker: `mqtt.ecoflow.com:8883` (TLS)
  - Real-time device status updates
  - Automatic reconnection
- 🔀 **Hybrid Coordinator** - Intelligent data merging
  - MQTT data priority (more real-time)
  - REST API fallback for reliability
  - Reduced REST polling when MQTT active (4x less frequent)

### Changed
- 📦 **Dependencies** - Added `paho-mqtt>=1.6.1` for MQTT support
- 🔧 **Coordinator** - Can now be hybrid (REST+MQTT) or REST-only
- ⚙️ **Configuration** - MQTT settings in OptionsFlow (Settings → Configure)

### Technical Details
- ✅ **New files**: `mqtt_client.py`, `hybrid_coordinator.py`
- ✅ **MQTT authentication**: Uses EcoFlow account credentials
- ✅ **Connection modes**: `hybrid`, `mqtt_standby`, `rest_only`
- ✅ **Graceful degradation**: Works without MQTT if not configured

### Beta Notes
- ⚠️ **Beta release** - Please test and report issues
- 🧪 **MQTT is optional** - Integration works fine without it
- 📝 **Feedback needed**: MQTT connection stability, data accuracy
- 🔍 **Known limitations**: MQTT credentials must be EcoFlow account (email/password)

## [1.2.1] - 2025-12-10

### Added
- 🎛️ **Dynamic Update Interval Control** - New select entity for runtime interval changes
  - `select.ecoflow_delta_pro_3_update_interval` - Change polling frequency on the fly
  - Options: 5s (Fast), 10s, 15s (Recommended), 30s, 60s (Slow)
  - Changes apply immediately without restart
  - Settings persist after Home Assistant restart
- ⚙️ **OptionsFlow Configuration** - Configure update interval through Settings → Configure

### Fixed
- 🐛 **OptionsFlow 500 error** - Fixed "Config flow could not be loaded" error
  - Removed unused `UPDATE_INTERVAL_OPTIONS` import
  - Simplified options handling logic

### Technical Details
- ✅ **Coordinator enhancement** - Added `async_set_update_interval()` method
- ✅ **Local settings support** - Select platform now supports both device and local settings
- ✅ **Config persistence** - Interval changes saved to config entry options

## [1.2.0] - 2025-12-10

### Added
- 🎉 **Complete Delta Pro 3 support based on real API data**
  - 📊 **40+ sensors** - All available metrics from actual Delta Pro 3 device
  - 🔋 **Battery sensors** - BMS and CMS battery data (SOC, SOH, remaining time, capacity)
  - ⚡ **Power sensors** - Total input/output, AC, Solar (HV/LV), DC outputs (12V/24V), USB-C, QC USB
  - 🌡️ **Temperature sensors** - Min/Max cell and MOSFET temperatures
  - ⚙️ **Settings sensors** - Standby times, LCD brightness, frequency
  - 🔌 **13 binary sensors** - Charging status (AC, Solar, batteries), X-Boost, GFCI, etc.
  - 🎛️ **3 switches** - X-Boost, Beep, AC Energy Saving
  - 🔢 **7 number controls** - AC charging power, charge levels, standby times, LCD brightness
- 📚 **Comprehensive documentation**
  - 📖 **DELTA_PRO_3_API_MAPPING.md** - Complete API reference with real data examples
  - 🔍 **MQTT vs REST API comparison** - Detailed analysis and recommendations
  - 📝 **Cycles explanation** - Why cycles are not available in REST API and alternatives
- 🧪 **Template sensors examples** - Ready-to-use Home Assistant templates for:
  - 🔄 Estimated cycles calculation based on SOH
  - 💚 Battery health status
  - ⚡ Charging status with multiple sources
  - 📊 Net power flow
  - ⏱️ Runtime and charge time estimates
  - 🚨 Low battery and high temperature alerts
- 🧪 **API testing tools** - Standalone test script to verify API responses

### Changed
- 🔄 **Sensor definitions updated** - All sensors now use actual API keys from real Delta Pro 3
- 📊 **Sensor naming** - More descriptive names (e.g., "Battery Level (BMS)" vs "Battery Level (CMS)")
- 📝 **Documentation improvements** - Based on actual device testing (SN: MR51ZES5PG860274)

### Technical Details
- ✅ **Tested with real device** - DELTA Pro 3 (online, SOH 100%, 8192Wh capacity)
- 📡 **API endpoint verified** - `/iot-open/sign/device/quota/all`
- 🔐 **Authentication working** - EcoFlow Developer API (api-e.ecoflow.com)
- 🌍 **Timezone support** - UTC timezone handling (Europe/Kiev tested)

### Notes
- ⚠️ **Cycles not available** - REST API does not provide cycle count (only available via MQTT)
- 💡 **Alternative solution** - Template sensor for estimated cycles based on SOH included
- 📖 **Why REST API?** - More stable and officially supported than MQTT (see documentation)

## [1.1.4] - 2024-12-10

### Fixed
- 🐛 **Binary sensors fixed** - Corrected API key mappings for all binary sensors
- 🔋 **Charging/Discharging detection** - Now uses correct `powInSumW` and `powOutSumW` keys
- 🔌 **AC Input Connected** - Fixed to use `powGetAcIn` instead of non-existent `acInPower`
- ☀️ **Solar Connected** - Fixed to use `powGetPvH` instead of non-existent `solarInPower`
- 🪫 **Battery Low/Full** - Fixed to use `bmsBattSoc` instead of non-existent `soc`
- 🌡️ **Over Temperature** - Fixed to use `bmsMaxCellTemp` instead of non-existent `bmsTemp`
- ⚡ **Threshold adjustment** - Changed charging/discharging detection threshold from 0W to 10W to avoid false positives

## [1.1.3] - 2024-12-09

### Fixed
- 🐛 **Timestamp sensor error** - Fixed "str object has no attribute 'tzinfo'" error for timestamp sensors
- 🕐 **Datetime conversion** - Timestamp sensors now correctly return timezone-aware datetime objects

## [1.1.2] - 2024-12-09

### Fixed
- 🐛 **ACTUALLY fixed signature generation for PUT requests** - Now correctly includes flattened JSON body parameters in signature calculation, as required by EcoFlow API documentation
- 🔧 **Boolean conversion** - Boolean values now converted to lowercase strings (true/false) in signature
- ✅ **Tested and verified** - AC Charging Power control tested successfully (1200W → 1500W)

### Added
- 🧪 **Test script** - Added `test_set_ac_power.py` for manual testing of device controls

## [1.1.1] - 2024-12-09

### Fixed
- 🐛 **Critical fix: Signature generation for PUT requests** - Fixed "signature is wrong" error (code 8521) when controlling devices. PUT requests now correctly generate signature only from auth parameters, not from JSON body content.

## [1.1.0] - 2024-12-09

### Added
- 🏗️ **Improved code structure** - Better organization of entity management
- 📝 **Enhanced translations** - Updated English and Ukrainian translations
- 🧪 **Better test coverage** - Improved test structure and documentation
- 🔧 **Configuration improvements** - Enhanced config flow and diagnostics

### Changed
- 🔄 **Entity management** - Improved binary sensor, number, select, and switch entities
- 📊 **Coordinator updates** - Better device state handling
- 📖 **Code quality** - Refactored code for better maintainability

### Fixed
- 🐛 **Minor bug fixes** - Various small improvements and fixes

## [1.0.8] - 2024-12-09

### Added
- 🏗️ **Modular device structure** - Device-specific logic organized in `devices/` subdirectories
- 🧪 **Comprehensive test suite** - Unit tests for API client, config flow, and integration structure
- 📊 **Structure validation script** - Quick check for file structure without dependencies (`check_structure.py`)
- 📚 **Testing documentation** - Detailed testing guide in `tests/README.md`
- ⚙️ **Configurable update interval** - Users can now choose update frequency (5/10/15/30/60 seconds)
- 🔄 **Immediate state refresh** - After control actions, state updates after 2 seconds
- 📝 **Changelog** - This file to track all changes

### Changed
- 🔧 **Default update interval** - Changed from 30s to 15s for better responsiveness
- 📁 **Project structure** - Device-specific constants moved to `devices/delta_pro_3/`
- 📖 **README updates** - Added testing section, options configuration, and updated troubleshooting

### Fixed
- 🐛 **Nonce generation** - Corrected to generate 6-digit nonce (was 16 characters)
- 🔐 **API signature** - Fixed signature generation to match EcoFlow API requirements
- ⏱️ **Timestamp issues** - Ensured fresh timestamps for each API request

## [1.0.7] - 2024-12-08

### Added
- 📊 **85+ sensors** - All available data points from Delta Pro 3 API
- 🎛️ **23 control entities** - 8 switches, 12 numbers, 4 selects
- 🇺🇦 **Ukrainian translations** - Full localization for all entities

### Changed
- 🔄 **Sensor definitions** - Based on real API response keys
- 🛠️ **Control commands** - Updated to match EcoFlow API documentation

## [1.0.6] - 2024-12-07

### Fixed
- 🔧 **Content-Type header** - Conditionally added based on HTTP method (GET vs POST/PUT)
- 🌐 **API base URL** - Corrected to `https://api-e.ecoflow.com`

## [1.0.5] - 2024-12-06

### Fixed
- 🔐 **GET request parameters** - Parameters now in URL query string, not request body
- 📝 **Signature generation** - Parameter order corrected (request params first, then auth params)

## [1.0.4] - 2024-12-05

### Fixed
- 🔐 **API authentication** - Initial fix for signature generation

## [1.0.3] - 2024-12-04

### Added
- 🔍 **HACS validation** - Repository topics for HACS discovery

### Fixed
- 📦 **HACS download** - Removed `zip_release` from `hacs.json`

## [1.0.2] - 2024-12-03

### Added
- ✨ **Manual device entry** - Users can manually enter device serial number and type
- 📋 **Device selection menu** - Choose between auto-discovery and manual entry

### Fixed
- 🔧 **Config flow** - Improved error handling and user experience

## [1.0.1] - 2024-12-02

### Added
- 🔧 **Config flow improvements** - Better device discovery

### Fixed
- 🐛 **Initial setup issues** - Various bug fixes

## [1.0.0] - 2024-12-01

### Added
- 🎉 **Initial release**
- ✅ **Delta Pro 3 support** - Full support for EcoFlow Delta Pro 3
- 🔌 **Basic sensors** - Battery level, power, temperature, etc.
- 🎛️ **Basic controls** - AC/DC output, charging power, X-Boost
- 🔧 **Config flow** - Easy setup through Home Assistant UI
- 📡 **Official API** - Uses EcoFlow Developer API
- 🇺🇦 **Ukrainian localization** - Translations for Ukrainian language

---

## Legend

- 🎉 Major features
- ✅ Features
- 🔧 Improvements
- 🐛 Bug fixes
- 🔐 Security
- 📝 Documentation
- 🧪 Testing
- 🇺🇦 Localization
- 📊 Sensors
- 🎛️ Controls
- 🏗️ Architecture
- 🌐 API


