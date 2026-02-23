# AqualinkD Architecture Overview

AqualinkD is a Linux daemon that acts as a gateway between Jandy/Pentair pool control panels and network services (HTTP, WebSocket, MQTT). It communicates with pool equipment over an RS485 serial bus and exposes the panel state to web UIs, home automation systems (HomeKit, Home Assistant), and MQTT clients.

## System Data Flow

```
                        ┌─────────────────────────────────┐
                        │        Pool Equipment           │
                        │  (Pumps, Heaters, Lights, SWG)  │
                        └────────────┬────────────────────┘
                                     │ RS485 Bus
                                     │
                        ┌────────────▼────────────────────┐
                        │     Jandy/Pentair Panel         │
                        │  (Allbutton, OneTouch, PDA, …)  │
                        └────────────┬────────────────────┘
                                     │ RS485 Serial
                                     │ /dev/ttyUSB0
                        ┌────────────▼────────────────────┐
                        │      aq_serial.c                │
                        │  Open port, read/write packets, │
                        │  CRC validation, framing        │
                        └────────────┬────────────────────┘
                                     │ Raw packets
                        ┌────────────▼────────────────────┐
                        │   Protocol Handlers             │
                        │  allbutton.c  onetouch.c        │
                        │  iaqtouch.c   pda.c             │
                        │  iaqualink.c  serialadapter.c   │
                        │  devices_jandy.c                │
                        │  devices_pentair.c              │
                        └────────────┬────────────────────┘
                                     │ Parsed state
                        ┌────────────▼────────────────────┐
                        │   struct aqualinkdata           │
                        │  (Central state in aqualink.h)  │
                        │  Temps, LEDs, pumps, SWG, …     │
                        └───┬────────┬───────────┬────────┘
                            │        │           │
               ┌────────────▼──┐ ┌───▼────────┐ ┌▼───────────────┐
               │  HTTP/WS API  │ │    MQTT     │ │  Home Assistant│
               │net_services.c │ │net_services │ │ mqtt_discovery │
               │json_messages.c│ │  .c (mqtt)  │ │     .c         │
               └───────┬───────┘ └──────┬──────┘ └───────┬────────┘
                       │                │                 │
               ┌───────▼────┐   ┌───────▼──────┐ ┌───────▼────────┐
               │  Web UI /  │   │ MQTT Broker  │ │ HA Discovery   │
               │  REST API  │   │              │ │ auto-config    │
               └────────────┘   └──────────────┘ └────────────────┘
```

## Daemon Lifecycle

The entry point is `main()` in `aqualinkd.c`:

1. **Argument parsing** - `-c` config file path, `-d` foreground mode, `-v` log verbosity
2. **Root check** - Serial port access requires root privileges
3. **Config loading** - `init_config()` sets defaults, then `read_config()` parses `aqualinkd.conf`
4. **Daemonize** - Forks to background unless `-d` flag is set
5. **Signal handlers** - Registers handlers for SIGINT, SIGTERM, SIGQUIT, SIGRESTART, SIGRUPGRADE
6. **Network startup** - Initializes Mongoose HTTP/WebSocket server and MQTT client
7. **Serial port open** - `init_serial_port()` opens configured serial device
8. **Device ID probing** - If `device_id` is `0xFF` (auto) or `0x00` (find unused), probes the RS485 bus to discover/claim an address
9. **Panel initialization** - Based on detected panel type, initializes the appropriate protocol handler (PDA, OneTouch, iAqualink Touch, RS Serial Adapter)
10. **Sensor thread** - Starts background thread for GPIO/1-Wire sensor readings if configured
11. **Main loop** - Infinite packet processing loop (see below)
12. **Shutdown** - On signal: closes serial port, stops network services, stops sensor thread, exits

### Main Loop

The main loop in `main_loop()` runs continuously:

- **Serial monitoring** - Detects disconnection or excessive blank reads; reopens with backoff
- **Packet reception** - `get_packet()` reads from serial, validates checksum
- **Packet routing** - Checks destination ID against configured device IDs:
  - **Emulation mode** (packet addressed to us): Calls the appropriate protocol handler (`process_allbutton_packet()`, `process_onetouch_packet()`, etc.) and sends ACK
  - **Monitoring mode** (packet for other devices, `read_RS485_devmask` enabled): Calls `processJandyPacket()` or `processPentairPacket()` to passively extract device state
- **Delayed commands** - Executes queued commands after a 2-second delay
- **State broadcast** - When `aqualinkdata.is_dirty` is set, broadcasts JSON to all WebSocket clients and publishes MQTT updates

## Module Map

### Core

| File | Purpose |
|------|---------|
| `aqualinkd.c` | Main daemon: argument parsing, initialization, main packet loop, shutdown |
| `aqualink.h` | Central data structures (`struct aqualinkdata`, `struct aqconfig`) |
| `config.c` / `config.h` | Parse `aqualinkd.conf`, manage runtime configuration (~60 parameters) |
| `version.h` | Version string macro |

### Serial I/O

| File | Purpose |
|------|---------|
| `aq_serial.c` / `aq_serial.h` | Open/close serial port, read/write packets, CRC validation, framing (0xFF 0x00 sync), non-blocking I/O with `select()` |
| `packetLogger.c` / `packetLogger.h` | Write raw bytes and parsed packets to log files for debugging |

### Protocol Handlers

Each panel type has dedicated handler files. These are **off-limits for external contributions** due to hardware-dependent behavior.

| Panel Type | ID Range | Files |
|------------|----------|-------|
| Allbutton (older Jandy) | 0x08-0x0B | `allbutton.c/h`, `allbutton_aq_programmer.c` |
| OneTouch | 0x40-0x43 | `onetouch.c/h`, `onetouch_aq_programmer.c` |
| iAqualink Touch | 0x30-0x33 | `iaqtouch.c/h`, `iaqtouch_aq_programmer.c` |
| PDA | 0x60-0x63 | `pda.c/h`, `pda_menu.c/h`, `pda_aq_programmer.c` |
| RS Serial Adapter | 0x48-0x49 | `serialadapter.c/h` |
| iAqualink | 0xA0-0xA3 | `iaqualink.c/h` |

**Common protocol layer:**

| File | Purpose |
|------|---------|
| `aq_panel.c` / `aq_panel.h` | Unified panel initialization, button/LED management |
| `aq_programmer.c` / `aq_programmer.h` | Command queuing and execution across protocol types |
| `rs_msg_utils.c` / `rs_msg_utils.h` | RS protocol message parsing utilities |
| `rs_devices.h` | Device ID ranges and type mappings |
| `auto_configure.c` | Auto-discovery of panel type and device ID on RS485 bus |

### Device Handling

| File | Purpose |
|------|---------|
| `devices_jandy.c` / `devices_jandy.h` | Read-only monitoring of Jandy devices (SWG, heat pump, heater, chemical feeder) |
| `devices_pentair.c` / `devices_pentair.h` | Read-only monitoring of Pentair variable-speed pumps (RPM, GPM, watts) |
| `color_lights.c` / `color_lights.h` | Light mode/color programming (Jandy Color, IntelliBrite, Hayward, etc.) |
| `sensors.c` / `sensors.h` | GPIO and 1-Wire temperature sensor reading (separate thread) |

### Network Services

| File | Purpose |
|------|---------|
| `net_services.c` / `net_services.h` | Mongoose HTTP/WebSocket server, MQTT client, API endpoint routing |
| `mongoose.c` / `mongoose.h` | Embedded Mongoose web server library |
| `json_messages.c` / `json_messages.h` | JSON response builders for status, devices, config, schedules |
| `web_config.c` / `web_config.h` | Dynamic web configuration generation |
| `net_interface.c` / `net_interface.h` | Network interface info (IP, DNS) |
| `mqtt_discovery.c` / `mqtt_discovery.h` | Home Assistant MQTT auto-discovery messages |
| `aq_mqtt.h` | MQTT topic name defines and state constants |

### Scheduling and Timers

| File | Purpose |
|------|---------|
| `aq_scheduler.c` / `aq_scheduler.h` | Cron-like job scheduling for device actions |
| `aq_timer.c` / `aq_timer.h` | Command duration timers (e.g., pump run times) |
| `debug_timer.c` / `debug_timer.h` | Performance instrumentation macros |
| `timespec_subtract.c` | Timing utility |

### Utilities

| File | Purpose |
|------|---------|
| `utils.c` / `utils.h` | Logging (syslog), string/buffer helpers, error handling |
| `aq_systemutils.c` | System-level file I/O, mount operations |
| `epump.h` | Extended pump definitions |

### Testing and Simulation

| File | Purpose |
|------|---------|
| `dummy_device.c` | Stub device for testing |
| `dummy_reader.c` | Log file reader for offline testing |
| `simulator.c` | Simulator command queuing |

## Configuration System

Configuration is loaded from `aqualinkd.conf` (default path `./aqualinkd.conf`, overridden with `-c`).

`config.c` processes the file:
1. `init_config()` calls `init_parameters()` to set hardcoded defaults
2. `read_config()` reads line by line, skips comments (`#`) and blank lines
3. Each `key = value` pair is passed to `setConfigValue()` for validation and storage in the global `_aqconfig_` struct

Key configuration categories:
- **Device IDs** - `device_id`, `rssa_device_id`, `extended_device_id`
- **Serial** - `serial_port` (e.g., `/dev/ttyUSB0`)
- **HTTP** - `listen_address` (e.g., `0.0.0.0:80`), `web_directory`
- **MQTT** - `mqtt_address`, `mqtt_user`, `mqtt_passwd`, `mqtt_aq_topic`
- **Logging** - `log_level`, `log_protocol_packets`, `log_raw_bytes`
- **Device monitoring** - `read_RS485_devmask` (bitmask for SWG, pumps, heaters, etc.)
- **Feature flags** - `enable_scheduler`, `enable_iaqualink`, etc.

## Threading Model

- **Main thread** - Serial packet processing + Mongoose HTTP/WS/MQTT event loop
- **Sensor thread** - Periodic GPIO/1-Wire sensor readings
- **Timer threads** - Background command duration tracking
- Minimal shared-state contention; all protocol handlers write to a single `struct aqualinkdata` instance, with the `is_dirty` flag triggering network broadcasts
