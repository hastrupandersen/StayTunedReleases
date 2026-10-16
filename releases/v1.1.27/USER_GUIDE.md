# StayTuned - User Guide

## Table of Contents
1. [Overview](#overview)
2. [Getting Started](#getting-started)
3. [Main Screen](#main-screen)
4. [Operating Modes](#operating-modes)
5. [Button Reference](#button-reference)
6. [Menu System](#menu-system)
7. [Preset Management](#preset-management)
8. [Settings & Configuration](#settings--configuration)
9. [Calibration](#calibration)
10. [Auto-Tune](#auto-tune)
11. [SD Card Operations](#sd-card-operations)
12. [Firmware Update from SD Card](#firmware-update-from-sd-card)
13. [About Screen](#about-screen)
14. [SWR Alarms](#swr-alarms)
15. [WiFi & OTA Updates](#wifi--ota-updates)
16. [Web Interface](#web-interface)

---

## Overview

**StayTuned** is an ESP32-based automatic antenna tuner with a 2.8" touchscreen display (320×240 pixels). It features dual stepper motor control for capacitor and inductor adjustment, real-time SWR monitoring, and comprehensive preset management.

### Key Features
- **Dual Stepper Motor Control**: Independent control of capacitor and inductor positions
- **Two Operating Modes**: Preset Mode (quick recall) and Manual Mode (fine tuning)
- **Real-time SWR Monitoring**: Forward/reflected power + forward PEP display with color-coded alerts
- **Preset System**: Save up to 50 tuning positions with 12 quick-access touch buttons and up to 16 configurable physical hardware buttons
- **Visual & Audio Alarms**: High SWR warnings with user acknowledgment
- **Dual Storage**: Internal flash + optional SD card for backup/restore
- **Physical Button Support**: Optional MCP23017 port expander supports up to 16 configurable hardware buttons, each assignable to any preset
- **WiFi Support**: OTA firmware updates and remote serial debugging
- **SD Card Firmware Updates**: Browse and flash tuner firmware (`.bin`) or RF meter PIC firmware (`.hex`) from a scrollable, version-sorted list on the About screen — no USB or WiFi required

---

## Getting Started

### Initial Power-Up Sequence

1. **Startup Message**: Brief "StayTuned v1.0.0" splash screen

2. **Touch Calibration** (first boot only):
   - Follow on-screen prompts to tap calibration targets
   - Calibration data is saved and reused on subsequent boots

3. **WiFi Configuration** (if enabled):
   - If no saved WiFi credentials, device creates access point: **"StayTuned"**
   - Password: **"staytuned2026"**
   - Connect and configure WiFi via captive portal
   - On successful connection, OTA updates become available via `pio run -e esp32-ota --target upload` (device reachable as `staytuned.local`)

4. **Automatic Homing**:
   - Motors automatically move to limit switches using intelligent homing sequence
   - Status displayed: "Waiting..." → "Homing..." → "Complete" or "FAILED"
   - Both motors must home successfully before system is operational
   - **Improved Homing Logic**:
     - If limit switch already active at startup, motor backs off 300 steps first
     - Then moves forward until limit switch activates
     - Stops immediately when switch activates
     - Applies configurable homing offset (default: 100 steps)
   - If homing fails, check:
     - Limit switch wiring (GPIO 32 for Motor 1, GPIO 33 for Motor 2)
     - Switches must deactivate after backing off (not stuck/shorted)
     - Motor power supply (12-24V)
     - No mechanical obstructions

5. **Ready to Use**: System displays Main Screen

---

## Main Screen

The Main Screen is the primary interface and has two distinct layouts depending on the operating mode.

### Screen Layout - Preset Mode

```
┌─────────────────────────────────────────────────┐
│ StayTuned                           [WiFi] [⚙]  │  ← Header
├─────────────────────────────────────────────────┤
│                              │ Power            │
│ SWR: [████████░░] 1.5:1      │ Fwd: 50.0W      │  ← SWR & Power
│ Color: Green/Yellow/Red      │ Ref: 2.5W       │     Display
├──────────────────────────────┴──────────────────┤
│  ┌────────┬────────┬────────┬────────┐         │
│  │   1    │   2    │   3    │   4    │         │  ← Quick Access
│  │ Band A │ Band B │        │        │         │     Buttons
│  │───────│───────│───────│───────│         │     (12 total)
│  ├────────┼────────┼────────┼────────┤         │
│  │   5    │   6    │   7    │   8    │         │     Row 1-3:
│  │ 20m    │ 40m    │        │        │         │     Preset quick
│  │───────│───────│───────│───────│         │     access
│  ├────────┼────────┼────────┼────────┤         │
│  │   9    │   10   │   11   │   12   │         │
│  │ 80m    │        │        │        │         │
│  │───────│───────│───────│───────│         │
│  └────────┴────────┴────────┴────────┘         │
└─────────────────────────────────────────────────┘

Legend:
• Assigned buttons show preset name (e.g., "Band A", "20m")
• Unassigned buttons show number only (e.g., "1", "2")
• Active preset shows status bar at bottom:
  - Blue bar = motors moving to position
  - Red bar = motors at position
  - Black bar = inactive
```

### Screen Layout - Manual Mode

```
┌─────────────────────────────────────────────────┐
│ StayTuned                           [WiFi] [⚙]  │  ← Header
├─────────────────────────────────────────────────┤
│                              │ Power            │
│ SWR: [████████░░] 1.5:1      │ Fwd: 50.0W      │  ← SWR & Power
│ Color: Green/Yellow/Red      │ Ref: 2.5W       │     Display
├──────────────────────────────┴──────────────────┤
│                                                  │
│  Cap:  [ - ]  [ + ]           12345             │  ← Capacitor
│                                                  │     Control
│  Ind:  [ - ]  [ + ]           23456             │  ← Inductor
│                                                  │     Control
│                                                  │
│  [    AUTO-TUNE    ]  [     STOP      ]         │  ← System
│                                                  │     Controls
│  [  1  ] [ 10  ] [ 100 ] [ 1000 ]               │  ← Step Size
│                                                  │     Selector
└─────────────────────────────────────────────────┘

Legend:
• Minus (-) decreases position, Plus (+) increases
• Position values shown on right side (real-time)
• Step size buttons (1/10/100/1000) - active one highlighted
• AUTO-TUNE: Launches automatic tuning (see Auto-Tune section)
• Stop: Emergency stop (halts motor movement, also stops continuous mode)
• There is no manual "Home" button here — homing runs automatically at
  boot; to re-home, power-cycle the device (or use the `h` serial command)
```

### Header Elements

- **StayTuned**: Application title
- **WiFi Icon**: Shows connection status (if WiFi enabled)
- **Gear Icon (⚙)**: Opens settings menu

### SWR Display

- **Bar Graph**: Visual representation of SWR level
- **Numeric Value**: Displays as ratio (e.g., "1.5:1")
- **Color Coding**:
  - 🟢 **Green**: SWR ≤ 1.5 (Good)
  - 🟡 **Yellow**: 1.5 < SWR ≤ 2.0 (Caution)
  - 🔴 **Red**: SWR > 2.0 (High - Alarm triggered)

### Power Display

- **Forward Power (Fwd)**: Transmitted power in watts
- **Reflected Power (Ref)**: Reflected power in watts
- **Forward PEP**: Peak envelope power in watts
- Values update approximately every 50ms

---

## Operating Modes

StayTuned has two distinct operating modes that change the Main Screen layout:

### Preset Mode (Default)

**Purpose**: Quick recall of saved antenna tuning positions

**Features**:
- 12 quick-access buttons in 3×4 grid
- Each button can be assigned to a saved preset
- One-touch preset loading
- SWR alarms are **ACTIVE** in this mode
- Status indicators show active preset and motor status
- If a MCP23017 port expander is fitted, up to 16 physical hardware buttons can also activate presets independently of the touchscreen mode

**Use Case**: Operating with known frequencies/bands where presets have been saved

**Switching**: Tap ⚙ → **"Manual Mode"** → Switches to Manual Mode

### Manual Mode

**Purpose**: Fine-tuning antenna positions and saving new presets

**Features**:
- Independent +/- control for capacitor and inductor
- Variable step size (1, 10, 100, 1000 steps)
- **Continuous mode**: Hold any motor button to run continuously
- **Variable speed control**: Speed scales with step size selection
- **Real-time speed adjustment**: Change speed while motor running
- Real-time position display
- AUTO-TUNE and Stop buttons (no manual Home button — homing runs automatically at boot)
- SWR alarms are **DISABLED** in this mode (high SWR expected during tuning)

**Use Case**: 
- Tuning for new frequencies
- Fine-tuning existing presets
- Exploring optimal positions

**Switching**: Tap ⚙ → **"Preset Mode"** → Switches to Preset Mode

---

### Continuous Mode (Manual Mode Feature)

**Purpose**: Run motors continuously for fast positioning without repeated button taps

**Activation**:
1. Select desired step size (1, 10, 100, or 1000)
2. **Press and hold** any motor button (Cap+, Cap-, Ind+, or Ind-)
3. After **400ms**, motor enters continuous mode and runs at controlled speed
4. Motor continues until:
   - Button is released
   - Stop button is pressed
   - Motor reaches limit

**Speed Control**:
Motor speed is automatically scaled based on selected step size:
- **Step 1**: 25% of max speed (500 steps/sec) - Very precise, slow movement
- **Step 10**: 50% of max speed (1000 steps/sec) - Medium speed for fine adjustments
- **Step 100**: 75% of max speed (1500 steps/sec) - Fast positioning
- **Step 1000**: 100% of max speed (2000 steps/sec) - Maximum speed for coarse movements

**Real-Time Speed Adjustment**:
While motor is running in continuous mode:
1. Simply **tap any step size button** (1, 10, 100, 1000)
2. Motor speed smoothly transitions to the new speed
3. AccelStepper's acceleration profile ensures smooth changes
4. No need to stop and restart - speed changes on-the-fly
5. Serial monitor shows: "Speed adjusted to XX% (step size: X)"

**Example Workflow**:
1. Hold **Cap+** with step **1** selected → Motor starts slowly (25% speed)
2. While still holding, tap **100** button → Speed smoothly ramps up to 75%
3. Tap **1000** button → Accelerates to 100% maximum speed
4. Tap **10** button → Smoothly decelerates to 50% speed
5. Release button → Motor stops, original speed restored

**Technical Details**:
- **Hold threshold**: 400ms (optimized for resistive touchscreen)
- **Drift tolerance**: 30 pixels (allows natural finger movement during hold)
- **Touch polling**: 50ms interval (minimal overhead ~10-20µs per check)
- **Speed restoration**: Original max speed automatically restored when continuous mode exits
- **Smooth transitions**: AccelStepper library handles acceleration/deceleration curves

**Tips**:
- Continuous mode works with all four motor buttons independently
- Step size button highlighting updates when speed changes
- Stop button provides emergency halt during continuous operation
- Original motor speed configuration preserved - settings not affected
- Ideal for initial positioning (fast with 1000), then switch to precise adjustments (slow with 1)

---

## Button Reference

### Main Screen - Preset Mode

| Button | Location | Function |
|--------|----------|----------|
| **1-12** | Center grid | Quick-access preset buttons. Tap to load assigned preset. |
| **⚙ (Gear)** | Top-right | Opens settings menu |
| **Assigned Preset** | Varies | Shows preset name. Tap to load. Status bar shows movement. |
| **Unassigned** | Varies | Shows number only. Assign via Preset List screen. |

### Main Screen - Manual Mode

| Button | Location | Function |
|--------|----------|----------|
| **Cap -** | Left, Row 1 | **Tap**: Decrease capacitor by step size<br>**Hold 400ms**: Continuous backward movement at variable speed |
| **Cap +** | Right, Row 1 | **Tap**: Increase capacitor by step size<br>**Hold 400ms**: Continuous forward movement at variable speed |
| **Ind -** | Left, Row 2 | **Tap**: Decrease inductor by step size<br>**Hold 400ms**: Continuous backward movement at variable speed |
| **Ind +** | Right, Row 2 | **Tap**: Increase inductor by step size<br>**Hold 400ms**: Continuous forward movement at variable speed |
| **AUTO-TUNE** | Left half, Row 3 | Launch automatic tuning (see [Auto-Tune](#auto-tune)) |
| **STOP** | Right half, Row 3 | Emergency stop - halt all motor movement (also stops continuous mode) |
| **1** | Row 4, Col 1 | Set step size to 1 (fine adjustment, 25% speed in continuous mode) |
| **10** | Row 4, Col 2 | Set step size to 10 (50% speed in continuous mode) |
| **100** | Row 4, Col 3 | Set step size to 100 (75% speed in continuous mode) |
| **1000** | Row 4, Col 4 | Set step size to 1000 (coarse adjustment, 100% speed in continuous mode) |
| **⚙ (Gear)** | Top-right | Opens settings menu |

**Note**: Step size buttons can be tapped **during continuous mode** to change speed in real-time without releasing the motor button.

### Preset List Screen

| Element | Location | Function |
|---------|----------|----------|
| **Preset Row** | Center | Shows preset name, positions, SWR. Tap to load preset; hold ~0.6 s to edit it. |
| **[Assign]** | Middle | Tap to assign preset to on-screen quick-access buttons (1-12) and/or physical hardware buttons |
| **[X]** | Right | Delete preset (confirmation not required) |
| **Save Current** | Bottom-left | Save current motor positions as new preset |
| **Back** | Bottom-right | Return to Main Screen |

### Settings Screen

| Setting | Controls | Function |
|---------|----------|----------|
| **Brightness** | [-] [+] | Adjust display backlight (20-255) |
| **Screen Timeout** | [-] [+] | Backlight-off timeout in seconds (0 = always on, ±10s) |
| **SWR Alarm** | [-] [+] | Set alarm threshold (±0.1 per tap) |
| **Motor Speed** | [-] [+] | Max motor speed in steps/sec (±200) |
| **Motor Accel** | [-] [+] | Motor acceleration in steps/sec² (±100) |
| **M1 Homing Speed** | [-] [+] | Capacitor speed during homing operation (±50) |
| **M2 Homing Speed** | [-] [+] | Inductor speed during homing operation (±50) |
| **M1 Home Offset** | [-] [+] | Capacitor homing offset in steps (±10) |
| **M2 Home Offset** | [-] [+] | Inductor homing offset in steps (±10) |
| **M1 Home Inverted** | [Toggle] | Invert capacitor homing direction (for reversed motor/mount) |
| **M2 Home Inverted** | [Toggle] | Invert inductor homing direction (for reversed motor/mount) |
| **MC Backlash (steps)** | [-] [+] | Extra capacitor steps on each direction reversal (0-1000, ±1) |
| **ML Backlash (steps)** | [-] [+] | Extra inductor steps on each direction reversal (0-1000, ±1) |
| **Motor Auto-Disable** | [Toggle] | Cut stepper driver holding current once motors settle (reduces heat/current draw) |
| **SWR Good** | [-] [+] | Green/yellow threshold (±0.1) |
| **SWR Caution** | [-] [+] | Yellow/red threshold (±0.1) |
| **Touch Cal** | Button | Perform touchscreen calibration |
| **Hardware Cal** | Button | Motor limit switch calibration |
| **WiFi Toggle** | Button | Enable/disable WiFi (takes effect on reboot) |
| **Forget WiFi** | Button | Clear saved WiFi credentials |
| **Back** | Bottom | Return to Main Screen |

*Note: Settings screen is scrollable - swipe up/down to see all options*

### Calibration Screen

| Button | Function |
|--------|----------|
| **Touch Calibration** | Recalibrate touchscreen (5-point calibration) |
| **Set Motor 1 End** | Set maximum travel position for capacitor motor |
| **Set Motor 2 End** | Set maximum travel position for inductor motor |
| **Reset to Defaults** | Reset motor limits to 30,000 steps |
| **Back** | Return to Main Screen |

### Auto-Tune Screen

| Button | Function |
|--------|----------|
| **ABORT** | Shown while tuning is in progress (maroon) — stops the motors immediately |
| **DONE** | Shown once tuning finishes/fails/aborts (green if converged, grey otherwise) — returns to the Main Screen |

### SD Card Screen

| Button | Function |
|--------|----------|
| **Save Settings** | Save display, motor and SWR configuration to a new dated (or sequentially-numbered) file — never overwrites a previous save |
| **Load Settings** | Shows a scrollable list of saved settings files (newest first) — tap one to load it |
| **Save Presets** | Save all presets (including button assignments) to a new dated (or sequentially-numbered) file |
| **Load Presets** | Shows a scrollable list of saved preset files (newest first) — tap one to load it |
| **Save Settings** | Export display, motor and SWR configuration to SD card |
| **Load Settings** | Import configuration from SD card |
| **Save Presets** | Export all presets (including on-screen and physical button assignments) to SD card |
| **Load Presets** | Import presets and all button assignments from SD card |
| **Back** | Return to Main Screen |

### On-Screen Keyboard

| Area | Function |
|------|----------|
| **QWERTY Layout** | Full keyboard with numbers, letters, space |
| **Shift** | Toggle uppercase/lowercase |
| **Del** | Delete last character (bottom row for the name; on the keypad for kHz) |
| **Back** | Leave without saving and return to the preset list |
| **Update C/L** | Edit only: sets this preset's C/L to the current motor positions (and SWR) after a warning showing stored vs. current C/L. Saved with OK, discarded with Back |
| **OK** | Confirm text entry and save |

---

## Menu System

### Accessing the Menu

Tap the **⚙ (Gear Icon)** in the top-right corner of the Main Screen.

### Menu Options

```
┌───────────────────┐
│ Presets          │ → Preset List Screen
├───────────────────┤
│ Settings         │ → Settings Screen
├───────────────────┤
│ SD Card          │ → SD Card Screen
├───────────────────┤
│ Manual Mode / Preset Mode │ → Switch to the other mode (label shows destination)
└───────────────────┘
```

**To close menu**: Tap anywhere outside the menu box or select an option.

---

## Preset Management

### What is a Preset?

A preset stores:
- Motor 1 position (Capacitor)
- Motor 2 position (Inductor)
- Preset name (up to 20 characters)
- Last measured SWR (optional)

### Saving a New Preset

1. **Switch to Manual Mode** (if not already): ⚙ → "Manual Mode"
2. **Adjust motors** to desired position using +/- buttons
3. **Verify SWR** is acceptable
4. **Open Presets**: ⚙ → "Presets"
5. **Tap "Save Current"** button
6. **Enter name** using on-screen keyboard (e.g., "20m CW", "40m Phone")
7. **Tap "OK"** to save

### Loading a Preset

**Method 1: From Preset List**
1. ⚙ → "Presets"
2. Tap preset row
3. Motors move to saved positions
4. Returns to Main Screen

**Method 2: Quick-Access Button (Preset Mode only)**
1. Ensure preset is assigned to a button (see below)
2. From Main Screen, tap assigned button (1-12)
3. Motors move immediately to saved positions
4. Button shows status:
   - **Blue bar**: Motors moving
   - **Red bar**: Motors at position

**Method 3: Physical Hardware Button (requires MCP23017 port expander)**
1. Ensure the physical button is assigned to a preset (see *Assigning Presets* below)
2. Press the physical button at any time (system must not be transmitting)
3. Motors move to the assigned preset positions
4. The corresponding on-screen preset button lights up (if also assigned)

### Assigning Presets to Quick-Access Buttons

1. ⚙ → "Presets"
2. Find preset in list
3. Tap **[Assign]** area (middle of row)
4. The assignment dialog opens, showing two sections:

**Virtual Buttons (1–12)**:
- Shows the 12 on-screen quick-access buttons in a 3×4 grid
- **Green** = already assigned to this preset
- **Orange** = assigned to a different preset
- **Grey** = unassigned
- Tap a button number to toggle its assignment

**Physical Buttons** (only shown if MCP23017 port expander is detected):
- Shows currently assigned hardware button numbers (e.g. "3,7") or "[none]"
- Tap **[Press btn]** to enter listen mode — the button turns red and shows "Cancel"
- Press any physical hardware button on your panel; its assignment to this preset is toggled immediately
- Tap **[Cancel]** to exit listen mode without making a change

5. Tap **Done** to close the dialog
6. All assignments are saved automatically

**Notes**:
- One preset can be assigned to multiple on-screen or physical buttons
- Assignments persist across power cycles (stored in NVS flash)
- If no MCP23017 port expander is fitted, the Physical Buttons section does not appear

### Deleting a Preset

1. ⚙ → "Presets"
2. Find preset in list
3. Tap red **[X]** on right side
4. Preset deleted immediately (no confirmation)
5. Any on-screen and physical button assignments for this preset are automatically cleared; all other assignments are shifted to remain consistent

### Editing a Preset

1. ⚙ → "Presets"
2. **Tap and hold** the preset row for about 0.6 s (a short tap loads it instead)
3. The keyboard opens as **Edit Preset** with the current name and frequency range filled in
4. Change the name and/or the **Min kHz** / **Max kHz** fields
5. Optional: tap **Update C/L** to take the current motor positions (and SWR), e.g. after
   re-tuning the band in Manual Mode. A warning shows the stored and current C/L; tap
   **Overwrite** to accept or **Cancel**. The button then shows the new C/L, which is saved
   only when you press **OK**
6. Tap **OK** to save, or **Back** to leave without changes

The preset keeps its place in the list and its button assignments. A name that another preset
already uses is rejected, and Min kHz must not be above Max kHz.

To clear a kHz value, tap the field and **Del** all digits.

### Preset List Display

Each preset row shows:
- **Name**: User-defined preset name
- **[Assign]**: Button assignment area (shows on-screen button numbers if assigned)
- **Positions**: Motor 1 and Motor 2 positions
- **SWR**: Last recorded SWR value
- **[X]**: Delete button

### Storage Limits

- **Maximum presets**: 50
- **Storage**: Internal flash memory (persistent)
- **Backup**: Optional SD card export/import (settings and presets saved separately)

---

## Settings & Configuration

Access via: ⚙ → "Settings"

### Display Settings

#### Brightness
- **Range**: 20-255 (0 = off, 255 = maximum)
- **Default**: 200
- **Adjust**: [-] decreases by 20, [+] increases by 20
- **Effect**: Immediate
- **Use Case**: Reduce for night operation, increase for bright sunlight

#### Screen Timeout
- **Range**: 0-3600 seconds (0 = always on)
- **Default**: 60 seconds
- **Adjust**: [-] decreases by 10 s, [+] increases by 10 s
- **Effect**: Backlight turns off after the set period of inactivity
- **Wake**: Tap anywhere on the screen to restore full brightness — the activating touch is ignored so it cannot trigger controls accidentally
- **Persistence**: Setting is saved to flash and survives reboots
- **Use Case**: Set to 0 for permanent-on displays; use a short value (e.g. 60 s) during normal operation to protect the TFT backlight and save power

### SWR Settings

#### SWR Alarm Threshold
- **Range**: 1.0-5.0
- **Default**: 2.0
- **Adjust**: [-] decreases by 0.1, [+] increases by 0.1
- **Effect**: Sets point where audio/visual alarm triggers
- **Note**: Only active in Preset Mode

#### SWR Good Threshold (Green/Yellow)
- **Range**: 1.0-3.0
- **Default**: 1.5
- **Purpose**: Color-coding boundary
- **Below this**: Green display (good match)
- **Above this**: Yellow display (caution)

#### SWR Caution Threshold (Yellow/Red)
- **Range**: 1.0-5.0
- **Default**: 2.0
- **Purpose**: Color-coding boundary
- **Below this**: Yellow display (caution)
- **Above this**: Red display (high SWR)

### Motor Settings

#### Motor Speed
- **Range**: 200-4000 steps/second
- **Default**: 2000
- **Adjust**: [-] decreases by 200, [+] increases by 200
- **Effect**: Maximum speed during movement
- **Lower = slower/quieter**, **Higher = faster/noisier**

#### Motor Acceleration
- **Range**: 100-2000 steps/second²
- **Default**: 1000
- **Adjust**: [-] decreases by 100, [+] increases by 100
- **Effect**: How quickly motors reach max speed
- **Lower = gradual**, **Higher = aggressive**

#### Homing Speed (M1 / M2, independent)
- **Range**: 100-2000 steps/second
- **Default**: 2000
- **Adjust**: [-] decreases by 50, [+] increases by 50
- **Effect**: Speed used during automatic homing, set independently for the
  capacitor (M1) and inductor (M2) motors
- **Recommendation**: Keep at a moderate speed for reliable limit switch
  detection — if one motor's homing feels too fast/unreliable, you can now
  slow just that motor down without affecting the other

#### Backlash Compensation (MC / ML, independent)
- **Range**: 0-1000 steps
- **Default**: 0 (off)
- **Adjust**: [-] decreases by 1, [+] increases by 1
- **Problem it solves**: with microstepping (e.g. 1/8), a motor turning
  against friction lags slightly behind its commanded position. After a
  direction change the first steps only take up that lag, so the first move
  after a reversal is shorter than the following ones. This is
  magnetic/electrical, not mechanical slack, and grows with load torque.
- **Effect**: on every direction reversal the motor runs this many extra
  steps in the new direction. They are **not** counted: the displayed
  position, saved presets and the web/REST position always show the value
  you asked for. Moves in the same direction are unaffected. The
  compensation resets after homing, and is reduced near the 0/end limits so
  the motor never runs past them.
- **How to measure**:
  1. Set the motor's backlash to 0.
  2. On the Manual screen, jog a few times in one direction, then reverse.
  3. Compare how far the first move after the reversal travels with a
     normal move of the same size. The difference (in steps) is the value
     to enter.
  4. Repeat +10 ×3, −10 ×3, +10 ×3: every move should now travel the same
     distance and the position should return to its start value.
- **Tip**: raising the TB6600 current (within the motor's rating) or
  reducing friction lowers the lag and therefore the value needed. Leave
  a motor that does not show the problem at 0.

### Calibration

#### Touch Calibration
- **Function**: Recalibrate touchscreen for accurate touch detection
- **When to use**: If buttons seem unresponsive or misaligned
- **Process**: Follow on-screen prompts to tap 5 targets
- **Duration**: ~30 seconds

#### Hardware Calibration
- **Function**: Set motor travel limits and verify limit switches
- **Access**: Opens dedicated Calibration Screen
- **See**: [Calibration](#calibration) section

### WiFi Settings

#### WiFi Toggle
- **Function**: Enable/disable WiFi on next boot
- **Default**: Enabled
- **When disabled**: Skips WiFi connection, slightly faster boot. WiFi, the
  web server, telnet and OTA stay completely off until re-enabled (no
  background reconnect attempts)
- **Use Case**: Disable if not using OTA updates or remote debugging

#### Forget WiFi
- **Function**: Clear saved WiFi credentials
- **Confirmation**: Required (Yes/No dialog)
- **Effect**: Device will create AP on next boot for reconfiguration
- **Use Case**: Changing network or troubleshooting connection issues

### Scrolling the Settings

The Settings screen has more content than fits on one screen.

**To scroll**:
- **Swipe up**: See later settings (WiFi options)
- **Swipe down**: Return to earlier settings (Brightness)
- Scroll position resets when leaving screen

---

## Calibration

### Touch Calibration

**Purpose**: Ensure accurate touchscreen response

**When needed**:
- First power-up
- After firmware update
- If touches are offset or unresponsive

**Process**:
1. White crosshair appears at screen corner
2. Tap center of crosshair precisely
3. Repeat for 5 positions (corners + center)
4. Calibration saved to flash memory
5. Returns to previous screen

**Tips**:
- Use stylus or fingernail for precision
- Tap firmly but briefly
- If calibration seems wrong, redo from Settings

### Hardware Calibration

**Purpose**: Configure motor travel limits and verify limit switches

**Access**: ⚙ → Settings → "Hardware Calibration"

#### Calibration Screen Layout

```
┌─────────────────────────────────────────┐
│ Hardware Calibration                    │
├─────────────────────────────────────────┤
│                                          │
│ Motor 1 (Capacitor)                     │
│ Current Position: 12345                 │
│ End Position: 30000                     │
│ [ Set Motor 1 End Position ]            │
│                                          │
│ Motor 2 (Inductor)                      │
│ Current Position: 23456                 │
│ End Position: 30000                     │
│ [ Set Motor 2 End Position ]            │
│                                          │
│ [ Touch Calibration ]                   │
│                                          │
│ [ Reset to Defaults ]                   │
│                                          │
│ [ Back ]                                │
└─────────────────────────────────────────┘
```

#### Setting Motor End Positions

The "End Position" defines the maximum travel distance from the home position (limit switch).

**Procedure**:
1. Tap **"Set Motor X End Position"**
2. Motor extends fully until mechanical stop OR current position
3. Position is saved as maximum
4. Use this to prevent over-travel damage

**Default**: 30,000 steps (can accommodate most tuner designs)

**When to adjust**:
- Your tuner has shorter travel range
- You want to limit range for safety
- Physical stops are reached before 30,000 steps

#### Resetting to Defaults

Tap **"Reset to Defaults"** to:
- Motor 1 end: 30,000 steps
- Motor 2 end: 30,000 steps

#### Touch Calibration (from this screen)

Same as Touch Calibration from Settings screen - shortcut placement.

#### Power Calibration (RF meter)

The **RF Meter Calibration** section sets a forward and a reflected
correction factor per band (×1000, so 1000 = no correction, 1100 = reads
10 % higher). You need a reference wattmeter in line with a dummy load.

**Procedure**:
1. Key a steady carrier on the band to calibrate. While a carrier is present,
   **Band** follows the measured frequency, because the meter applies the
   factors of the band it measures.
2. **Meas** shows the forward and reflected power the meter reads, refreshed
   about twice a second.
3. Tap the step button on the Forward row (**x1 / x10 / x100**, shared by
   Forward and Reflected), then use **−** / **+**. Each tap takes effect on
   the meter immediately (status "Preview - not saved"), so Meas follows.
4. Repeat until Meas matches the reference wattmeter.
5. Tap **Save To RF Meter** to store the band's factors in the meter
   ("Saved to RF meter"). An unsaved value is lost when the meter restarts.
6. Repeat for each band you use. Reflected is calibrated the same way, with
   a known reflected power (e.g. the coupler reversed).

Allowed range per factor: 100–10000.

#### Frequency Correction (RF meter)

Scroll to the bottom of the screen. The RF meter times its frequency
measurement from an internal oscillator that can be off by 1–2 %. The
**PPM** value corrects every frequency reading by that ratio. The meter
stores it, so it survives power cycles.

**Procedure**:
1. Key a carrier at a known frequency. A higher band gives a finer result, e.g. 28.000 MHz.
2. **Meas** shows the frequency the meter reads. It refreshes by itself about twice a second.
3. Tap the step button (**x1 / x10 / x100 / x1000**) to pick the step size, then use **−** / **+** to adjust PPM. A positive PPM raises the reading.
   Each tap takes effect on the meter immediately (status "Preview - not saved"), so Meas follows.
4. Repeat until Meas matches the carrier.
5. Tap **Save PPM To RF Meter** to store it in the meter. An unsaved value is lost when the meter restarts.

Starting from the current value, the exact correction is
`new = (1000000 + old) × f_true / f_read − 1000000`. For example, 28.000 MHz
read as 27.860 MHz at 0 ppm gives +5025. The allowed range is ±30000 ppm.

#### Verifying Limit Switches

**New Homing Sequence**:
1. If switch already active → backs off 300 steps
2. Moves forward until switch activates
3. Stops immediately
4. Applies configured offset (default 100 steps)
5. Sets position as home (zero)

**Testing**:
1. Home motors (power cycle or Home button)
2. Observe homing status screen
3. Both motors should show "Complete" in green
4. If "FAILED" appears:
   - Check wiring (GPIO 32, GPIO 33)
   - Verify switches activate when pressed (LOW when pressed)
   - **Important**: If switch stuck active, motor backs off then tries again
   - Verify switch deactivates after 300-step backoff
   - Check motor power (12-24V) and TB6600 drivers
   - If backoff fails, switch may be shorted or mechanically stuck

**Adjusting Homing Position**:
- Use Motor Homing Offset settings to fine-tune final home position
- Positive offset = further from limit (safer)
- Test after adjustment by power cycling and re-homing

**Inverting Homing Direction**:
- If a motor homes in the wrong direction (moves away from limit switch), enable the corresponding **M1/M2 Home Inverted** toggle in Settings
- This reverses the direction of the initial sweep and pre-backoff move
- Useful when a motor is physically mounted in reverse, or wired with DIR signal inverted at the TB6600
- Setting is saved to flash and takes effect on the next homing sequence

---

## Auto-Tune

**Purpose**: Automatically find the capacitor and inductor positions with the lowest SWR, the same way you would tune by hand.

**Access**: Main Screen → Manual Mode → tap **AUTO-TUNE**

**Before you start**:
- Motors must have completed homing successfully, and the RF meter must be connected
- Transmit a **low-power carrier** (CW or FM) on the frequency to tune, between the min and max tune power (default 1-20 W)
- The tune waits up to 60 s for the carrier to appear after you tap AUTO-TUNE

### How It Works

1. **Dip search**: the capacitor is set to 50 % capacitance and the roller inductor runs slowly across its whole travel while the SWR is sampled. As soon as a clear **dip** in SWR has been passed, the inductor stops. If no dip is seen, the capacitor is moved to 75 %, then 25 %, 100 % and 0 %, sweeping the inductor each time (alternating direction, so no time is lost returning).
2. **Refinement**: starting at the dip, C and L are adjusted **together** (they influence each other) with a pattern search that shrinks its steps as it closes in on the lowest SWR.
3. **Finish**: the motors move to the best position found and a final reading decides the result.

If the SWR at the start is already 3.0 or lower, the dip search is skipped and the tuner refines from where it is - fast when you have only moved a little in frequency.

### Auto-Tune Screen Layout

```
┌─────────────────────────────────────────┐
│ AUTO-TUNE                               │
├─────────────────────────────────────────┤
│ Status: Searching for dip               │
│ Dip search C=75% (2/5), L sweep 42%     │
│ Current SWR:      4.30                  │
│ Best SWR:         3.10                  │
│ Target SWR:       1.5                   │
│ Progress: [██████░░░░░░░░] 40%          │
│ Time: 48.0s        Step: 2/5            │
│ Best pos: C=600  L=21400                │
│                                         │
│          [        ABORT        ]        │
└─────────────────────────────────────────┘
```

The second line shows what the tuner is doing right now: the dip search attempt and sweep progress, or the refinement step sizes (`Refine dC=8 dL=120 SWR 1.32`). **Step** counts dip-search attempts (of 5), then refinement evaluations.

**Results**:
| Result | Meaning |
|--------|---------|
| **Converged** (green) | Best SWR is at or below the target SWR |
| **Failed: No dip found** | None of the five capacitor settings gave a dip; motors return to where they started |
| **Failed: Best SWR x.x (target y.y)** | Motors are at the best position found, but it is above the target |
| **Failed: Timeout / RF power lost / No RF power** | The time limit was reached, or the carrier stopped for more than 10 s |
| **Aborted** | Stopped with ABORT, too much power, or the SWR ran away during refinement |

**Aborting**: Tap **ABORT** at any time. The motors stop where they are (not necessarily at the best position found).

**Saving the result**: after a successful tune, save the position as a preset as usual.

### Auto-Tune Settings

On the Settings screen (locked while a tune runs):

| Setting | Default | Range | Effect |
|---------|---------|-------|--------|
| Tune Target SWR | 1.5 | 1.1 - 3.0 | Result counts as Converged at or below this |
| Tune L Sweep (st/s) | 1000 | 100 - 4000 | Inductor speed during the dip search: slower finds narrow dips more reliably, faster saves time |
| Tune Max Time (s) | 600 | 60 - 1200 | Overall time limit |
| Tune Min Power (W) | 1.0 | 0.1 - 10 | Below this the tune pauses |
| Tune Max Power (W) | 20 | 1 - 100 | Above this the tune aborts immediately |
| Tune Settle (ms) | 300 | 100 - 2000 | Wait after each move before measuring |

**Also available remotely**: `POST /api/autotune/start` / `/stop` and `GET /api/autotune/status` from the web interface's Motors page or any HTTP client — see [REST_API.md](REST_API.md).

---

## SD Card Operations

**Purpose**: Backup/restore presets and settings to external storage

**Requirements**:
- SD card module connected via SPI
- SD card formatted as FAT32
- Chip select on GPIO 22

**Hot-insert supported**: The SD card can be inserted at any time. The device will attempt to mount it automatically on the next SD card operation — no restart required.

**Access**: ⚙ → "SD Card"

### SD Card Screen Layout

```
┌─────────────────────────────────────────┐
│ SD Card Operations                      │
├─────────────────────────────────────────┤
│                                          │
│ Status: Card Detected / Not Found       │
│                                          │
│ [ Save Settings ]                       │
│                                          │
│ [ Load Settings ]                       │
│                                          │
│ [ Save Presets ]                        │
│                                          │
│ [ Load Presets ]                        │
│                                          │
│ [ Back ]                                │
│                                          │
└─────────────────────────────────────────┘
```

### Saving Settings

**What's saved**:
- Display settings (brightness, SWR thresholds)
- Motor settings (speed, acceleration, homing speed, homing offset/inversion,
  auto-disable, backlash compensation)
- Auto-tune settings (target SWR, sweep speed, time limit, power window, settle delay)
- Auto-tune settings (target SWR, sweep speed, time limit, power window, settle delay)
- SWR alarm threshold

**File created**: a new file every time — never overwrites a previous save. Named
`/staytuned/settings_<YYYYMMDD_HHMMSS>.json` if WiFi is connected and the
clock is NTP-synced (e.g. `settings_20260804_213045.json`), or
`/staytuned/settings_<NNNN>.json` with an auto-incrementing 4-digit counter
otherwise (e.g. `settings_0007.json`). The generated filename is shown in
the confirmation message.

### Loading Settings

Tap **Load Settings** to see a scrollable list of all saved settings files,
**newest first**. Tap a row to load that file's configuration values.

### Saving Presets

**What's saved**:
- All tuning presets (name, motor positions, SWR)
- On-screen button assignments for each preset (which of the 12 quick-access buttons it is assigned to)
- Physical button assignments for each preset (which MCP23017 hardware button it is assigned to, if any)

**File created**: a new file every time, using the same date/time-or-counter
naming as Settings above — e.g. `/staytuned/presets_20260804_213045.json`
or `/staytuned/presets_0007.json`.

### Loading Presets

Tap **Load Presets** to see a scrollable list of all saved preset files,
**newest first**. Tap a row to load it — all presets replace the current
preset list, and button assignments are restored from the file and saved
to flash.
**What's loaded**:
- All presets replace the current preset list
- On-screen and physical button assignments are restored from the file and saved to flash

**File required**: `/staytuned/presets.json`

**Note**: If the file was created on a different unit the button assignments may not match your layout — edit the `"button"` or `"physicalButton"` fields in the JSON file to reassign, or use the Preset List screen to reassign after loading.

**No files found**: If no matching files exist yet, tapping Load shows "No saved files found" instead of a list.

### SD Card Not Detected

If "Card Not Found" displays:
- Check SD card is inserted (contacts up)
- Verify FAT32 formatting
- Check SD module wiring
- Verify CS pin (GPIO 22)
- Try another SD card operation after inserting — mount is retried automatically

### JSON Format

Files are human-readable JSON and can be edited with any text editor.

**Example `/staytuned/presets_20260804_213045.json`**:
```json
[
  {
    "name": "20m CW",
    "motorCapacitor": 12500,
    "motorInductor": 8300,
    "freqMin": 14000000,
    "freqMax": 14350000,
    "swr": 1.2,
    "timestamp": 0,
    "button": 0,
    "physicalButton": 2
  },
  {
    "name": "40m SSB",
    "motorCapacitor": 24000,
    "motorInductor": 16000,
    "freqMin": 7000000,
    "freqMax": 7300000,
    "swr": 1.4,
    "timestamp": 0,
    "button": -1,
    "physicalButton": -1
  }
]
```

The `"button"` field is the on-screen quick-access button slot (0–11). Set to `-1` if not assigned.

The `"physicalButton"` field is the MCP23017 hardware button index (0–15). Set to `-1` if not assigned to a physical button. This field is only acted on if a port expander is present; it is silently ignored otherwise.

---

## About Screen

**Access**: ⚙ → Settings → About

The touchscreen About screen shows system info and provides two SD card firmware update actions.

**Information Displayed**:
- **Creator**: OZ9KHA
- **Firmware Version**: Current software version
- **Compile Date/Time**: Build timestamp
- **IP Address**: Current WiFi IP address (or "Not connected" if WiFi is not joined)
- **WebSocket Clients**: List of connected web client IPs (refreshes every 2 seconds)

**Buttons**:

| Button | Colour | Condition | Action |
|--------|--------|-----------|--------|
| Update Firmware | Navy (active) | SD card available | Opens a scrollable list of `.bin` files found on the SD card |
| SD unavailable | Grey (inactive) | No SD card detected | No action |
| RF Meter FW | Dark Green (active) | SD card available and RF meter hardware detected | Opens a scrollable list of `.hex` files found on the SD card |
| Meter absent | Grey (inactive) | RF meter board not detected on I2C | No action |
| SD unavailable | Grey (inactive) | No SD card detected | No action |
| Back | Maroon | Always | Return to Settings |

If the SD card is available but no matching files are found, the respective list screen shows "No .bin files found on SD card" or "No .hex files found on SD card" instead of a list.

---

## Firmware Update from SD Card

**Purpose**: Flash new firmware directly from the SD card — no USB cable or WiFi connection required.

Two independent, symmetric update flows are available from **Settings → About**, each browsing the SD card for matching files and letting you pick which one to flash:
- **Tuner update** — browses `.bin` files, flashes the ESP32 itself
- **RF meter PIC update** — browses `.hex` files, flashes the RF meter board's PIC over I2C

**Requirements**:
- SD card inserted and readable
- One or more valid ESP32 firmware `.bin` files (e.g. exported from PlatformIO) in the root directory of the SD card, for a tuner update
- One or more valid RF meter `.hex` files in the root directory of the SD card, for an RF meter update
- Any filename with the matching extension is picked up — not a single fixed name — so you can keep multiple versions side-by-side (e.g. `firmware-1.1.16.bin`, `rf_meter-2.3.hex`)

### Tuner Firmware Procedure

1. **Copy** one or more `.bin` firmware files to the root of a FAT32-formatted SD card
2. **Insert** the SD card into the StayTuned device
3. Navigate to **Settings → About**
4. The **"Update Firmware"** button will appear active (navy blue) if the SD card is available
5. **Tap the button** — a scrollable list of every `.bin` file found on the card is shown (up to 6 at a time, scroll for more)
6. **Tap a file** — a confirmation dialog appears showing the filename and a "This will reboot the device" warning
7. **Tap Yes** to proceed (or Cancel to return to the list without flashing) — a progress screen appears:
   - File size is displayed
   - A green progress bar fills as the firmware is written
8. On success: **"Success! Rebooting..."** — device reboots into the new firmware automatically
9. On failure: an error message is shown and the device returns to the file list

---

### RF Meter PIC Firmware Procedure

1. **Copy** one or more `.hex` firmware files to the root of a FAT32-formatted SD card
2. **Insert** the SD card into the StayTuned device
3. Navigate to **Settings → About**
4. The **"RF Meter FW"** button appears active (dark green) if the SD card is available and the RF meter board is detected on I2C
5. **Tap the button** — a scrollable list of every `.hex` file found on the card is shown (up to 6 at a time, scroll for more), newest version first
6. **Tap a file** — a confirmation dialog appears showing the filename and a "This will update the RF meter" notice
7. **Tap Yes** to proceed (or Cancel to return to the list without flashing) — a progress screen appears, running through phases:
  - Checking
  - Preparing
  - Erasing
  - Writing
  - Verifying
  - Committing
8. Target image is selected automatically using bootloader inactive-slot selection (`IMAGE_SELECT_AUTO`)
9. On success: success message is shown and the file list is restored
10. On failure: updater error text is shown and the file list is restored

**Canceling RF Meter Update**:
- While RF meter update is running, tap **Cancel** on the update screen
- Status changes to **Cancelling...** and updater exits safely

**Safety interlocks**: an RF meter flash is blocked (with an on-screen message) if the motors are currently running or an auto-tune is in progress — retry once the tuner is idle.

### Button States

| Button Text | Colour | Meaning |
|-------------|--------|---------|
| Update Firmware | Navy (active) | SD card available — tap to browse `.bin` files |
| SD unavailable | Grey (inactive) | No SD card detected |

RF meter update button states:

| Button Text | Colour | Meaning |
|-------------|--------|---------|
| RF Meter FW | Dark Green (active) | SD card available and RF meter hardware detected — tap to browse `.hex` files |
| Meter absent | Grey (inactive) | RF meter board not detected on I2C |
| SD unavailable | Grey (inactive) | No SD card detected |

### Getting a firmware.bin

**From PlatformIO**:
```
pio run -e esp32
```
The compiled binary is at `.pio/build/esp32/firmware.bin`.

**Via OTA build**:
```
pio run -e esp32-ota
```
Same output path — copy the binary to the SD card before uploading via the SD method. You can rename it or keep multiple versions side-by-side (e.g. `firmware-1.1.16.bin`) since the picker lists every `.bin` file present.

### Tips

- **Backup presets first**: Use SD Card → Save Presets/Save Settings before updating firmware — the update does not affect NVS (presets and settings are preserved)
- **Do not power off** during a tuner update — a failed partial write may require reflashing via USB
- Both pickers accept any filename with the matching extension (`.bin` or `.hex`) — no fixed name is required
- After a tuner update reboot, or an RF meter update completes, verify the new version on the About screen

### Comparison of Update Methods

| Method | Requires WiFi | Requires USB | Requirements |
|--------|:---:|:---:|---|
| USB (initial flash) | No | Yes | USB cable, PlatformIO |
| OTA via WiFi | Yes | No | Same WiFi network, browser |
| **SD Card tuner update** | **No** | **No** | **SD card with one or more `.bin` files** |
| **SD Card RF meter update** | **No** | **No** | **SD card with one or more `.hex` files** |

---

## SWR Alarms

**Purpose**: Alert operator to high SWR conditions that may damage equipment

### Alarm Behavior

#### When Alarm Triggers

**Condition**: SWR exceeds alarm threshold (default 2.0) **AND** system is in Preset Mode

**Visual Alert**:
- Red popup window appears (center screen)
- Text: "WARNING: HIGH SWR" / "TAP TO SILENCE"
- Popup blocks all other screen interactions

**Audio Alert**:
- Continuous 1000Hz tone from speaker (GPIO 2)
- Sounds immediately when alarm triggers

**Blockage**:
- All buttons are disabled
- Menu cannot be opened
- Preset buttons do not respond
- **ONLY** the alarm popup can be touched

#### Alarm in Manual Mode

**Important**: Alarms do **NOT** trigger in Manual Mode.

**Reason**: High SWR is expected and normal during manual tuning. Operator is actively adjusting and monitoring.

**Effect**: You can tune freely without alarm interruptions.

#### Silencing the Alarm

1. **Tap anywhere on red popup**
2. Audio tone stops immediately
3. Popup remains visible (as reminder)
4. Screen interactions remain blocked

**Note**: Alarm cannot be dismissed while SWR is still high - only silenced.

#### Alarm Clearing

**Automatic**: When SWR drops below threshold
- Popup disappears
- Screen returns to normal operation
- All buttons become responsive

**Manual**: Switch to Manual Mode
- Alarm is disabled in Manual Mode
- Popup cleared
- Allows manual adjustment to reduce SWR

### Alarm Configuration

#### Setting Alarm Threshold

1. ⚙ → Settings
2. Find "SWR Alarm Threshold"
3. Adjust with [-] [+] buttons
4. **Range**: 1.0-5.0
5. **Default**: 2.0
6. **Increment**: 0.1 per tap

**Recommendations**:
- **2.0**: Standard for solid-state PAs
- **3.0**: More tolerant for tube rigs
- **1.5**: Conservative for QRP

#### Disabling Alarms

Alarms cannot be permanently disabled, but are automatically:
- **Disabled** in Manual Mode
- **Enabled** in Preset Mode

**Workaround**: Stay in Manual Mode if alarms are unwanted.

### Troubleshooting False Alarms

If alarm triggers unexpectedly:

1. **Check SWR reading**: Is it actually high?
2. **Verify threshold**: Settings → SWR Alarm
3. **Calibrate SWR bridge**: Ensure proper coupling factor
4. **Check RF connections**: Loose connections cause reflections
5. **Reduce power**: Test with low power first
6. **Re-tune**: Preset may have drifted, retune and save

---

## WiFi & OTA Updates

### WiFi Connection

#### First-Time Setup

1. **Power on device** with WiFi enabled
2. **Access Point created**: 
   - SSID: **"StayTuned"**
   - Password: **"staytuned2026"**
3. **Connect** from phone/computer
4. **Captive portal** opens automatically
5. **Select your WiFi network** from list
6. **Enter password** and save
7. **Device connects** to your network
8. **AP shuts down** after successful connection

#### Status Indicator

- **WiFi icon** shown on Main Screen header
- **Connected**: Icon present
- **Not connected**: Icon absent

#### Reconnection

Device automatically reconnects to saved network on boot:
- **Max retries**: 20 attempts
- **Retry interval**: 30 seconds
- **Fallback**: Creates AP if all retries fail

### mDNS Hostname

Device is accessible as: **`staytuned.local`**

**Uses**:
- OTA updates: `pio run -e esp32-ota --target upload` resolves `staytuned.local` via mDNS (not HTTP — the espota protocol)
- Web interface: `http://staytuned.local`
- REST API access
- Network identification

### OTA (Over-The-Air) Updates

**Purpose**: Update firmware without a USB cable, from a PC on the same network.

**Important**: OTA updates use the `ArduinoOTA` (espota) protocol, not a browser upload page — there is no firmware-upload form at `http://staytuned.local`. Updates are pushed from a PC running PlatformIO.

**Requirements**:
- Device connected to WiFi
- PC on the same network with [PlatformIO](https://platformio.org/) installed
- This project's source checked out (or at minimum, its compiled `.bin` and `platformio.ini`)

#### Performing an OTA Update

1. On your PC, open a terminal in the project directory
2. Run:
   ```bash
   pio run -e esp32-ota --target upload
   ```
3. PlatformIO builds the firmware, finds the device via mDNS (`staytuned.local`), and pushes it over WiFi
4. Progress is shown as a percentage in the terminal
5. Device reboots automatically into the new firmware once the transfer completes

If `staytuned.local` doesn't resolve, use `pio run -e esp32-ota-ip --target upload` after editing `upload_port` in `platformio.ini` with the device's IP address (see the About screen or your router for the current IP).

**Tips**:
- Use a stable WiFi connection — an interrupted transfer can leave the device unable to boot, requiring a USB reflash
- Backup presets/settings to SD card first (see [SD Card Operations](#sd-card-operations))
- After reboot, verify the new version on the About screen

#### OTA Security

- **Password**: None by default — anyone on the network can push an OTA update
- **To secure**: Set `OTA_PASSWORD` in `include/config.h` and recompile/reflash

---

## Web Interface

### Overview

StayTuned includes a built-in web interface for remote monitoring and control when connected to WiFi. The interface is responsive and works on phones, tablets, and desktop browsers.

**Access**: `http://staytuned.local` or `http://[device-ip]`

**Features**:
- Real-time status monitoring via WebSocket
- Preset loading and saving
- Manual motor control
- System information
- Multiple simultaneous connections supported

### Web Pages

The web interface consists of three main pages accessible via navigation tabs:

#### 1. Presets Page (Default)

**URL**: `http://staytuned.local/`

**Purpose**: Monitor system status and quickly load saved presets

**Features**:
- **Status Panel**:
  - SWR (color-coded: Green ≤1.5, Yellow ≤2.0, Red >2.0)
  - Forward/Reflected Power + Forward PEP
  - Motor positions (Capacitor/Inductor)
  - Real-time updates via WebSocket

- **Preset List**:
  - All saved presets displayed as buttons
  - Shows preset name and ID
  - Click to load preset
  - Visual indicator shows active preset:
    - Blue bar = motors moving to position
    - Red bar = motors at position
  - TX lockout: Cannot load presets while transmitting (≥5W)

- **Save Button**:
  - Save current motor positions as new preset
  - Prompts for preset name
  - Immediately available after saving

**Navigation**: Switch between pages using top navigation bar

#### 2. Motors Page

**URL**: `http://staytuned.local/motors`

**Purpose**: Manual motor control for fine-tuning

**Features**:
- **Status Display**: Same real-time status as Presets page

- **Motor Controls**:
  - **Capacitor Motor**: [−] and [+] buttons
  - **Inductor Motor**: [−] and [+] buttons
  - Position values displayed next to each motor
  
- **Step Size Selector**:
  - Buttons: 1, 10, 100, 1000 steps
  - Changes increment for +/− buttons
  - Active step size highlighted

- **System Controls**:
  - **Home**: Return both motors to zero position
  - **Stop**: Emergency stop (halt all movement)
  - **Save Preset**: Save current positions with name

- **Confirmation Dialogs**:
  - Home operation requires user confirmation
  - Prevents accidental homing

**Use Cases**:
- Fine-tuning antenna match
- Creating new presets
- Testing motor operation
- Emergency motor stop

#### 3. About Page

**URL**: `http://staytuned.local/about`

**Purpose**: System information and diagnostics

**Information Displayed**:
- **Firmware Version**: Current software version
- **Compile Date/Time**: Build timestamp
- **IP Address**: Device's current WiFi IP address
- **Connected WebSocket Clients**: 
  - Number of active web connections
  - Client IP addresses
  - Client connection IDs

**Updates**: Client list refreshes every 2 seconds

**Use Cases**:
- Verify firmware version
- Find device IP address for direct access
- Check active connections
- Troubleshoot connectivity
- Identify connected devices

### WebSocket Communication

**Endpoint**: `ws://staytuned.local/ws`

**Update Rate**: 100ms (10 Hz) for smooth real-time display

**Status Message Format**:
```json
{
  "type": "status",
  "swr": 1.5,
  "fwd": 50.0,
  "ref": 2.5,
  "mc": 12345,
  "ml": 23456,
  "moving": false,
  "activePreset": 0
}
```

**Connection Management**:
- Automatic reconnection on disconnect
- Multiple clients supported simultaneously
- Clean disconnection on page close
- Connection logged to SD card (if available and time synced)

### Usage Tips

#### Bookmark for Quick Access
Add `http://staytuned.local` to browser bookmarks for easy access

#### Multiple Devices
- Control from phone while monitoring on tablet/PC
- Up to 4 simultaneous WebSocket clients supported
- All clients see synchronized status updates

#### Mobile Optimization
- Responsive design adapts to screen size
- Touch-friendly buttons (minimum 40px height)
- Works in portrait and landscape modes
- No pinch-zoom needed

#### Offline Detection
- WebSocket disconnects show "Connecting..." status
- Page automatically reconnects when WiFi restored
- Status values show "--" when disconnected

#### TX Lockout
- Preset changes blocked when forward power ≥5W
- Prevents retuning during transmission
- Error message displayed: "Cannot change preset while transmitting"
- Motor control (Motors page) not affected by lockout

### Troubleshooting Web Interface

| Problem | Solution |
|---------|----------|
| Cannot access web interface | Check WiFi connection, try IP address instead of `.local` |
| Page loads but no data | Check WebSocket connection, verify device is powered |
| Status not updating | Refresh page, check WiFi signal strength |
| Preset won't load | Check if transmitting (≥5W), verify motors are homed |
| mDNS not resolving | Use IP address directly, check router mDNS support |
| Buttons unresponsive | Check JavaScript enabled, try different browser |
| Multiple tabs interfere | No interference - all tabs receive synchronized updates |

### REST API

For programmatic access, StayTuned provides a REST API. See **[REST_API.md](REST_API.md)** for complete endpoint documentation with request/response examples.

**Quick Reference**:
- `GET /api/status` - Get current system status
- `GET /api/presets` - List all presets
- `POST /api/preset/load` - Load a preset
- `POST /api/preset/save` - Save current position
- `POST /api/motor` - Move motor by steps
- `POST /api/home` - Home motors
- `POST /api/stop` - Emergency stop

**Authentication**: None (relies on network security)

**Rate Limiting**: None (use responsibly)

### Remote Serial Debugging (Advanced)

Gated by `WIFI_ENABLED` in `include/config.h` (default `true`) — the telnet debug server starts automatically once WiFi connects. Note that the RF meter PCB repurposes the USB TX/RX pins for I2C, so telnet is the *only* way to see debug output while it's connected.

1. **Connect** to same WiFi network
2. **Open telnet**: `telnet staytuned.local 3232`
3. **View** serial output remotely
4. **Send** serial commands
5. **Multiple clients** supported (up to 4)

**Use cases**:
- Remote troubleshooting
- Embedded installation monitoring
- Development/debugging

### Disabling WiFi

**Why**: Slightly faster boot, reduced power consumption, no RF interference

**Method**:
1. ⚙ → Settings → Scroll down
2. Tap **"WiFi Toggle"**
3. Message: "WiFi disabled on next boot"
4. **Reboot** device
5. WiFi startup skipped

**Re-enabling**: Repeat process (button toggles state)

### Forgetting WiFi Credentials

**Purpose**: Clear saved network to reconfigure

**Method**:
1. ⚙ → Settings → Scroll down
2. Tap **"Forget WiFi"**
3. **Confirmation dialog**: "YES" / "NO"
4. Tap **"YES"** to confirm
5. Message: "WiFi Forgotten - Reboot to reconfigure"
6. **Reboot** device
7. AP mode starts (first-time setup)

**Use cases**:
- Changing WiFi networks
- Moving device to new location
- Troubleshooting connection issues

---

## Appendix A: Hardware Connections

### Display & Touch
- **Display**: ILI9341 (320×240, SPI)
- **Touch**: XPT2046 (resistive, SPI)
- **Backlight**: GPIO 17 (PWM controlled)

### Motors
- **Motor 1 (Capacitor)**: Step=GPIO25, Dir=GPIO26, Enable=GPIO27, Limit=GPIO32
- **Motor 2 (Inductor)**: Step=GPIO12, Dir=GPIO13, Enable=GPIO14, Limit=GPIO33
- **Drivers**: TB6600 (opto-isolated)

### RF Meter (Primary SWR Source)
- **Interface**: I2C, address `0x30`
- **Pins**: SDA=GPIO1 (repurposed UART0 TX), SCL=GPIO3 (repurposed UART0 RX) — USB serial is unavailable while the meter is connected; use telnet debugging instead
- **Provides**: Forward/reflected power (W), SWR (×10), frequency (Hz) — read as a 14-byte register burst at ~20 Hz (50ms)

### Legacy SWR Bridge (disabled by default, `ADC_ENABLED false`)
- **Forward Power**: GPIO 36 (ADC1_CH0, VP)
- **Reflected Power**: GPIO 39 (ADC1_CH3, VN)
- **Voltage Range**: 0-3.3V

### Audio/Indicators
- **Speaker**: GPIO 2 (PWM)
- **Status LED**: GPIO 5 (optional)

### SD Card (Optional)
- **CS**: GPIO 22
- **Shared SPI**: MOSI=23, MISO=19, SCK=18

### Power
- **ESP32**: 5V (via USB or regulator)
- **Display**: 5V (typically from ESP32 board)
- **Motors**: 12-24V (separate supply)

---

## Appendix B: Quick Reference Card

### Screen Navigation
- **Main**: Default screen, tuning and presets
- **Presets**: ⚙ → Presets
- **Settings**: ⚙ → Settings
- **SD Card**: ⚙ → SD Card
- **Calibration**: Settings → Hardware Calibration
- **Back**: Use "Back" button on each screen

### Emergency Actions
- **Stop Motors**: Manual Mode → [STOP] button
- **Re-Home Motors**: Power-cycle the device (homing runs automatically at boot), or send the `h` serial command
- **Disable Alarm**: Switch to Manual Mode
- **Forget WiFi**: Settings → "Forget WiFi"
- **Factory Reset**: Not available (manually reflash firmware)

### Serial Commands (via USB)
Connect at 115200 baud:
- `s` - Print system status
- `m1+` / `m1-` - Move motor 1
- `m2+` / `m2-` - Move motor 2
- `h` - Home motors
- `p` - List presets
- `b<value>` - Set brightness (e.g., `b200`)
- `+` / `-` - Increase/decrease brightness
- `a` - Manual alarm test (if `DEBUG_ALARM` enabled)

### Default Values
- **Brightness**: 200 (range 20-255)
- **SWR Alarm**: 2.0
- **SWR Good**: 1.5 (green/yellow)
- **SWR Caution**: 2.0 (yellow/red)
- **Motor Speed**: 2000 steps/sec
- **Motor Accel**: 1000 steps/sec²
- **M1 Homing Speed**: 2000 steps/sec
- **M2 Homing Speed**: 2000 steps/sec
- **M1 Homing Offset**: 100 steps
- **M2 Homing Offset**: 100 steps
- **M1 Homing Inverted**: false
- **M2 Homing Inverted**: false
- **MC Backlash**: 0 steps (off)
- **ML Backlash**: 0 steps (off)
- **Motor End**: 30,000 steps
- **Step Size**: 1000 (manual mode)

---

## Appendix C: Troubleshooting

### Display Issues

| Problem | Solution |
|---------|----------|
| Blank screen | Check 5V power, backlight connection (GPIO 17) |
| Dim display | Settings → Brightness → Increase value |
| Touch unresponsive | Settings → Touch Calibration |
| Touch offset | Redo touch calibration, use stylus for accuracy |
| Screen flickers | Check SPI connections, lower SPI speed if modified |

### Motor Issues

| Problem | Solution |
|---------|----------|
| Motors don't move | Check enable pins, TB6600 power, 12-24V supply |
| Homing fails | Verify limit switches (GPIO 32/33), check wiring polarity |
| Motors move wrong direction | Settings → Hardware Cal → Re-home motors |
| Erratic movement | Check TB6600 current setting, mechanical binding |
| Position drift | Home motors periodically, check for missed steps |
| First move after a direction change is shorter than the next | Load-induced microstep lag: set MC/ML Backlash in Settings (see [Backlash Compensation](#backlash-compensation-mc--ml-independent)), or raise the TB6600 current |
| First step after a direction change goes the wrong way | Update to current firmware (adds the TB6600 DIR setup time); check DIR wiring |

### SWR Reading Issues

| Problem | Solution |
|---------|----------|
| SWR always 1.0 / reads 0W | Verify the RF meter PCB acknowledges on I2C (address 0x30); check SDA=GPIO1/SCL=GPIO3 wiring. If using the legacy ADC bridge instead, confirm `ADC_ENABLED true` and check GPIO 36/39 wiring |
| SWR too high | Check RF connections; if on the legacy ADC path, verify coupling factor in `config.h` |
| SWR fluctuates | On the legacy ADC path: add capacitors to ADC inputs, increase `ADC_SAMPLES` |
| False alarms | Increase alarm threshold; verify RF meter/SWR bridge calibration |

### WiFi Issues

| Problem | Solution |
|---------|----------|
| AP not appearing | Check WiFi not disabled in settings, reboot device |
| Reboots about every 30 s with WiFi disabled | Fixed in current firmware; update. Older builds tried to reconnect WiFi in the background even when disabled |
| Can't connect to AP | Verify password "staytuned2026", check WiFi band (2.4GHz only) |
| Loses connection | Check router signal, increase retry count in `config.h` |
| OTA not accessible | Verify `staytuned.local` resolves, try IP address directly |

### Preset Issues

| Problem | Solution |
|---------|----------|
| Can't save preset | Check flash memory not full (max 50), try deleting old presets |
| Preset doesn't load | Verify motors are homed, check positions are within range |
| Button assignment lost | Re-assign in Preset List, ensure assignment saved |
| Preset loads wrong position | Delete and re-save, verify motors don't skip steps |

---

## Appendix D: Specifications

### System Specifications
- **Processor**: ESP32 (240MHz dual-core)
- **Memory**: 520KB RAM, 4MB Flash
- **Display**: 2.8" TFT (320×240), ILI9341 driver
- **Touch**: XPT2046 resistive (4-wire)
- **Storage**: Internal flash + optional SD card (FAT32)
- **WiFi**: 802.11 b/g/n (2.4GHz)
- **OTA**: ArduinoOTA (espota protocol), pushed via PlatformIO CLI — no browser upload page

### Motor Specifications
- **Type**: Stepper motors (NEMA 17/23 compatible)
- **Drivers**: TB6600 (opto-isolated)
- **Max Speed**: 4000 steps/second
- **Max Acceleration**: 2000 steps/sec²
- **Position Range**: 0-50,000 steps (configurable)
- **Homing**: Limit switches (active LOW)

### SWR Monitor
**Primary (I2C RF Meter PCB)**:
- **Interface**: I2C, address `0x30`, 100 kHz bus
- **Update Rate**: ~20 Hz (50ms)
- **Provides**: Forward/reflected power (0-1000W), SWR ×10, frequency (Hz) — no on-device calibration needed

**Legacy (analog ADC bridge, disabled by default)**:
- **ADC Resolution**: 12-bit (0-4095)
- **Sample Rate**: ~20 Hz (50ms update)
- **Oversampling**: 32 samples averaged
- **Coupling Factor**: Configurable (default 100:1)

**SWR Range** (either path): 1.0-10.0+ (practical limit ~5.0)

### Power Requirements
- **ESP32 + Display**: ~300mA @ 5V
- **Motors**: Variable, depends on load (typically 1-2A per motor @ 12-24V)
- **Total System**: ~5-10W typical operation

### Environmental
- **Operating Temp**: 0-50°C (limited by display)
- **Storage Temp**: -20-70°C
- **Humidity**: 20-80% RH (non-condensing)

---

## Appendix E: Firmware Information

- **Version**: 1.1.23
- **Build Platform**: PlatformIO
- **Framework**: Arduino (ESP32)
- **Key Libraries**:
  - TFT_eSPI (display driver)
  - TFT_eWidget (button widgets)
  - ThingPulse XPT2046 Touch (touch controller)
  - AccelStepper (motor control)
  - ESPAsyncWiFiManager (WiFi configuration portal)
  - ESPAsyncWebServer / AsyncTCP (web UI, REST API, WebSocket)
  - ArduinoJson (preset/settings storage)
  - ArduinoOTA (OTA firmware updates — built into the ESP32 core, not a separate lib_deps entry)

### Configuration Files
- **`platformio.ini`**: Build configuration, pin assignments
- **`include/config.h`**: System parameters, pin definitions
- **`include/TFT_eSPI_Setup.h`**: Display configuration reference

### Customization
All system parameters can be modified in `config.h`:
- Pin assignments
- Motor limits and speeds
- SWR thresholds and coupling factor
- Display brightness range
- WiFi credentials and OTA settings
- Serial debugging options

---

## Support & Additional Information

### Documentation
- **README.md**: Project overview and status
- **REQUIREMENTS.md**: Detailed feature specifications
- **REQUIREMENTS_COMPLIANCE.md**: Implementation checklist
- **REST_API.md**: REST API & WebSocket reference with examples
- **AUTO_TUNE_REQUIREMENTS.md**: Auto-tune algorithm strategy and constraints
- **QUICK_START.md**: Quick reference for common operations
- **SCREEN_MOCKUPS.md**: UI layout reference

### Development
- **Repository Structure**:
  - `src/` - Main application code
  - `include/` - Header files
  - `extra_script.py` - Build automation

### Known Limitations / Future Enhancements
Auto-tune, the web interface, and REST API are already implemented (see [Auto-Tune](#auto-tune) and [Web Interface](#web-interface)). What's still outstanding:
- Graphical UI elements — SWR meter arc widget, power bars, preset scatter-plot visualization (text-only display today; backend data already exists)
- Data logging / tuning history — no persistent history or analytics yet
- System self-test routine, power management/sleep modes
- Long-term stability / full integration testing (see [REQUIREMENTS_COMPLIANCE.md](REQUIREMENTS_COMPLIANCE.md))

---

**End of User Guide**

*Last Updated: August 2026*  
*Document Version: 1.3*  
*Firmware Version: 1.1.16*
