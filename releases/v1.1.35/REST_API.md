# StayTuned REST API Documentation

## Overview

The StayTuned antenna tuner provides a RESTful HTTP API for programmatic control and monitoring. All endpoints return JSON responses and support CORS for browser-based applications.

**Base URL**: `http://staytuned.local` or `http://[device-ip-address]`

> **Response time with `staytuned.local`**: the name is resolved through mDNS. On Windows that lookup can take about 5 s, and API clients such as Bruno, Postman or scripts repeat it for every request, so each call seems slow (the tuner itself answers in about 0.1 s). Use the device's IP address instead: it is shown on the TFT About screen (⚙ → About, `IP:`) and in the serial/telnet log at boot. The IP can change between boots unless the router reserves it for the tuner.

**Protocol**: HTTP (no HTTPS/TLS)

**Authentication**: None (relies on network-level security)

**Content-Type**: `application/json` for all requests and responses

---

## Table of Contents

1. [Status Endpoints](#status-endpoints)
2. [Preset Endpoints](#preset-endpoints)
3. [Motor Control Endpoints](#motor-control-endpoints)
4. [Auto-Tune Endpoints](#auto-tune-endpoints)
5. [System Information](#system-information)
6. [Settings Endpoints](#settings-endpoints)
7. [WebSocket API](#websocket-api)
8. [Error Responses](#error-responses)
9. [Usage Examples](#usage-examples)

---

## Status Endpoints

### Get System Status

Retrieves current real-time status of the antenna tuner including SWR, power, and motor positions.

**Endpoint**: `GET /api/status`

**Request**: None

**Response**:
```json
{
  "swr": 1.5,
  "fwd": 50.0,
  "ref": 2.5,
  "pep": 120.0,
  "mc": 12345,
  "ml": 23456,
  "moving": false,
  "activePreset": 0,
  "rfMeter": true,
  "tuneMode": "auto",
  "autoMode": {
    "active": true,
    "state": "idle",
    "message": "Memory 14.205 MHz",
    "windowLoKHz": 14202,
    "windowHiKHz": 14208
  },
  "freqMHz": 14.205,
  "homed": true,
  "tx": true,
  "alarm": false,
  "alarmSilenced": false
}
```

**Response Fields**:
- `swr` (float): Standing Wave Ratio (1.0 to 10.0+)
- `fwd` (float): Forward power in watts
- `ref` (float): Reflected power in watts
- `pep` (float): Forward peak envelope power (PEP) in watts
- `mc` (integer): Capacitor motor position in steps
- `ml` (integer): Inductor motor position in steps
- `moving` (boolean): `true` if motors are currently moving
- `activePreset` (integer): Index of active preset (-1 if none)
- `tuneMode` (string): `"fine"` or `"auto"` (see [Set Tune Mode](#set-tune-mode))
- `autoMode` (object): auto mode state
  - `active` (boolean): auto mode on and following (the tuner shows its main screen, no tune running)
  - `state` (string): `idle`, `wait_release` (above Tune Max Power, moves when the key is released) or `moving`
  - `message` (string): last auto mode message, e.g. `Memory 14.205 MHz`, `Preset 20m SSB`, `No memory/preset for 14.300 MHz`, `HIGH power - release key`
  - `windowLoKHz` / `windowHiKHz` (integer): frequency range the tuner is set for, 0 / 0 when there is none
- `rfMeter` (boolean): RF meter board present
- `freqMHz` (float): frequency measured by the RF meter, 0 when there is no valid reading (no RF)
- `homed` (boolean): both motors homed (auto-tune needs this)
- `tx` (boolean): RF detected, the same as the "TX" shown on the touchscreen
- `alarm` (boolean): SWR alarm active (SWR above the alarm threshold, Preset mode)
- `alarmSilenced` (boolean): the active alarm has been silenced (touch or [Silence SWR Alarm](#silence-swr-alarm))

**Status Codes**:
- `200 OK`: Success
- `503 Service Unavailable`: Motor control or SWR monitor not initialized

**Example**:
```bash
curl http://staytuned.local/api/status
```

**Update Rate**: Values updated in real-time. Poll at 100ms intervals for smooth monitoring or use WebSocket API for push updates.

---

## Preset Endpoints

### List All Presets

Retrieves all saved tuning presets with names and motor positions.

**Endpoint**: `GET /api/presets`

**Request**: None

**Response**:
```json
{
  "presets": [
    {
      "id": 0,
      "name": "20m Band",
      "mc": 12345,
      "ml": 23456,
      "freqMin": 14000,
      "freqMax": 14350,
      "swr": 1.2,
      "buttons": [5],
      "physicalButtons": []
    },
    {
      "id": 1,
      "name": "40m Band",
      "mc": 15678,
      "ml": 28901,
      "freqMin": 0,
      "freqMax": 0,
      "swr": 0,
      "buttons": [3, 4],
      "physicalButtons": [2]
    }
  ],
  "physicalButtons": true
}
```

**Response Fields**:
- `presets` (array): List of all saved presets
  - `id` (integer): Preset index (0-49)
  - `name` (string): Preset name (max 20 characters)
  - `mc` (integer): Capacitor motor position
  - `ml` (integer): Inductor motor position
  - `freqMin` / `freqMax` (integer): frequency range in kHz, 0 = not set
  - `swr` (float): last SWR measured on the preset, 0 = unknown
  - `buttons` (array): main screen buttons (1-12) that recall the preset
  - `physicalButtons` (array): physical buttons (1-16) that recall the preset
- `physicalButtons` (boolean): the physical button board (MCP23017) is connected

**Status Codes**:
- `200 OK`: Success
- `503 Service Unavailable`: Preset manager not initialized

**Example**:
```bash
curl http://staytuned.local/api/presets
```

---

### Load Preset

Moves motors to a saved preset position.

**Endpoint**: `POST /api/preset/load`

**Request Body**:
```json
{
  "id": 0
}
```

**Request Fields**:
- `id` (integer, required): Preset index to load (0-49)

**Response** (Success):
```json
{
  "message": "Loading preset: 20m Band"
}
```

**Status Codes**:
- `200 OK`: Preset loaded successfully (motors moving)
- `400 Bad Request`: Invalid JSON or missing `id` field
- `403 Forbidden`: Cannot load preset while transmitting (forward power ≥1 W)
- `404 Not Found`: Preset ID does not exist
- `503 Service Unavailable`: Motor control or preset manager not available

**Example**:
```bash
curl -X POST http://staytuned.local/api/preset/load \
  -H "Content-Type: application/json" \
  -d '{"id": 0}'
```

**Behavior**:
- Motors begin moving to preset positions immediately
- Both motors move simultaneously
- Active preset indicator updated on display and web clients
- TX lockout active: Request blocked if forward power ≥1 W

**Motor Movement**:
- Check `/api/status` field `moving` to detect when motors arrive
- Movement time depends on distance and motor speed settings
- Typical time: 1-10 seconds for full range

---

### Save Preset

Saves current motor positions as a new preset or overwrites existing preset.

**Endpoint**: `POST /api/preset/save`

**Request Body**:
```json
{
  "name": "New Preset"
}
```

**Request Fields**:
- `name` (string, required): Preset name (1-20 characters)

**Response** (Success):
```json
{
  "message": "Preset saved successfully"
}
```

**Status Codes**:
- `200 OK`: Preset saved successfully
- `400 Bad Request`: Invalid JSON, missing name, or name length invalid
- `500 Internal Server Error`: Failed to save preset (flash full or write error)
- `503 Service Unavailable`: Motor control or preset manager not available

**Example**:
```bash
curl -X POST http://staytuned.local/api/preset/save \
  -H "Content-Type: application/json" \
  -d '{"name": "40m CW"}'
```

**Behavior**:
- Captures current motor positions at time of request
- If preset with same name exists, overwrites it
- New preset assigned next available ID (0-49)
- Maximum 50 presets supported
- Preset immediately available for loading
- Flash memory updated (persists across reboots)

---

### Update Preset

Changes a preset: name, frequency range, C/L positions and the buttons that recall it. The same changes as the touchscreen preset editor and its Assign dialog.

**Endpoint**: `POST /api/preset/update`

**Request Body** (only `id` is required; keys left out keep their value):
```json
{
  "id": 1,
  "name": "40m SSB",
  "freqMin": 7050,
  "freqMax": 7200,
  "updatePositions": true,
  "buttons": [3],
  "physicalButtons": [2]
}
```

**Request Fields**:
- `id` (integer, required): preset index from `GET /api/presets`
- `name` (string): 1-20 characters, not used by another preset
- `freqMin` / `freqMax` (integer): kHz, 0 = not set; `freqMin` may not be above `freqMax`
- `updatePositions` (boolean): `true` stores the current motor positions (and the current SWR when RF is present) in the preset
- `buttons` (array): the complete list of main screen buttons (1-12) for this preset. A listed button moves from any other preset; a button not listed is freed if it was on this preset
- `physicalButtons` (array): the same for the physical buttons (1-16); only with the button board connected

**Response** (`202 Accepted`):
```json
{
  "message": "Preset updated"
}
```

**Status Codes**:
- `202 Accepted`: checked and accepted; the tuner applies it a moment later (writes flash and redraws the touchscreen)
- `400 Bad Request`: invalid JSON, missing `id`, invalid or duplicate name, `freqMin` above `freqMax`, a button number out of range, or `physicalButtons` without the button board
- `404 Not Found`: no preset with that `id`
- `409 Conflict`: motors moving, auto-tune running, a preset being edited on the touchscreen, or the previous change not applied yet
- `503 Service Unavailable`: preset manager or display not available

**Example**:
```bash
curl -X POST http://staytuned.local/api/preset/update \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "buttons": [3, 4]}'
```

### Delete Preset

**Endpoint**: `POST /api/preset/delete`

**Request Body**:
```json
{
  "id": 1
}
```

**Response** (`202 Accepted`):
```json
{
  "message": "Deleting preset: 40m SSB"
}
```

**Status Codes**:
- `202 Accepted`: accepted; the tuner deletes it a moment later
- `400 Bad Request`: invalid JSON or missing `id`
- `404 Not Found`: no preset with that `id`
- `409 Conflict`: as for Update Preset
- `503 Service Unavailable`: preset manager or display not available

**Behavior**:
- The presets after it move down one index (`id`); re-read `GET /api/presets`
- Its buttons become free, and the buttons of the following presets keep pointing to the same preset
- If it was the active preset, no preset is active afterwards

---

## Motor Control Endpoints

**Note**: The TFT touchscreen interface includes a **Continuous Mode** feature (hold motor button for 400ms to run continuously with variable speed control). This is a hardware UI feature and not available via REST API. The API uses discrete step-based movement as documented below. For continuous operation via API, send repeated `/api/motor` commands with your desired timing and step size.

---

### Move Motor

Moves a motor by relative number of steps from current position.

**Endpoint**: `POST /api/motor`

**Request Body**:
```json
{
  "motor": "MC",
  "steps": 100
}
```

**Request Fields**:
- `motor` (string, required): Motor identifier
  - `"MC"` = Capacitor motor
  - `"ML"` = Inductor motor
- `steps` (integer, required): Number of steps to move
  - Positive = increase position (move forward)
  - Negative = decrease position (move backward)
  - Range: typically -50000 to +50000

**Response** (Success):
```json
{
  "message": "Motor command sent"
}
```

**Status Codes**:
- `200 OK`: Motor command accepted (motor moving)
- `400 Bad Request`: Invalid JSON, missing fields, or invalid motor ID
- `409 Conflict`: Auto-tune running, or the previous move command has not been applied yet (send again)
- `503 Service Unavailable`: Motor control not available

**Example**:
```bash
# Move capacitor motor forward 1000 steps
curl -X POST http://staytuned.local/api/motor \
  -H "Content-Type: application/json" \
  -d '{"motor": "MC", "steps": 1000}'

# Move inductor motor backward 500 steps
curl -X POST http://staytuned.local/api/motor \
  -H "Content-Type: application/json" \
  -d '{"motor": "ML", "steps": -500}'
```

**Behavior**:
- Movement is relative to current position
- Motor accelerates/decelerates smoothly (AccelStepper)
- Non-blocking: Command returns immediately; the tuner's main loop starts the move
- A command sent while the motor still moves retargets it from its current position
- The target is kept within 0 and the travel limit (hardware calibration)
- Does not update active preset indicator

**Safety**:
- No TX lockout (movement allowed during transmission)
- Monitor position via `/api/status`

---

### Move To Position

Moves one or both motors to an absolute position.

**Endpoint**: `POST /api/motor/goto`

**Request Body** (`mc`, `ml` or both):
```json
{
  "mc": 1200,
  "ml": 45000
}
```

**Response** (Success):
```json
{
  "message": "Moving"
}
```

**Status Codes**:
- `200 OK`: Move started
- `400 Bad Request`: Invalid JSON, neither `mc` nor `ml` given, or a position outside 0 and the travel limit (the message gives the limits; `GET /api/about` lists them too)
- `403 Forbidden`: Transmitting (as for a preset load)
- `409 Conflict`: Auto-tune running, or the motors are still moving
- `503 Service Unavailable`: Motor control not available

**Example**:
```bash
curl -X POST http://staytuned.local/api/motor/goto \
  -H "Content-Type: application/json" \
  -d '{"mc": 1200}'
```

### Stop Motors

Stops a manual move, a move to a position or a preset load. The motors decelerate to a stop; the position stays known.

**Endpoint**: `POST /api/motor/stop`

**Request**: None

**Response** (Success):
```json
{
  "message": "Motors stopping"
}
```

**Status Codes**:
- `200 OK`: Stop requested (also when nothing moves)
- `409 Conflict`: Auto-tune running: use `POST /api/autotune/stop`
- `503 Service Unavailable`: Motor control not available

**Note**: homing is stopped with the ABORT button on the touchscreen, not with this request.

---

### Home Motors

Moves both motors to their zero positions using limit switches.

**Endpoint**: `POST /api/home`

**Request**: None (empty body or no body)

**Response** (Success):
```json
{
  "message": "Homing started"
}
```

**Status Codes**:
- `200 OK`: Homing sequence initiated
- `409 Conflict`: Auto-tune running
- `503 Service Unavailable`: Motor control not available

**Example**:
```bash
curl -X POST http://staytuned.local/api/home
```

**Behavior**:
- The request returns at once; the tuner then runs the same homing as at boot (the TFT shows the progress with an ABORT button, and the failure screen with Reboot if a switch isn't found)
- The motors home one after the other: capacitor first, then inductor
- The tuner's main loop is busy while homing runs, so the web page and WebSocket updates pause until it is done
- Intelligent homing:
  - If already at limit, backs off 300 steps first
  - Moves forward slowly until limit switch activates
  - Applies homing offset (default: 100 steps forward)
- Homing sequence takes 10-30 seconds depending on positions
- Motors marked as "homed" after successful completion
- Check `/api/status` field `moving` to detect completion

**Important**:
- All motor positions reset to zero after homing
- Any active preset cleared
- Homing required after power cycle or if motors skip steps

---

## Auto-Tune Endpoints

Auto-tune searches for the lowest SWR automatically (see [AUTO_TUNE_REQUIREMENTS.md](AUTO_TUNE_REQUIREMENTS.md)). Transmit a low-power carrier while it runs. While a tune is active, `POST /api/motor`, `POST /api/home` and `POST /api/preset/load` return **409** `{"error":"Auto-tune running"}`.

The tuning parameters and the frequency memory used by fine tune (see [FREQ_MEMORY_REQUIREMENTS.md](FREQ_MEMORY_REQUIREMENTS.md)) are set in the `autoTune` object of [`/api/settings`](#settings-endpoints). Memory messages (`Memory 14.203 MHz - 987 free` when a fine tune ends on a memory hit, `Memory entry needs re-tune`, `Saved to memory (12)`, ...) appear in `phaseDetail`.

### Start Auto-Tune

**Endpoint**: `POST /api/autotune/start`

Starts a tune from the current position using the stored settings. The touch screen switches to the auto-tune screen.

**Response** (200): `{"message": "Auto-tune started"}`

**Errors** (400): motors moving, motors not homed, RF meter not connected, or a tune already running.

```bash
curl -X POST http://staytuned.local/api/autotune/start
```

### Start Fine Tune

**Endpoint**: `POST /api/finetune/start`

Starts a fine tune from the active preset, the same as tapping FINE TUNE on the touch screen. It first checks the frequency memory: on a memory hit the motors move straight to the stored point; otherwise it refines from the current position while you key a carrier. The fine tune ends when you unkey. Works in fine tune and auto mode.

**Response** (200): `{"message": "Fine tune started"}`

**Errors**: 400 motors not homed, RF meter not connected, or start failed; 409 a tune already running, no preset selected (`Select a preset first`), or the preset still moving; 503 auto-tuner unavailable.

```bash
curl -X POST http://staytuned.local/api/finetune/start
```

### Stop Auto-Tune

**Endpoint**: `POST /api/autotune/stop`

Aborts a running tune. The motors stop where they are.

**Response** (200): `{"message": "Auto-tune aborted"}`

### Get Auto-Tune Status

**Endpoint**: `GET /api/autotune/status`

**Response** (200):
```json
{
  "state": "searching",
  "status": "Searching for dip",
  "currentSWR": 4.3,
  "bestSWR": 3.1,
  "targetSWR": 1.5,
  "bestCapacitor": 600,
  "bestInductor": 21400,
  "currentIteration": 2,
  "totalIterations": 5,
  "progressPercent": 40,
  "elapsedTime": 48.0,
  "phase": "dip_search",
  "phaseDetail": "Dip search C=75% (2/5), L sweep 42%",
  "dipFound": false,
  "capStep": 0,
  "indStep": 0
}
```

| Field | Meaning |
|-------|---------|
| `state` | `idle`, `initializing`, `waiting_for_power`, `searching` (dip search), `measuring`, `refining`, `converged`, `failed`, `aborted`, `error` |
| `status` | Human-readable status or result message (e.g. `"No dip found"`) |
| `phase` | `dip_search`, `refine`, `finish`, or `""` when not running |
| `phaseDetail` | What the tuner is doing right now |
| `dipFound` | `true` once the dip search has succeeded (or was skipped because the start SWR was already low) |
| `capStep` / `indStep` | Current refinement step sizes in steps (0 outside refinement) |
| `currentIteration` / `totalIterations` | Dip-search attempt of 5, then refinement evaluations of the maximum |

The WebSocket status message carries the same information in an `autoTune` object while a tune is not idle.

```bash
curl http://staytuned.local/api/autotune/status
```

### Set Tune Mode

**Endpoint**: `POST /api/tunemode`

Switches the FINE TUNE button between fine tune mode and **auto mode**, the same as holding the button on the touch screen. In auto mode the tuner follows the transmit frequency by itself: on each new stable frequency it moves to the memory entry for it, or else to the preset whose frequency range covers it. It only follows while the touch screen shows its main screen (see [FREQ_MEMORY_REQUIREMENTS.md](FREQ_MEMORY_REQUIREMENTS.md) section 3.5). The mode is kept after a reboot. `POST /api/finetune/start` works in both modes.

**Request Body**:
```json
{"mode": "auto"}
```
`mode` is `"auto"` or `"fine"`.

**Response** (200): `{"mode": "auto"}`. The change is applied by the main loop a moment later; `tuneMode` in the status shows it.

**Errors**: 400 `{"error":"mode must be auto or fine"}`, 409 `{"error":"RF meter not connected"}` (like the TFT button, the mode can only be switched with the RF meter board present), 503 when auto mode isn't available.

```bash
curl -X POST -d '{"mode":"auto"}' http://staytuned.local/api/tunemode
```

---

## System Information

### Get About Information

Retrieves firmware version, compile time, and connected WebSocket clients.

**Endpoint**: `GET /api/about`

**Request**: None

**Response**:
```json
{
  "version": "0.1.0",
  "compiled": "Feb 17 2026 14:35:42",
  "hostname": "staytuned",
  "ip": "192.168.1.95",
  "rssi": -58,
  "uptime": 86523,
  "rfMeter": {
    "present": true,
    "deviceId": 161,
    "firmware": 36
  },
  "physicalButtons": false,
  "motors": {
    "homed": true,
    "mcEnd": 1200,
    "mlEnd": 120000
  },
  "clients": [
    {
      "id": 1,
      "ip": "192.168.1.100"
    },
    {
      "id": 2,
      "ip": "192.168.1.101"
    }
  ]
}
```

**Response Fields**:
- `version` (string): Firmware version
- `compiled` (string): Build date and time
- `hostname` (string): Network name; the tuner answers at `<hostname>.local`
- `ip` (string): IP address; `rssi` (integer): WiFi signal in dBm; `uptime` (integer): seconds since boot
- `rfMeter` (object): `present`, and the board's `deviceId` and `firmware` version bytes (0 when absent), read at boot
- `physicalButtons` (boolean): physical button board (MCP23017) connected
- `motors` (object): `homed`, and the travel limits `mcEnd` / `mlEnd` in steps (positions run from 0 to these)
- `clients` (array): Connected WebSocket clients
  - `id` (integer): WebSocket client ID
  - `ip` (string): Client IP address

**Status Codes**:
- `200 OK`: Success (always)

**Example**:
```bash
curl http://staytuned.local/api/about
```

**Use Cases**:
- Check installed firmware version
- Verify client connections
- Troubleshoot connectivity issues
- Monitor active users

### Silence SWR Alarm

Silences the alarm tone, like tapping the alarm popup on the touchscreen. The popup stays until the SWR is below the threshold again.

**Endpoint**: `POST /api/alarm/silence`

**Request**: None

**Status Codes**:
- `200 OK`: `{"message":"Alarm silenced"}`
- `409 Conflict`: No SWR alarm active
- `503 Service Unavailable`: Display not available

`alarm` and `alarmSilenced` in `GET /api/status` and the WebSocket status show the alarm state.

### Pictures

**Endpoints**: `GET /img/splash.jpg` (320x240 boot picture), `GET /img/heading.jpg` (320x25 main screen heading)

JPEG images used by the web pages (banner and background). They are the same pictures the touchscreen shows, served straight from flash with `Cache-Control: max-age=86400`.

---

## Settings Endpoints

The same settings as the touchscreen Settings page and the web Settings page. The JSON format is shared with the SD card settings file (`settings.json`), except for `network`, which only the API carries.

### Get Settings

**Endpoint**: `GET /api/settings`

**Request**: None

**Response** (example):
```json
{
  "version": 1,
  "display": {
    "brightness": 120,
    "screenTimeout": 0,
    "swrGood": 1.5,
    "swrCaution": 2
  },
  "motor": {
    "mcMaxSpeed": 2200,
    "mlMaxSpeed": 6000,
    "mcAccel": 800,
    "mlAccel": 1900,
    "mcHomingSpeed": 100,
    "mlHomingSpeed": 2000,
    "mCHomingOffset": 0,
    "mLHomingOffset": -1250,
    "mCHomingInv": true,
    "mLHomingInv": true,
    "autoDisable": true,
    "mcBacklash": 6,
    "mlBacklash": 0
  },
  "autoTune": {
    "targetSWR": 1.5,
    "indSweepSpeed": 2500,
    "maxTime": 600,
    "minPower": 1,
    "maxTunePower": 15,
    "settleDelay": 300,
    "fmEnabled": true,
    "fmWindowKHz": 3,
    "fmStoreMaxSWR": 2
  },
  "swr": {
    "alarmThreshold": 2.8
  },
  "network": {
    "hostname": "staytuned"
  }
}
```

`motor` is left out if motor control is unavailable, `autoTune` without the auto-tuner and `swr` without the SWR monitor.

**Fields** (out-of-range values sent with `POST` are clamped to the range):

| Key | Type | Range | Meaning |
|-----|------|-------|---------|
| `version` | integer | 1 | Format version (read only) |
| **`display`** | | | |
| `brightness` | integer | 0-255 | Backlight brightness |
| `screenTimeout` | integer | seconds | Backlight off after this long without a touch; 0 = always on |
| `swrGood` | number | 1.0-3.0 | SWR shown green up to this value |
| `swrCaution` | number | 1.0-5.0 | SWR shown yellow up to this value, red above |
| **`motor`** | | | `mc…` / `mC…` = capacitor motor, `ml…` / `mL…` = inductor motor |
| `mcMaxSpeed`, `mlMaxSpeed` | number | 100-10000 | Maximum speed, steps/s |
| `mcAccel`, `mlAccel` | number | 100-5000 | Acceleration, steps/s² |
| `mcHomingSpeed`, `mlHomingSpeed` | number | 100-2000 | Homing speed, steps/s |
| `mCHomingOffset`, `mLHomingOffset` | integer | -5000 to 5000 | Steps from the limit switch to position 0 after homing |
| `mCHomingInv`, `mLHomingInv` | boolean | | `true` = the limit switch is at the minimum end of the travel ("Inverted" on the TFT) |
| `autoDisable` | boolean | | Disable the motor drivers when idle |
| `mcBacklash`, `mlBacklash` | integer | 0-1000 | Extra steps on a direction reversal (backlash compensation) |
| **`autoTune`** | | | |
| `targetSWR` | number | 1.1-3.0 | A tune stops when it reaches this SWR |
| `indSweepSpeed` | number | 100-4000 | Inductor sweep speed during the dip search, steps/s |
| `maxTime` | number | 60-1200 | Maximum tune time, seconds |
| `minPower` | number | 0.1-10 | Minimum forward power for tuning, watts |
| `maxTunePower` | number | 1-100 | Maximum forward power for tuning, watts (above it the tune waits) |
| `settleDelay` | integer | 100-2000 | Wait after a move before measuring, ms |
| `fmEnabled` | boolean | | Frequency memory on/off (default `true`) |
| `fmWindowKHz` | integer | 1-20 | Memory match window, kHz (default 3) |
| `fmStoreMaxSWR` | number | 1.0-3.0 | Only tunes with an SWR up to this are stored in the memory (default 1.5) |
| **`swr`** | | | |
| `alarmThreshold` | number | 1.0-10.0 | SWR alarm popup and beep above this value |
| **`network`** | | | |
| `hostname` | string | 1-31 characters | Network name, see below |

The motor end positions (travel limits) are not part of the API: they are set by hardware calibration on the touchscreen.

**Status Codes**:
- `200 OK`: Success
- `500 Internal Server Error`: Settings could not be read
- `503 Service Unavailable`: Display UI not available

**Example**:
```bash
curl http://staytuned.local/api/settings
```

### Save Settings

Changes one or more settings and saves them in flash.

**Endpoint**: `POST /api/settings`

**Request Body**: any subset of the `GET /api/settings` object. Only the keys you send are changed, e.g.:
```json
{
  "display": { "brightness": 200 },
  "autoTune": { "targetSWR": 1.3 }
}
```

**Response** (Success, `202 Accepted`):
```json
{
  "message": "Settings saved"
}
```

**Status Codes**:
- `202 Accepted`: Settings accepted; they are applied and saved by the tuner's main loop a moment later
- `400 Bad Request`: Invalid JSON, or an invalid network name
- `409 Conflict`: Motors moving, a preset load in progress, auto-tune running, or the previous save not applied yet; try again
- `413 Payload Too Large`: Body larger than one network packet (about 1.4 KB); send fewer keys
- `503 Service Unavailable`: Display UI not available

**Example**:
```bash
curl -X POST http://staytuned.local/api/settings \
  -H "Content-Type: application/json" \
  -d '{"swr":{"alarmThreshold":2.5}}'
```

**Behavior**:
- Saving writes flash, which briefly stalls the tuner, so it is refused while the motors move or a tune runs
- The touchscreen Settings page shows the new values at once if it is open
- Keys with the wrong type are ignored (for example a string where a number is expected)

**Network name**: `{"network":{"hostname":"tuner-2"}}` sets the tuner's network name: 1-31 characters `a-z 0-9 -`, not starting or ending with `-` (uppercase is accepted and stored lowercase; anything else returns 400). The tuner then answers at `<name>.local` at once, and the response message says so; OTA and the DHCP hostname follow at the next reboot. The name is not included in the SD card settings file.

---

## WebSocket API

For real-time bidirectional communication, use the WebSocket endpoint instead of polling REST endpoints.

**Endpoint**: `ws://staytuned.local/ws`

**Protocol**: WebSocket (RFC 6455)

**Update Rate**: 100ms (10 Hz)

### Connection

```javascript
const ws = new WebSocket('ws://staytuned.local/ws');

ws.onopen = function() {
  console.log('WebSocket connected');
};

ws.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('Status update:', data);
};

ws.onerror = function(error) {
  console.error('WebSocket error:', error);
};

ws.onclose = function() {
  console.log('WebSocket disconnected');
};
```

### Message Format

**Status Updates** (Server → Client):
```json
{
  "type": "status",
  "swr": 1.5,
  "fwd": 50.0,
  "ref": 2.5,
  "pep": 120.0,
  "mc": 12345,
  "ml": 23456,
  "moving": false,
  "activePreset": 0,
  "presetMoving": false,
  "rfMeter": true,
  "freqMHz": 14.205,
  "homed": true,
  "tx": true,
  "alarm": false,
  "alarmSilenced": false,
  "autoTune": {
    "state": "refining",
    "status": "Refining",
    "bestSWR": 1.32,
    "targetSWR": 1.5,
    "current": 14,
    "total": 80,
    "percent": 17,
    "elapsed": 72.5,
    "phase": "refine",
    "phaseDetail": "Refine dC=8 dL=120 SWR 1.32",
    "dipFound": true,
    "capStep": 8,
    "indStep": 120
  }
}
```

`autoTune` is present only when the auto-tuner is not idle. `tuneMode` and `autoMode` are always present, as in `GET /api/status`.

**Ping/Pong** (Client ↔ Server):
```json
// Client sends:
{"type": "ping"}

// Server responds:
{"type": "pong"}
```

### Behavior

- **Automatic Reconnection**: Client should implement reconnection logic
- **Multiple Clients**: Up to 4 simultaneous connections supported
- **Synchronized Updates**: All clients receive same status updates
- **Connection Logging**: Connections logged to SD card if available and time synced

### Advantages Over REST Polling

| Feature | WebSocket | REST Polling |
|---------|-----------|--------------|
| Update latency | 100ms guaranteed | Depends on poll rate |
| Network efficiency | High (push-based) | Low (repeated requests) |
| Server load | Minimal | High with fast polling |
| Client complexity | Moderate (reconnection) | Simple (just fetch) |

**Recommendation**: Use WebSocket for web interfaces, REST for scripts and integrations.

---

## Error Responses

All error responses follow a consistent JSON format:

```json
{
  "error": "Description of error"
}
```

### Common Error Codes

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 400 | Bad Request | Invalid JSON, missing fields, out-of-range values |
| 403 | Forbidden | TX lockout active (cannot load preset while transmitting) |
| 404 | Not Found | Endpoint or preset ID does not exist |
| 409 | Conflict | Auto-tune running, motors moving, or a previous change not applied yet |
| 500 | Internal Server Error | Failed to save preset, flash write error |
| 503 | Service Unavailable | Component not initialized (motor/SWR/preset manager) |

### Error Examples

**Invalid JSON**:
```json
{
  "error": "Invalid JSON"
}
```

**Preset Not Found**:
```json
{
  "error": "Preset not found"
}
```

**TX Lockout**:
```json
{
  "error": "Cannot change preset while transmitting"
}
```

**Service Unavailable**:
```json
{
  "error": "Motor control unavailable"
}
```

---

## Usage Examples

### Check Status and SWR

```python
import requests

response = requests.get('http://staytuned.local/api/status')
data = response.json()

print(f"SWR: {data['swr']}:1")
print(f"Forward: {data['fwd']}W")
print(f"Reflected: {data['ref']}W")
print(f"Forward PEP: {data['pep']}W")

if data['swr'] > 2.0:
    print("WARNING: High SWR!")
```

### Load Preset by Name

```python
import requests

# Get all presets
response = requests.get('http://staytuned.local/api/presets')
presets = response.json()['presets']

# Find preset by name
target_name = "40m Band"
preset_id = next((p['id'] for p in presets if p['name'] == target_name), None)

if preset_id is not None:
    # Load preset
    requests.post('http://staytuned.local/api/preset/load',
                  json={'id': preset_id})
    print(f"Loading preset: {target_name}")
else:
    print(f"Preset '{target_name}' not found")
```

### Tune for Minimum SWR

```python
import requests
import time

def get_swr():
    response = requests.get('http://staytuned.local/api/status')
    return response.json()['swr']

def move_motor(motor, steps):
    requests.post('http://staytuned.local/api/motor',
                  json={'motor': motor, 'steps': steps})

# Simple hill-climbing algorithm
best_swr = get_swr()
step_size = 100

for i in range(20):  # Max 20 iterations
    # Try moving capacitor forward
    move_motor('MC', step_size)
    time.sleep(1)  # Wait for movement and SWR to settle
    
    new_swr = get_swr()
    if new_swr < best_swr:
        best_swr = new_swr
        print(f"Improved SWR: {best_swr:.2f}")
    else:
        # Move back and try other direction
        move_motor('MC', -step_size * 2)
        time.sleep(1)
    
    if best_swr < 1.5:
        print("Good SWR achieved!")
        break
```

### Monitor WebSocket in Python

```python
import websocket
import json

def on_message(ws, message):
    data = json.loads(message)
    if data['type'] == 'status':
        print(f"SWR: {data['swr']:.2f}, "
              f"Fwd: {data['fwd']:.1f}W, "
              f"MC: {data['mc']}, "
              f"ML: {data['ml']}")

def on_error(ws, error):
    print(f"Error: {error}")

def on_close(ws, close_status_code, close_msg):
    print("WebSocket closed")

def on_open(ws):
    print("WebSocket connected")

ws = websocket.WebSocketApp(
    "ws://staytuned.local/ws",
    on_open=on_open,
    on_message=on_message,
    on_error=on_error,
    on_close=on_close
)

ws.run_forever()
```

### Save Current Position as Preset

```bash
#!/bin/bash

# Save current position
curl -X POST http://staytuned.local/api/preset/save \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"${1:-AutoSaved}\"}"

# List all presets to verify
curl http://staytuned.local/api/presets | jq
```

Usage: `./save_preset.sh "My Preset Name"`

---

## Best Practices

### Rate Limiting

While no rate limiting is enforced, be respectful of device resources:

- **REST Polling**: Max 10 requests/second
- **Motor Commands**: Max 5 commands/second
- **WebSocket**: Use push updates instead of polling

### Error Handling

Always check HTTP status codes and handle errors gracefully:

```python
try:
    response = requests.post('http://staytuned.local/api/preset/load',
                             json={'id': 0}, timeout=5)
    response.raise_for_status()  # Raises exception for 4xx/5xx
    print(response.json()['message'])
except requests.exceptions.RequestException as e:
    print(f"Error: {e}")
```

### TX Lockout

Check forward power before loading presets:

```python
status = requests.get('http://staytuned.local/api/status').json()
if status['fwd'] >= 1.0:   # Same threshold as the tuner (POWER_LOCKOUT_THRESHOLD)
    print("Cannot load preset while transmitting")
else:
    requests.post('http://staytuned.local/api/preset/load', json={'id': 0})
```

### Wait for Movement Completion

Poll `/api/status` until `moving` becomes `false`:

```python
def wait_for_motors():
    while True:
        status = requests.get('http://staytuned.local/api/status').json()
        if not status['moving']:
            break
        time.sleep(0.1)

# Load preset and wait
requests.post('http://staytuned.local/api/preset/load', json={'id': 0})
wait_for_motors()
print("Motors arrived at preset position")
```

---

## Security Considerations

**No Authentication**: The API has no authentication mechanism. Security relies on:

1. **Network Isolation**: Device should be on trusted network only
2. **Firewall**: Block external access to port 80
3. **Physical Security**: Unauthorized access requires physical proximity

**Recommendations**:
- Do not expose device to internet directly
- Use VPN for remote access
- Consider network segmentation
- Monitor `/api/about` for unexpected clients

**Future Enhancements**:
- API key authentication
- HTTPS/TLS support
- Role-based access control
- Rate limiting per client

---

## Changelog

### October 2026
- `POST /api/tunemode` (fine tune / auto mode); 409 without the RF meter board
- `GET /api/status` and WebSocket status: `tuneMode` and `autoMode` (state, message, window)
- `POST /api/finetune/start`; frequency memory settings (`fmEnabled`, `fmWindowKHz`, `fmStoreMaxSWR`) in `/api/settings`
- Web pages served as UTF-8; web preset load handled on the main task (no more reboot)
- Network name: `network.hostname` in `GET`/`POST /api/settings` (1-31 characters a-z 0-9 `-`; 400 if not valid), `hostname` in `/api/about`
- `GET /img/splash.jpg`, `GET /img/heading.jpg` (pictures for the web pages)
- `GET`/`POST /api/settings` documented in full (all keys and ranges); the `POST /api/stop` section, removed from the firmware in April 2026, is gone from this document
- `POST /api/home` returns at once and homes on the main loop (no more reboot)
- New: `POST /api/preset/update`, `POST /api/preset/delete`, `POST /api/motor/goto`, `POST /api/motor/stop`, `POST /api/alarm/silence`
- `GET /api/presets`: `freqMin`/`freqMax`/`swr` documented, `buttons` and `physicalButtons` added
- `GET /api/status` and WebSocket: `freqMHz`, `homed`, `tx`, `alarm`, `alarmSilenced` (and `rfMeter` in `/api/status`)
- `GET /api/about`: `ip`, `rssi`, `uptime`, `rfMeter`, `physicalButtons`, `motors`
- `POST /api/motor` is applied by the main loop (409 if the previous command is still pending)

### Version 0.1.0 (February 2026)
- Initial REST API implementation
- WebSocket real-time updates
- All preset and motor control endpoints
- System information endpoint
- Error handling and status codes

---

**Last Updated**: October 17, 2026  
**API Version**: 1.0  
**Firmware Version**: 1.1.27

For additional support, see [USER_GUIDE.md](USER_GUIDE.md) for web interface documentation.
