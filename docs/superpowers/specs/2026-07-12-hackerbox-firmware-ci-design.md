# HackerBox #0111 Meshtastic Firmware CI — Design

## Background

HackerBox #0111 ("Relay") bundles two Meshtastic-capable LoRa boards, each paired
with a Ra-01SH (SX1262) module:

- **ESP32-C3 OLED Kit** — PlatformIO env `hackerboxes-esp32c3-oled`, platform `esp32c3`
- **ESP32 I/O Kit** — PlatformIO env `hackerboxes-esp32-io`, platform `esp32`

Both are already merged, maintainer-reviewed board variants in `meshtastic/firmware`
(`variants/esp32c3/hackerboxes_esp32c3_oled`, `variants/esp32/hackerboxes_esp32_io`,
added in [PR #6319](https://github.com/meshtastic/firmware/pull/6319)). A reviewer
caught a real wiring bug pre-merge (missing `SX126X_DIO2_AS_RF_SWITCH`) and the
contributor fixed and re-verified it on hardware, and the configs have received at
least one further correction since (a `platformio.ini` base-flags fix). They are
accurate and hardware-tested as they stand today.

Both variants are marked `board_level = extra` in their `platformio.ini`. This
excludes them from meshtastic/firmware's own CI, nightly builds, and — confirmed by
diffing the latest Beta release's manifest (129 targets, neither board present) and
its `firmware-esp32*.zip` bundles — from every official release. The only way to
build them today is a manual, one-off `workflow_dispatch` of meshtastic's internal
`build_one_target.yml`, whose output is a 90-day CI artifact, never a release asset.
**No official prebuilt binary exists anywhere for either board.**

### OLED display (explicitly out of scope)

The ESP32-C3 board's 0.42" (72x40) OLED does not work today and both variants ship
`HAS_SCREEN 0` (headless). Confirmed this is not a config gap:

- Meshtastic's I2C display path uses a fixed `OLEDDISPLAY_GEOMETRY` enum
  (`128_64`, `128_32`, `64_48`, `64_32`, `128_128`, `RAWMODE`-for-SPI-only) with no
  72x40 entry. The `USE_SSD1306_72_40` / `TFT_WIDTH 72` / `TFT_HEIGHT 40` lines
  already present (commented out) in `hackerboxes_esp32c3_oled/variant.h` are
  unread by any driver code — dead placeholders.
- No other meshtastic/firmware variant has a working tiny-OLED config, and no
  open issue/PR proposes adding one. [Issue #6290](https://github.com/meshtastic/firmware/issues/6290)
  (same board) was closed by its author as "not working yet."
- Real support would mean adding a new geometry to the display library fork,
  wiring it into `Screen.cpp`/`main.cpp`, and likely reworking the UI layout code
  (72x40 is ~1/8 the pixel area Meshtastic's screen frames assume) — a driver/UI
  feature contribution to meshtastic/firmware, not something this project builds
  or unblocks. Both boards are shipped and built headless.

## Prior art

- `cjcase/nibble-esp32-firmware` — closest precedent: `git ls-remote --tags` to
  find the latest meshtastic/firmware tag, checkout, raw `pio run`, `gh release
  create` in its own repo. Validates the overall shape. Gaps this design closes:
  no schedule (manual-only), no Alpha/Beta distinction (tag-sort catches
  everything), no OTA-bundle/device-install-script parity with official releases.
- `skgsergio/meshtastic-firmware-builder` — copies its own out-of-tree `variants/`
  folder into the checkout before building. Not needed here since both boards are
  already merged upstream; confirms building straight from the checkout is correct
  for our case.
- `MeshEnvy/mesh-forge`, `Crank-Git/MTFWBuilder` — full hosted build-on-demand web
  services (Convex/Cloudflare app; Flask+Docker GUI) supporting many boards
  including both HackerBox variants. Related need, much larger scope (backend +
  UI) — out of scope for this project, which is a single CI workflow.

## Goal

A GitHub Actions workflow, in this repo, that automatically builds firmware for
both HackerBox boards whenever meshtastic/firmware cuts a new **Beta** release,
and publishes the results as a GitHub Release in this repo.

## Design

### Trigger

- **Scheduled poll, daily.** meshtastic/firmware is not this repo, so there is no
  webhook option — a `schedule` cron is the only way to detect new upstream
  releases. Cadence: Beta releases land roughly every few months, so daily is
  ample margin without wasted runs.
- Each run calls `GET /repos/meshtastic/firmware/releases/latest` — this endpoint
  returns exactly the newest **non-prerelease, non-draft** release, which is
  precisely meshtastic's "Beta" (confirmed: `prerelease:false` on Beta releases,
  `prerelease:true` on Alpha releases via the API).
- **Idempotency / dedup:** before building, check whether a release already exists
  in this repo tagged with the same version. If yes, skip (no-op exit). This makes
  the daily poll safe to run indefinitely and also safe to re-run manually.
- `workflow_dispatch` with an optional `tag` input allows manually building any
  upstream tag (including Alphas) on demand, for testing — independent of the
  scheduled Beta-only path.

### Build (matrix over the two boards)

For each of `{ env: hackerboxes-esp32-io, platform: esp32 }` and
`{ env: hackerboxes-esp32c3-oled, platform: esp32c3 }`:

1. Checkout `meshtastic/firmware` at the target tag, `submodules: recursive`.
2. Build via `meshtastic/gh-action-firmware@main` (`pio_platform`, `pio_env`,
   `pio_target: build`) — reuses meshtastic's own build action rather than
   reimplementing PlatformIO invocation, for accuracy and to track their build
   environment automatically.
3. Download and merge the matching OTA companion binary into the manifest, same
   as upstream `build_firmware.yml`:
   - `esp32` → unified OTA from `meshtastic/esp32-unified-ota`
   - `esp32c3` → BLE-only OTA from `meshtastic/firmware-ota`
4. Upload the board's `.bin`/`.uf2`/`.mt.json`/device-install scripts as a
   workflow artifact.

### Publish

A final job downloads both boards' artifacts, zips each board's output
(`firmware-<board>-<version>.zip`, matching upstream's convention), and creates a
GitHub Release in this repo:

- Tag: `v<version>` — same version string as the upstream tag.
- Body: short note plus a link back to the upstream release/release notes.
- Assets: both zips attached.

### Repo layout

```
.github/workflows/build-hackerbox-firmware.yml   # schedule + workflow_dispatch, build matrix, publish
README.md                                         # hardware background, board links, flashing pointer, how to trigger a manual build
docs/superpowers/specs/...                        # this doc
```

No vendored firmware source, no variant files of our own — both boards build
straight from the meshtastic/firmware checkout since their variants are already
upstream.

## Error handling

- **Duplicate tag:** skip cleanly (see Idempotency above), not a failure.
- **Env name stability:** `hackerboxes-esp32-io` / `hackerboxes-esp32c3-oled` are
  hardcoded. PlatformIO resolves envs by name, not path, so these survived the one
  directory reorg (`variants/hackerboxes_x` → `variants/esp32/hackerboxes_x`) that
  has happened in this project's history so far.
- **If a board is renamed/removed upstream:** the build step fails loudly
  (PlatformIO "unknown environment" in the job log/summary). Not auto-healing —
  acceptable given how rarely this has changed historically; a human fixes the
  hardcoded env name.
- **No secrets required** beyond the default `GITHUB_TOKEN` (`contents: write`,
  for creating the release in this repo). Reading meshtastic/firmware needs no
  auth (public repo).
- **Mutable refs are a deliberate trade-off, not an oversight.** Pinning
  `meshtastic/gh-action-firmware@main` to a mutable branch (rather than a SHA or
  version tag), and downloading OTA companion binaries from each OTA repo's
  `.../releases/latest/...` (rather than a tag-matched release), are both
  intentional choices consistent with this project's "track upstream
  automatically" goal: they mean the build action and OTA binaries always match
  meshtastic's current tooling/firmware without this workflow needing manual
  bumps. The accepted cost is reduced reproducibility — rebuilding an old tag
  later may not byte-for-byte reproduce what was originally published, since
  `@main` and `latest` can have moved on. That's acceptable here because this
  project's whole purpose is following upstream forward, not archiving exact
  historical builds.

## Testing / verification plan

Trigger `workflow_dispatch` with `tag: v2.7.26.54e0d8d` (current latest Beta at
design time) once the workflow is written. Confirm:

- Both boards build successfully.
- OTA companion binaries download and merge into each manifest.
- A release `v2.7.26.54e0d8d` appears in this repo with two correctly-named zips.
- Re-running the same trigger is a no-op (dedup works).

Then leave the daily schedule running and confirm the next real upstream Beta
release is picked up within 24 hours.

## Explicitly out of scope

- Making the 0.42" OLED display functional (see above) — headless is correct,
  upstream-matching behavior for both boards.
- A hosted/web build-on-demand service (that's what mesh-forge/MTFWBuilder are
  for) — this project is CI automation only.
- Building Alpha releases on a schedule — only available via the manual
  `workflow_dispatch` `tag` override.
