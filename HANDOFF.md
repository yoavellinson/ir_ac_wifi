# Handoff: WiFi remote for a TCL AC using an ESP32 IR board

## Goal
Turn the user's ESP32 IR transceiver board into a WiFi remote for their TCL air conditioner, using ESPHome (`climate: platform: tcl112`), ideally exposed to Home Assistant.

## Hardware
- **Target board:** "ESP8266/ESP32 Infrared Transmitting And Receiving NEC Command WIFI Transparent Transmission Module" (Shenzhen hua jie micro technology). PCB marking `ESP32_IR_TR_WIFI_V1.1 (303ESP32IR1)`, dated 20251115. Module: DOIT ESP32-32E N4 (4 MB flash).
- Onboard: IR LED (IR_TX), IR receiver (IR_RX), USB-C, 6-pin header.
- Header order (left to right): `GND, 5V, RX, TX, IO0, GND`. No EN/RST pin.
- **USB-C is almost certainly power-only**: no USB-serial chip visible in photos. Confirm by plugging into a computer; if no new serial port appears, flash via the header.
- **Programmer:** the user has a second ESP32 dev board (USB-serial chip next to micro-USB, RST and BOOT buttons, pins labeled EN, V5, GND, TXD, RXD, 3V3). Use it as a USB-TTL adapter.

## Pin mapping (UNVERIFIED)
Taken from a Home Assistant forum post about a near-identical "ESP32E-N4 Infrared Transmitter Receiver NEC" board:
- IR TX: GPIO4
- IR RX: GPIO14, inverted (many guides say GPIO5, which that poster found wrong)
- GPIO21: must be HIGH to power the IR parts

Source: https://community.home-assistant.io/t/solution-esp32-infrared-wifi-module-esp32e-n4-infrared-transmitter-receiver-correct-pinout-for-esphome/1003067

This may not match the user's revision. Verify before trusting it (see Testing).

## Wiring for first flash (dev board as USB-TTL)
Dev board labels are from its own ESP32's point of view; the connections below are straight, label to label.

| Dev board pin | IR board header |
|---|---|
| EN | GND (on the dev board itself, holds its ESP32 in reset) |
| GND | GND |
| V5 | 5V |
| RXD | RX |
| TXD | TX |

Plus: jumper the IR board's **IO0 to its adjacent GND**. This must be in place before power is applied.
Never connect 3V3 to the IR board's 5V pin. Power only from V5 during flashing (not also via USB-C).

## Steps for Claude Code
1. Check/install ESPHome (`pip install esphome`, ideally in a venv).
2. Create `secrets.yaml` next to `tcl-ac.yaml` with `wifi_ssid` and `wifi_password`. **Ask the user for these; do not guess.**
3. Run `esphome config tcl-ac.yaml`, then `esphome compile tcl-ac.yaml`.
4. Ask the user to wire the board as above and plug in the dev board. Confirm they have done it before flashing (hardware steps need the user).
5. `esphome run tcl-ac.yaml`, selecting the dev board's serial port. If the serial port isn't listed, a CP2102/CH340 driver may be missing.
6. After "Successfully uploaded": user removes both jumpers (EN to GND, IO0 to GND), unplugs, and powers the IR board via its USB-C. Subsequent updates go over WiFi (`esphome run tcl-ac.yaml`, choose OTA).
7. Watch logs (`esphome logs tcl-ac.yaml`) and run the tests below.

## Testing
- **Receiver:** point the original TCL remote at the board and press a button. `dump: all` should log the decoded protocol. If nothing appears on GPIO14, try GPIO5 (not inverted and inverted).
- **Transmitter:** open a phone camera, trigger a command from Home Assistant or the ESPHome web page, and look for the IR LED glowing purple. If it doesn't, check the GPIO21 power switch and try other TX pins.
- **AC response:** point the LED at the AC's receiver with line of sight. If `tcl112` doesn't control the unit, check the protocol the dump reports from the original remote and try another climate platform (for example `coolix`).

## Failure modes
- "Failed to connect" / "Wrong boot mode": IO0 must be grounded before power-up; replug with the jumper on. Then try swapping RX/TX. Use a data-capable cable.
- Board boots but WiFi never joins: recheck `secrets.yaml`, and that the network is 2.4 GHz.

## Status
Nothing has been flashed yet. The pin mapping is untested.
- `tcl-ac.yaml` passes `esphome config` (ESPHome 2026.9.1). Changes from the original: added `id: ir_power` to the GPIO21 switch (it was failing validation without one), a fallback AP plus captive portal, and `web_server` on port 80.
- `esphome compile` has not run yet: the cloud session's network policy blocks `dl.espressif.com`. Compile on your local machine.
- Copy `secrets.yaml.example` to `secrets.yaml` and fill it in. `secrets.yaml` is gitignored.
