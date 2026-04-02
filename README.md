# ESP32-S3 SuperMini Doorbell Camera Node

[![ESPHome](https://img.shields.io/badge/ESPHome-2025-blue?logo=esphome)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Integration-41BDF5?logo=home-assistant)](https://www.home-assistant.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-ffdd00?logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/ay129)
[![Donate](https://img.shields.io/badge/Donate-PayPal-blue.svg)](https://www.paypal.com/donate?business=nyashachipanga%40yahoo.com&currency_code=GBP)

Built with an ESP32-S3 SuperMini, a separate AliExpress display + encoder module, ESPHome, Home Assistant, PSRAM, and a Linux image server.

<p align="center">
  <img src="https://raw.githubusercontent.com/ay129-35MR/esp32-doorbell-viewer/main/IMG_7200.jpeg" width="380" alt="Clock mode">
  <img src="https://raw.githubusercontent.com/ay129-35MR/esp32-doorbell-viewer/main/IMG_7196.jpeg" width="380" alt="Camera mode">
</p>
<p align="center">
  <img src="https://raw.githubusercontent.com/ay129-35MR/esp32-doorbell-viewer/main/IMG_7089.jpeg" width="250" alt="Perfboard build front">
  <img src="https://raw.githubusercontent.com/ay129-35MR/esp32-doorbell-viewer/main/IMG_7189.jpeg" width="250" alt="Perfboard build rear">
  <img src="https://raw.githubusercontent.com/ay129-35MR/esp32-doorbell-viewer/main/IMG_7192.jpeg" width="250" alt="Desk setup">
</p>

### Overview

This is a small desktop doorbell camera node with two jobs:

- show a clean clock/status screen when idle
- jump to a camera image when Home Assistant reports motion or a person outside

The ESP32-S3 SuperMini is the controller, not the display. The screen and rotary encoder live on a separate module, connected through a custom perfboard motherboard with 2.54 mm female headers. One side takes the display/encoder assembly, the reverse takes the SuperMini.

The result is a compact device that:

- shows time and date when idle
- indicates when someone is outside
- switches to a front camera image on motion/person detection
- lets you swap between front and back/garden cameras with the encoder
- fetches prepared snapshots from a Linux server
- uses PSRAM to keep the display workflow comfortable

### Hardware

**Parts**

- **Controller:** ESP32-S3 SuperMini
- **Display / controls:** AliExpress display + rotary encoder module
- **Front panel module:** <https://www.aliexpress.com/item/1005009880904568.html>
- **Motherboard:** custom perfboard carrier with female 2.54 mm headers

In software, the display is driven as an **ST7789V 240x320 SPI panel**.

**Perfboard build**

I built the motherboard on a bit of perfboard rather than starting with a PCB. That made it easy to prototype the wiring and mechanical layout:

- display + encoder assembly plugs into one side
- ESP32-S3 SuperMini plugs into the reverse
- point-to-point wiring joins the two

It works well, but it still looks like a prototype. The USB cable currently exits awkwardly at the top because of the routing I chose during soldering. A 90-degree USB cable would improve it immediately, and a v2 PCB would fix it properly.

### Pinout

| Function | GPIO | Notes |
|---|---:|---|
| TFT Backlight PWM | `GPIO5` | Controlled via `ledc` |
| TFT SPI Clock | `GPIO12` | Shared display SPI bus |
| TFT SPI MOSI | `GPIO13` | Shared display SPI bus |
| TFT CS | `GPIO10` | ST7789 chip select |
| TFT DC | `GPIO9` | Display data/command |
| TFT Reset | `GPIO8` | Display reset |
| Rotary Encoder A | `GPIO6` | `INPUT_PULLUP` |
| Rotary Encoder B | `GPIO4` | `INPUT_PULLUP` |
| Navigation Button | `GPIO2` | `INPUT_PULLUP`, inverted |
| Knob Push Button | `GPIO7` | `INPUT_PULLUP`, inverted |

### How it works

The device has two main pages:

**Clock page**

The idle page shows:

- time
- date
- reduced brightness at night
- a visual indicator when someone is outside

**Camera page**

When triggered by Home Assistant or local input, the device switches to a full-screen snapshot page. That page:

- loads a fresh image for the selected camera
- overlays front/back selection
- overlays a timestamp
- supports manual refresh
- returns to the clock after inactivity

### Home Assistant

Home Assistant is the event layer that makes the device useful.

It provides:

- time sync
- motion/person detection state
- event-driven automation

In my setup, a Home Assistant binary sensor for the front camera area triggers the main behavior. When it fires, the node:

- switches to the camera page
- selects the front camera
- raises brightness
- fetches a fresh image

On the clock page, that same signal is also used to show that someone is outside.

### Linux server

The ESP does not handle full camera streams directly. That work is pushed onto a Linux server, which exposes ready-to-display snapshot endpoints like:

```text
http://<linux-server>:5051/image/front_fluent?width=240&height=320&format=png&fit=cover
http://<linux-server>:5051/image/garden_fluent?width=240&height=320&format=png&fit=cover
```

The server handles:

- upstream camera access
- resizing
- crop-to-fit
- PNG conversion
- stable URLs for the ESP

See the implementation of the server at https://github.com/ay129-35MR/Waveshare-ESP32-S3-Touch-LCD-4-Home-Assistant-Display  which is based on the excellent work of @strange-v

That leaves the ESP32-S3 to do the embedded-friendly part: fetch the image, render the UI, react to the encoder/buttons, and stay responsive.

### Why PSRAM matters

PSRAM is one of the reasons this project feels practical rather than fragile.

In ESPHome I enable:

```yaml
psram:
  mode: quad
  speed: 40MHz
```

That extra memory headroom helps with:

- display-heavy rendering
- image handling
- temporary buffers
- remote screenshot/debugging support

I also use a custom ESPHome screenshot component that captures the current framebuffer and serves it over HTTP as a BMP. That development workflow is much more comfortable once PSRAM is available.

### Why this architecture works

This split has worked well:

- **Linux server** handles image processing
- **Home Assistant** handles motion/person events and timing
- **ESP32-S3** handles local UI and interaction

That is a much better fit than trying to make the microcontroller act like a media client or analytics engine.

### User interaction

- **Navigation button:** switch between clock and camera page
- **Rotary encoder:** swap between front and back/garden cameras
- **Encoder press:** refresh the selected image

The interaction is intentionally simple so the device behaves like a small appliance rather than a menu-driven gadget.

### Why it has held up

A lot of small ESP display projects are fun for a week and then get abandoned. This one stuck because it solves a narrow problem well :

- always-visible clock/status node
- immediate camera view when motion or a person is detected <cough> Jehovah's witnesses <cough>
- local physical controls
- remote screenshot/debugging
- clean separation between ESP, Home Assistant, and the Linux server

### Next steps

- replace the perfboard with a proper PCB
- fix USB cable orientation cleanly
- improve enclosure and cable exit
- refine the front-panel mounting

The current build already proves the idea. v2 just needs to make the hardware cleaner.
