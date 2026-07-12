# HackerBox #0111 — Meshtastic Firmware CI

Unofficial, automated firmware builds of [Meshtastic](https://meshtastic.org) for
the two LoRa boards included in [HackerBox #0111 "Relay"](https://hackerboxes.com/collections/past-hackerboxes/products/hackerbox-0111-relay):

| Board | PlatformIO env | Meshtastic variant |
| --- | --- | --- |
| ESP32-C3 OLED Kit (ESP32-C3 + 0.42" OLED + Ra-01SH/SX1262) | `hackerboxes-esp32c3-oled` | [`variants/esp32c3/hackerboxes_esp32c3_oled`](https://github.com/meshtastic/firmware/tree/develop/variants/esp32c3/hackerboxes_esp32c3_oled) |
| ESP32 I/O Kit (ESP-WROOM-32 + Ra-01SH/SX1262) | `hackerboxes-esp32-io` | [`variants/esp32/hackerboxes_esp32_io`](https://github.com/meshtastic/firmware/tree/develop/variants/esp32/hackerboxes_esp32_io) |

Both variants are merged into `meshtastic/firmware` (added in
[PR #6319](https://github.com/meshtastic/firmware/pull/6319)) but are marked
`board_level = extra`, which excludes them from meshtastic's own CI, nightly, and
release builds. **No official prebuilt binary exists for either board** — this repo
fills that gap.

## What this does

A daily scheduled GitHub Actions workflow
(`.github/workflows/build-hackerbox-firmware.yml`) checks
[meshtastic/firmware's latest Beta release](https://github.com/meshtastic/firmware/releases/latest).
If it's new (no matching release already exists here), it builds both boards from
that exact tag — reusing meshtastic's own `meshtastic/gh-action-firmware` build
action and OTA-bundling steps for parity with official releases — and publishes a
GitHub Release here tagged with the same version, e.g. `v2.7.26.54e0d8d`, with a
firmware zip for each board attached.

## Manually building a specific version

Trigger the workflow by hand with an explicit upstream tag (Alpha or Beta):

```
gh workflow run build-hackerbox-firmware.yml -f tag=v2.7.26.54e0d8d
```

Leave `tag` blank to build whatever the current latest Beta is.

## Flashing

Each release zip contains the same `.bin`/`.uf2` files and `device-install`/`device-update`
scripts meshtastic ships in its own releases. Follow meshtastic's
[Flashing Firmware](https://meshtastic.org/docs/getting-started/flashing-firmware/) guide,
substituting the files from this repo's release instead of the official one.

## The OLED screen doesn't work

Both boards ship headless (`HAS_SCREEN 0`) — this matches upstream, not a bug in this
repo's build. Meshtastic's display driver has no support for the ESP32-C3 board's
tiny 72x40 OLED panel (its geometry enum is a fixed set: 128x64/128x32/64x48/64x32/128x128/RAWMODE-SPI-only);
adding it would be a real driver/UI feature contribution to meshtastic/firmware, not
something this build pipeline can unlock. See
[docs/superpowers/specs/2026-07-12-hackerbox-firmware-ci-design.md](docs/superpowers/specs/2026-07-12-hackerbox-firmware-ci-design.md)
for the full research trail.
