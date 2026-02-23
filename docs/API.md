# AqualinkD API Reference

AqualinkD exposes three network interfaces: an HTTP REST API, a WebSocket JSON protocol, and an MQTT client. All are served by the Mongoose embedded web server in `net_services.c`.

## HTTP API

The HTTP server listens on the address configured by `listen_address` in `aqualinkd.conf` (default: `0.0.0.0:80`). Static web UI files are served from the configured `web_directory` (default: `/var/www/aqualinkd/`).

HTTPS is supported when certificates are placed in the configured `cert_dir` (`crt.pem`, `key.pem`, optional `ca.pem` for mutual TLS).

### Endpoints

All API endpoints are under the `/api/` prefix. The value parameter can be passed as a query string (`?value=X`) or in the request body (`value=X` or `{"value":"X"}`).

#### Read-Only Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/devices` | GET | JSON list of all devices with current states and capabilities |
| `/api/status` | GET | Full system status JSON (temperatures, LED states, SWG, etc.) |
| `/api/homebridge` | GET | Device list in Homebridge-compatible JSON format |
| `/api/dynamicconfig` | GET | Dynamic web configuration JSON |
| `/api/schedules` | GET | Current schedules JSON |
| `/api/config` | GET | Current application configuration JSON |
| `/api/config/download` | GET | Download the raw `aqualinkd.conf` file |

#### Write Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/schedules/set` | POST | Save schedule data (JSON body) |
| `/api/config/set` | POST | Save application configuration (JSON body) |
| `/api/webconfig/set` | POST | Save web UI configuration (JSON body) |
| `/api/set_date_time` | PUT | Sync panel date/time |

#### Device Control

Device control uses the pattern `/api/<device_path>/set` with a `value` parameter.

| URI Pattern | Description | Example Value |
|-------------|-------------|---------------|
| `<button_name>/set` | Toggle device on/off | `on`, `off`, `1`, `0` |
| `<button_name>/timer/set` | Set device timer | duration in minutes |
| `<heater>/setpoint/set` | Set heater temperature setpoint | `85` (degrees) |
| `SWG/Percent/set` | Set SWG percent | `0`-`100` |
| `SWG/Percent_f/set` | Set SWG percent (with temp unit conversion) | `0`-`100` |
| `SWG/Boost/set` | Toggle SWG boost | `on`, `off` |
| `<light>/color/set` | Set light color mode | color mode number |
| `<light>/program/set` | Set light program | program index |
| `<light>/brightness/set` | Set light brightness | `0`-`100` |
| `Pump_<N>/RPM/set` | Set pump RPM | RPM value |
| `Pump_<N>/GPM/set` | Set pump GPM | GPM value |
| `Pump_<N>/Speed/set` | Set pump speed percentage | `0`-`100` |
| `CHEM/ORP/set` | Set ORP value | ORP number |
| `CHEM/Ph/set` | Set pH value | pH number |
| `Freeze/setpoint/set` | Set freeze protection setpoint | temperature |

#### AQ Manager Endpoints (requires `AQ_MANAGER` build flag)

| Endpoint | Access | Description |
|----------|--------|-------------|
| `/api/aqmanager` | WebSocket only | AQ Manager status JSON |
| `/api/logfile/download` | GET | Download syslog output |
| `/api/logfile/download/<lines>` | GET | Download last N lines of log |
| `/api/setloglevel` | WebSocket only | Set system log level |
| `/api/addlogmask` | WebSocket only | Add debug log mask |
| `/api/removelogmask` | WebSocket only | Remove debug log mask |
| `/api/restart` | WebSocket only | Send SIGRESTART to daemon |
| `/api/installrelease` | WebSocket only | Trigger release upgrade |
| `/api/seriallogger` | WebSocket only | Start rs485mon serial logger |

#### Debug Endpoints (non-AQ_MANAGER builds)

| Endpoint | Description |
|----------|-------------|
| `/api/debug/start` | Start inline debug |
| `/api/debug/stop` | Stop inline debug |
| `/api/debug/serialstart` | Start inline serial debug |
| `/api/debug/serialstop` | Stop inline serial debug |
| `/api/debug/clean` | Clean inline debug data |
| `/api/debug/download` | Download debug data |

### Response Format

**Success (HTTP 200):**
- Action endpoints return: `"Ok"` (text/plain)
- Data endpoints return: JSON object (application/json)

**Error (HTTP 400):**
- `"Unknown command"` - Unrecognized URI
- `"No matching Device found"` - Device name not found
- `"No matching Pump found"` - Pump number not found
- `"No programable light found"` - Light device not found
- `"No device for setpoint found"` - Setpoint target not found
- `"Pump VS programs not supported yet"` - Unsupported pump feature
- `"Invalid value"` - Value out of range or wrong type
- `"Didn't understand request"` - Malformed request

### Caching

API responses use `Cache-Control: no-cache, no-store, must-revalidate`. Static web assets use `Cache-Control: public, max-age=604800, immutable` (7 days).

---

## WebSocket Protocol

WebSocket connections are upgraded from standard HTTP connections. Connect to the same address/port as the HTTP server.

### Message Format

Messages are JSON objects with up to 4 key-value pairs:

```json
{
  "uri": "<endpoint_path>",
  "value": "<value>"
}
```

The `uri` field maps to the same endpoint paths as the HTTP API (without the `/api/` prefix). The `value` field carries the command value.

### Server Broadcasts

When device state changes (the `is_dirty` flag is set), the server broadcasts a status JSON to all connected WebSocket clients:

```json
{
  "type": "status",
  "version": "3.0.3",
  "date": "09/01/24 TUE",
  "time": "3:16 PM",
  "temp_units": "F",
  "air_temp": "96",
  "pool_temp": "86",
  "pool_htr_set_pnt": "85",
  "spa_htr_set_pnt": "100",
  "freeze_protection": "off",
  "frz_protect_set_pnt": "36",
  "leds": {
    "pump": "on",
    "spa": "off",
    "aux1": "off"
  },
  "swg": {
    "state": "on",
    "percent": "60",
    "ppm": "2200"
  }
}
```

### WebSocket Responses

```json
{"message": "ok"}
{"message": "Bad request"}
{"message": "<error_message>"}
```

### WebSocket-Only Endpoints

These endpoints are only accessible via WebSocket (rejected over HTTP):

- `simulator` (with variants: `simulator/onetouch`, `simulator/allbutton`, `simulator/aquapda`, `simulator/iaqtouch`)
- `simcmd`
- `restart`
- `installrelease`
- `seriallogger`
- `setloglevel`, `addlogmask`, `removelogmask` (AQ_MANAGER builds)

---

## MQTT

AqualinkD connects to an MQTT broker as a client, publishing device state and subscribing to commands. MQTT is optional and only active when `mqtt_address` is configured.

### MQTT Configuration

These are the exact configuration keys as used in `aqualinkd.conf` (defined in `config.h`):

| Config Key | Default | Description |
|------------|---------|-------------|
| `mqtt_address` | *(none)* | MQTT broker address (e.g., `localhost:1883`) |
| `mqtt_user` | *(none)* | MQTT broker username |
| `mqtt_passwd` | *(none)* | MQTT broker password |
| `mqtt_aq_topic` | `aqualinkd` | Root topic prefix for all AqualinkD messages |
| `mqtt_discovery_topic` | `homeassistant` | Home Assistant MQTT discovery prefix |
| `mqtt_discovery_use_mac` | `true` | Include MAC address in discovery entity IDs |
| `mqtt_timed_update` | `true` | Publish full state every 300 seconds |
| `mqtt_cert_dir` | *(none)* | TLS certificate directory for encrypted MQTT |
| `mqtt_convert_temp_to_c` | `true` | Convert temperatures to Celsius for MQTT payloads |

### Connection

- **Client ID**: Auto-generated
- **QoS**: 1 (at least once) for published messages
- **Retain**: true (messages retained on broker)
- **Subscription**: `<mqtt_aq_topic>/#` (all subtopics)
- **Last Will**: Topic `<mqtt_aq_topic>/Alive`, payload `"0"` (offline)
- **On connect**: Publishes `"1"` to `<mqtt_aq_topic>/Alive`

### Topic Reference

All topics below are relative to the configured `mqtt_aq_topic` prefix (default: `aqualinkd`).

#### Temperature Topics (publish)

| Topic | Payload |
|-------|---------|
| `Temperature/Air` | Temperature value |
| `Temperature/Pool` | Temperature value |
| `Temperature/Spa` | Temperature value |

#### Salt Water Generator Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `SWG/Percent` | Publish | SWG percent (integer 0-100) |
| `SWG/Percent_f` | Publish | SWG percent (with unit conversion) |
| `SWG/PPM` | Publish | Salt PPM (integer) |
| `SWG/PPM_f` | Publish | Salt PPM (with conversion) |
| `SWG/enabled` | Publish | `"0"` (off) or `"2"` (on) |
| `SWG/Boost` | Publish | `"0"` (off) or `"2"` (on) |
| `SWG/Boost/duration` | Publish | Boost remaining duration (seconds) |
| `SWG/fullstatus` | Publish | Extended SWG status |
| `SWG/Display_Message` | Publish | SWG status message string |
| `SWG/setpoint` | Publish | Current SWG setpoint |
| `SWG/setpoint/set` | Subscribe | Set SWG setpoint |

#### Chemistry Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `CHEM/pH` | Publish | pH value |
| `CHEM/pH_f` | Publish | pH (with conversion) |
| `CHEM/ORP` | Publish | ORP value (integer) |
| `CHEM/ORP_f` | Publish | ORP (with conversion) |

#### Heater Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `LXi/Status` | Publish | Heater status code |
| `LXi/Error` | Publish | Error code (integer) |
| `LXi/Error_Message` | Publish | Error description string |

#### Freeze Protection Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `Freeze_Protect` | Publish | `"0"` (off) or `"1"` (on) |
| `Freeze_Protect/enabled` | Publish | `"0"` (off) or `"1"` (on) |

#### Chiller Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `Chiller` | Publish | `"0"` (off) or `"2"` (on) |
| `Chiller/enabled` | Publish | `"0"` (off) or `"2"` (on) |

#### Pump Topics (per pump)

Published under `<pump_name>/`:

| Topic Suffix | Payload |
|-------------|---------|
| `/RPM` | Pump RPM (integer) |
| `/GPM` | Pump GPM (integer) |
| `/Watts` | Pump wattage (integer) |
| `/Mode` | Pump operating mode |
| `/Status` | Pump status code |
| `/PPC` | Pump PPC value |
| `/Speed` | Pump speed percentage (0-100) |

#### Light Topics (per light)

| Topic Suffix | Direction | Payload |
|-------------|-----------|---------|
| `/program` | Subscribe | Light program index |
| `/brightness` | Both | Brightness level (0-100) |

#### Other Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `Service_Mode` | Publish | `"0"` (off), `"1"` (on), `"2"` (flash) |
| `Display_Message` | Publish | Panel display message string |
| `Battery` | Publish | `"1"` (ok) or `"0"` (error) |
| `Sensor/<name>` | Publish | Sensor reading value |
| `Alive` | Publish | `"1"` (online) / `"0"` (offline, via LWT) |

### MQTT Command Processing

The daemon subscribes to `<mqtt_aq_topic>/#` and processes messages where the topic ends in `/set` or `/increment`. The topic path (minus the prefix) is routed through the same `action_URI()` function as HTTP and WebSocket requests.

Accepted string values are converted: `"on"` and `"heat"` become `1.0`, `"off"` becomes `0.0`. Numeric values are parsed with `strtof()`.

If `mqtt_convert_temp_to_c` is enabled (default), temperature values received via MQTT are converted from Celsius to Fahrenheit before being applied.

### Home Assistant MQTT Discovery

When `mqtt_discovery_topic` is configured (default: `homeassistant`), the daemon publishes auto-discovery messages for Home Assistant at:

```
<mqtt_discovery_topic>/<entity_type>/<app_id>/<device_id>/config
```

Supported entity types: `climate`, `switch`, `light`, `select`, `sensor`, `binary_sensor`, `humidifier`, `fan`.

### MQTT State Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `MQTT_OFF` | `"0"` | Device off |
| `MQTT_ON` | `"1"` | Device on |
| `MQTT_FLASH` / `MQTT_COOL` | `"2"` | Flashing/cooling state |
| `SWG_ON` | `2` | SWG enabled |
| `SWG_OFF` | `0` | SWG disabled |
