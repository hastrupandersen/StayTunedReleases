# StayTuned Firmware Releases

Precompiled firmware binaries for **StayTuned**, an ESP32-based automatic
balanced antenna tuner controller. This repository publishes ready-to-flash
`.bin` builds; project source code is maintained separately.

## Downloading

Each version lives in its own folder under
[`releases/`](releases/), named after its tag (e.g. `releases/v1.1.21/`),
containing:

- `firmware-X.Y.Z.bin` — the firmware binary
- `USER_GUIDE.md` — the user guide matching that version
- `RELEASE_NOTES.md` — what changed in that version

The newest folder (highest version number) is the latest release.

## Installing

1. Copy the downloaded `.bin` file onto the root of the device's SD card.
2. On the device, open the gear menu → **About** → **Update Firmware from SD**.
3. Select the file from the list and confirm — the device flashes it and
   reboots automatically. No USB cable or WiFi connection is required.

## License

These firmware binaries are licensed under **CC BY-NC-ND 4.0**
(Attribution-NonCommercial-NoDerivatives 4.0 International) — see
[LICENSE](LICENSE).

In short: free to download and use for personal, non-commercial purposes.
Modifying, reverse engineering, repackaging, or redistributing altered
versions of the binaries is not permitted, and commercial use requires
separate permission from the author.
