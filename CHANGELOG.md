# Changelog

All notable changes to AqualinkD are documented in this file. This changelog covers v3.0.0 onward.

## [Unreleased] (v3.0.3-dev)

### Features
- **PDA PS6 HOME menu navigation**: Added `pda_force_home_onprogra` config option to force navigation back to HOME menu before starting operations, preventing panel/AqualinkD state sync issues when the panel is on the STATUS screen
- **SWG setpoint setting for PDA v1.2**: Fixed menu detection for PDA SWG configuration using loose matching for indented menu items; improved message type detection to handle different PDA panel variations; enhanced handling of dual setpoint screens (pool/spa) and single-device setups

### Bug Fixes
- **Fixed typos and logic bugs in remote_install.sh and Makefile**: Fixed `$FASE` typo to `$FALSE` for FROM_CURL flag initialization; changed `else` to `elif` in `check_can_upgrade()` to prevent unconditional error logging; fixed missing closing parenthesis in Makefile `$(shell ...)` expression; fixed `realease` typo in dockerbuildnrun target

### Build/CI
- Added `.gitignore` with `.worktrees/` exclusion
- Updated GitHub Actions release workflow configuration
- Version bumps and ARM binary rebuilds across development milestones

### Documentation
- Added "Updates in 3.0.3 (dev)" section to README with SWG PDA v1.2 fix notes

## [3.0.2] - 2026-01-06

### Features
- **Custom light show tracking**: Added infrastructure to properly track custom light shows defined in configuration, with new `isShowMode()` function to recognize dynamically-loaded shows
- **Light mode detection enhancement**: Enhanced `set_aqualinkd_light_mode_name()` to properly distinguish between standard light colors and show modes

### Bug Fixes
- **SWG device status initialization**: Fixed Salt Water Generator incorrectly reporting as enabled when no SWG device was present; added proper detection via PPM messages
- **Light program name parsing**: Fixed programs ending in " - show" suffix not being recognized; corrected parsing logic to search for the suffix rather than assuming a fixed position
- **Web configuration cleanup**: Removed incorrect "Solar_Heater" entry from `web/config.json` that used the wrong ID for the "extra_aux" device

## [3.0.1] - 2025-12-29

### Features
- **Dark mode with theme switching**: Added comprehensive dark/light theme support to the web UI with automatic detection via `prefers-color-scheme`, theme configuration option (`"theme": "auto"|"light"|"dark"`), and dedicated background images for each theme
- **Custom web configuration file support**: Added `web_config` parameter to specify an alternative `config.json` file, allowing per-instance config customization
- **Apple mobile web app metadata**: Added `apple-mobile-web-app-title` meta tag for better iOS home screen integration

### Bug Fixes
- **Sensor removal logic fix**: Fixed handling of invalid sensors where multiple blank sensors could cause array index issues, preventing valid sensors from being skipped
- **HTTP request logging**: Improved HTTP request URI logging to include URI length for better debugging
- **Web file serving refactoring**: Refactored `action_web_request()` to properly separate file serving logic, ensuring `.json` files are served without caching while regular files use caching
- **Install script improvements**: Refactored `install.sh` and `remote_install.sh` for better installation flow and reliability

### Build/CI
- Updated ARM binaries for all architectures

### Documentation
- Finalized README v3.0.0 release notes (removed "dev" marker)
- Added dark mode documentation to README

## [3.0.0] - 2025-12-14

### Features
- **HTTPS with two-way authentication**: Implemented certificate-based authentication for secure client-server communication
- **MQTT autodiscovery**: Added autodiscovery support for HomeAssistant, Domoticz, Hubitat, and OpenHAB hubs
- **PDA color light and dimmer support**: Enhanced color light support for PDA panels including dimmer lights
- **WebConfig editing in AqManager**: Added ability to edit web configuration directly in the UI; migrated config.js to config.json format
- **Version selection for upgrades**: Enhanced upgrade functionality to allow selecting specific versions including development releases
- **Autoconfiguration for PDA panels**: PDA panels now support autoconfiguration to detect panel size and type
- **Dynamic tile color updates**: Tiles for pH/ORP/PPM now change color and label when values are outside optimal range
- **Serial optimization for AqualinkD HAT**: Optimized serial communication for HAT-based installations
- **Hardware detection enhancements**: Improved detection of external sensors and HomeKit compatibility
- **UI/WebSocket optimization**: Reduced unnecessary MQTT, WebSocket, and WebUI communications
- **HTTP server caching improvements**: Enhanced caching strategy for better UI configuration update handling

### Bug Fixes
- **External sensors and HomeKit issues**: Fixed problems with external sensor detection and HomeKit compatibility
- **SWG auto-finding**: Resolved Salt Water Generator autodiscovery problems
- **Jandy Infinite water color light support**: Added preliminary support for Jandy Infinite brand water color lights
- **PDA menu navigation and highlighting**: Fixed PDA panel menu navigation logic including highlight detection and menu transitions
- **Device configuration parsing**: Enhanced device configuration handling in devices_jandy.c
- **Serial communication edge cases**: Fixed edge cases in serial communication for various Jandy panel types
- **Removed stray debug printf statements**: Cleaned up debug output in color_lights.c

### Build/CI
- **Renamed serial_logger to rs485mon**: Rebranded binary and source files across all architectures
- Updated Makefile with build process improvements for binary rebranding
- Updated Dockerfile and Dockerfile.buildx for multi-architecture builds
- Enhanced remote_install.sh with improved tar extraction handling and version parameter logging
- Updated compiled release binaries for ARM64 and ARMHF

### Documentation
- Updated README with v3.0.0 feature list and configuration changes
- Added config.json migration guide (from config.js)
- Updated web UI pages (index.html, aqmanager.html) with new features
