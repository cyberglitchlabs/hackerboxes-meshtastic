# HackerBox #0111 Meshtastic Firmware CI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A GitHub Actions workflow in `cyberglitchlabs/hackerboxes-meshtastic` that automatically builds Meshtastic firmware for both HackerBox #0111 boards whenever meshtastic/firmware cuts a new Beta release, and publishes the result as a GitHub Release in this repo.

**Architecture:** Three chained jobs in a single workflow file: `resolve` (figures out which upstream tag to build and whether it's already been published here), `build` (a 2-item matrix, one per board, using meshtastic's own `gh-action-firmware` build action plus their OTA-bundling steps), and `publish` (zips each board's artifact and creates a GitHub Release tagged to match the upstream version). Triggered by a daily `schedule` cron, with a `workflow_dispatch` override to build any tag on demand.

**Tech Stack:** GitHub Actions (YAML), `gh` CLI (used inside workflow steps via the default `GITHUB_TOKEN`), PlatformIO (invoked indirectly via `meshtastic/gh-action-firmware@main`).

## Global Constraints

- PlatformIO env names are fixed and must be used verbatim: `hackerboxes-esp32-io` (platform `esp32`), `hackerboxes-esp32c3-oled` (platform `esp32c3`). Source: `meshtastic/firmware` `variants/esp32/hackerboxes_esp32_io/platformio.ini` and `variants/esp32c3/hackerboxes_esp32c3_oled/platformio.ini`.
- Action versions must match what `meshtastic/firmware`'s own current workflows use (confirmed by fetching their live `.github/workflows/*.yml` during design): `actions/checkout@v7`, `actions/upload-artifact@v7`, `actions/download-artifact@v8`.
- Build step must use `meshtastic/gh-action-firmware@main` (not raw `pio run`) — this is what makes the output match official releases.
- Trigger must be `schedule` (daily) + `workflow_dispatch` with an optional `tag` string input. No other trigger types.
- Release tag in this repo must equal the upstream tag verbatim (e.g. `v2.7.26.54e0d8d`).
- No vendored firmware source and no custom `variants/` files in this repo — both boards build straight from the `meshtastic/firmware` checkout.
- Full design rationale and out-of-scope items (OLED support, hosted build service) live in `docs/superpowers/specs/2026-07-12-hackerbox-firmware-ci-design.md` — don't re-litigate them mid-implementation.

---

## File structure

```
.gitignore                                        # OS cruft only
README.md                                          # hardware background, usage, flashing pointer
.github/workflows/build-hackerbox-firmware.yml     # the whole pipeline (resolve -> build -> publish)
docs/superpowers/specs/2026-07-12-hackerbox-firmware-ci-design.md   # already committed
docs/superpowers/plans/2026-07-12-hackerbox-firmware-ci.md          # this file
```

The workflow file is built up incrementally across Tasks 2-4 (one job added per task), each independently tested against the live repo via `gh workflow run` before the next job is added.

---

### Task 1: Repo scaffold (README, .gitignore, push to GitHub)

**Files:**
- Create: `/Users/jsievert/personal/hackerboxes-meshtastic/.gitignore`
- Create: `/Users/jsievert/personal/hackerboxes-meshtastic/README.md`

**Interfaces:** None (no code yet).

- [ ] **Step 1: Create `.gitignore`**

```
.DS_Store
```

- [ ] **Step 2: Create `README.md`**

```markdown
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
```

- [ ] **Step 3: Commit and push**

```bash
git add .gitignore README.md
git commit -m "Add README and gitignore"
git push -u origin main
```

Expected: push succeeds and creates `main` on the remote (confirm with `git log origin/main --oneline -1` after push, or `gh repo view cyberglitchlabs/hackerboxes-meshtastic --json defaultBranchRef --jq .defaultBranchRef.name` returning `main`).

---

### Task 2: `resolve` job — figure out what tag to build, and whether to skip

**Files:**
- Create: `.github/workflows/build-hackerbox-firmware.yml`

**Interfaces:**
- Produces: job `resolve` with outputs `tag` (string, upstream tag to build) and `skip` (string `"true"`/`"false"`) — Task 3's `build` job and Task 4's `publish` job both read `needs.resolve.outputs.tag` and gate on `needs.resolve.outputs.skip == 'false'`.

- [ ] **Step 1: Write the workflow file with only the `resolve` job**

```yaml
name: Build HackerBox Firmware

on:
  schedule:
    - cron: "17 6 * * *"
  workflow_dispatch:
    inputs:
      tag:
        description: "meshtastic/firmware tag to build (e.g. v2.7.26.54e0d8d). Leave blank to use the latest Beta release."
        required: false
        type: string

concurrency:
  group: build-hackerbox-firmware
  cancel-in-progress: false

permissions:
  contents: write

jobs:
  resolve:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.resolve.outputs.tag }}
      skip: ${{ steps.resolve.outputs.skip }}
    steps:
      - name: Resolve target tag and check for existing release
        id: resolve
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          INPUT_TAG: ${{ inputs.tag }}
        run: |
          set -euo pipefail
          if [ -n "$INPUT_TAG" ]; then
            TAG="$INPUT_TAG"
          else
            TAG=$(gh api repos/meshtastic/firmware/releases/latest --jq .tag_name)
          fi
          echo "Resolved upstream tag: $TAG"

          if gh release view "$TAG" --repo "${{ github.repository }}" >/dev/null 2>&1; then
            echo "Release $TAG already exists in this repo; skipping build."
            echo "tag=$TAG" >> "$GITHUB_OUTPUT"
            echo "skip=true" >> "$GITHUB_OUTPUT"
          else
            echo "tag=$TAG" >> "$GITHUB_OUTPUT"
            echo "skip=false" >> "$GITHUB_OUTPUT"
          fi
```

- [ ] **Step 2: Commit and push**

```bash
git add .github/workflows/build-hackerbox-firmware.yml
git commit -m "Add resolve job: pick upstream tag and dedup against existing releases"
git push
```

- [ ] **Step 3: Trigger with an explicit tag and verify resolution**

```bash
gh workflow run build-hackerbox-firmware.yml -f tag=v2.7.26.54e0d8d
sleep 10
RUN_ID=$(gh run list --workflow build-hackerbox-firmware.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --exit-status
gh run view "$RUN_ID" --log | grep -i "Resolved upstream tag"
```

Expected: run succeeds; log contains `Resolved upstream tag: v2.7.26.54e0d8d` (substitute whatever tag you passed). Since no release with that tag exists yet in this repo, the job output `skip` is `false` — confirm with:

```bash
gh api "repos/cyberglitchlabs/hackerboxes-meshtastic/actions/runs/$RUN_ID/jobs" --jq '.jobs[0].steps[] | select(.name | contains("Resolve")) | .conclusion'
```

Expected: `success`.

- [ ] **Step 4: Trigger with no tag and verify it picks the latest Beta**

```bash
gh workflow run build-hackerbox-firmware.yml
sleep 10
RUN_ID=$(gh run list --workflow build-hackerbox-firmware.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --exit-status
LATEST=$(gh api repos/meshtastic/firmware/releases/latest --jq .tag_name)
gh run view "$RUN_ID" --log | grep -i "Resolved upstream tag: $LATEST"
```

Expected: log line present, confirming the no-input path correctly falls back to `releases/latest`.

---

### Task 3: `build` job — matrix build both boards with OTA bundling

**Files:**
- Modify: `.github/workflows/build-hackerbox-firmware.yml`

**Interfaces:**
- Consumes: `needs.resolve.outputs.tag`, `needs.resolve.outputs.skip` (from Task 2).
- Produces: workflow artifacts named `firmware-hackerboxes-esp32-io-<tag>` and `firmware-hackerboxes-esp32c3-oled-<tag>`, each containing `*.mt.json`, `*.bin`, `*.uf2`, `*.hex`, `device-*.sh`, `device-*.bat` — Task 4's `publish` job downloads these by exact name.

- [ ] **Step 1: Append the `build` job**

Add this job below `resolve` in `.github/workflows/build-hackerbox-firmware.yml` (file otherwise unchanged from Task 2):

```yaml
  build:
    needs: resolve
    if: needs.resolve.outputs.skip == 'false'
    strategy:
      fail-fast: false
      matrix:
        include:
          - board: hackerboxes-esp32-io
            platform: esp32
          - board: hackerboxes-esp32c3-oled
            platform: esp32c3
    runs-on: ubuntu-latest
    steps:
      - name: Checkout meshtastic/firmware @ ${{ needs.resolve.outputs.tag }}
        uses: actions/checkout@v7
        with:
          repository: meshtastic/firmware
          ref: ${{ needs.resolve.outputs.tag }}
          submodules: recursive

      - name: Build ${{ matrix.board }}
        uses: meshtastic/gh-action-firmware@main
        with:
          pio_platform: ${{ matrix.platform }}
          pio_env: ${{ matrix.board }}
          pio_target: build

      - name: Download unified OTA firmware (esp32)
        if: matrix.platform == 'esp32'
        working-directory: release
        run: curl -fL -o mt-esp32-ota.bin https://github.com/meshtastic/esp32-unified-ota/releases/latest/download/mt-esp32-ota.bin

      - name: Download BLE-only OTA firmware (esp32c3)
        if: matrix.platform == 'esp32c3'
        working-directory: release
        run: curl -fL -o bleota-c3.bin https://github.com/meshtastic/firmware-ota/releases/latest/download/firmware-c3.bin

      - name: Merge OTA binary into manifest
        working-directory: release
        run: |
          set -euo pipefail
          if [ "${{ matrix.platform }}" = "esp32" ]; then
            OTA_FILE="mt-esp32-ota.bin"
          else
            OTA_FILE="bleota-c3.bin"
          fi
          OTA_MD5=$(md5sum "$OTA_FILE" | cut -d' ' -f1)
          OTA_SIZE=$(stat -c%s "$OTA_FILE")
          for manifest in firmware-*.mt.json; do
            [ -f "$manifest" ] || continue
            jq --arg name "$OTA_FILE" --arg md5 "$OTA_MD5" --argjson bytes "$OTA_SIZE" --arg part "app1" \
              'if .files | map(select(.name == $name)) | length == 0 then .files += [{"name": $name, "md5": $md5, "bytes": $bytes, "part_name": $part}] else . end' \
              "$manifest" > "${manifest}.tmp" && mv "${manifest}.tmp" "$manifest"
          done

      - name: Upload board artifact
        uses: actions/upload-artifact@v7
        with:
          name: firmware-${{ matrix.board }}-${{ needs.resolve.outputs.tag }}
          path: |
            release/*.mt.json
            release/*.bin
            release/*.uf2
            release/*.hex
            release/device-*.sh
            release/device-*.bat
          if-no-files-found: error
```

- [ ] **Step 2: Commit and push**

```bash
git add .github/workflows/build-hackerbox-firmware.yml
git commit -m "Add build job: matrix-build both HackerBox boards with OTA bundling"
git push
```

- [ ] **Step 3: Trigger and verify both boards build**

```bash
LATEST=$(gh api repos/meshtastic/firmware/releases/latest --jq .tag_name)
gh workflow run build-hackerbox-firmware.yml -f tag="$LATEST"
sleep 10
RUN_ID=$(gh run list --workflow build-hackerbox-firmware.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --exit-status
```

Expected: run succeeds (this takes several minutes — two PlatformIO builds). If a build fails, `gh run watch` exits non-zero; inspect with `gh run view "$RUN_ID" --log-failed`.

- [ ] **Step 4: Verify both artifacts were uploaded**

```bash
gh api "repos/cyberglitchlabs/hackerboxes-meshtastic/actions/runs/$RUN_ID/artifacts" --jq '.artifacts[].name'
```

Expected output (two lines):
```
firmware-hackerboxes-esp32-io-<LATEST>
firmware-hackerboxes-esp32c3-oled-<LATEST>
```

---

### Task 4: `publish` job — zip and release

**Files:**
- Modify: `.github/workflows/build-hackerbox-firmware.yml`

**Interfaces:**
- Consumes: artifacts `firmware-hackerboxes-esp32-io-<tag>` and `firmware-hackerboxes-esp32c3-oled-<tag>` (from Task 3), `needs.resolve.outputs.tag` (from Task 2).
- Produces: a GitHub Release in this repo tagged `<tag>` with two assets, `firmware-hackerboxes-esp32-io-<tag>.zip` and `firmware-hackerboxes-esp32c3-oled-<tag>.zip`.

- [ ] **Step 1: Append the `publish` job**

Add below `build` in `.github/workflows/build-hackerbox-firmware.yml`:

```yaml
  publish:
    needs: [resolve, build]
    if: needs.resolve.outputs.skip == 'false'
    runs-on: ubuntu-latest
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      TAG: ${{ needs.resolve.outputs.tag }}
    steps:
      - name: Checkout this repo
        uses: actions/checkout@v7

      - name: Download hackerboxes-esp32-io artifact
        uses: actions/download-artifact@v8
        with:
          name: firmware-hackerboxes-esp32-io-${{ needs.resolve.outputs.tag }}
          path: output/hackerboxes-esp32-io

      - name: Download hackerboxes-esp32c3-oled artifact
        uses: actions/download-artifact@v8
        with:
          name: firmware-hackerboxes-esp32c3-oled-${{ needs.resolve.outputs.tag }}
          path: output/hackerboxes-esp32c3-oled

      - name: Zip firmware bundles
        run: |
          set -euo pipefail
          mkdir -p dist
          ( cd output/hackerboxes-esp32-io && zip -j -9 -r "$OLDPWD/dist/firmware-hackerboxes-esp32-io-${TAG}.zip" . )
          ( cd output/hackerboxes-esp32c3-oled && zip -j -9 -r "$OLDPWD/dist/firmware-hackerboxes-esp32c3-oled-${TAG}.zip" . )

      - name: Create release
        run: |
          set -euo pipefail
          gh release create "$TAG" \
            --repo "${{ github.repository }}" \
            --title "HackerBox #0111 Firmware ${TAG}" \
            --notes "Unofficial community build of Meshtastic firmware ${TAG} for both HackerBox #0111 boards (hackerboxes-esp32-io, hackerboxes-esp32c3-oled). Built from https://github.com/meshtastic/firmware/releases/tag/${TAG}." \
            dist/*.zip
```

- [ ] **Step 2: Commit and push**

```bash
git add .github/workflows/build-hackerbox-firmware.yml
git commit -m "Add publish job: zip artifacts and create a GitHub Release"
git push
```

- [ ] **Step 3: Trigger a fresh (not-yet-released) tag end to end**

Use the same `LATEST` tag from Task 3 — no release exists for it yet since Task 3 only tested through the `build` job:

```bash
LATEST=$(gh api repos/meshtastic/firmware/releases/latest --jq .tag_name)
gh workflow run build-hackerbox-firmware.yml -f tag="$LATEST"
sleep 10
RUN_ID=$(gh run list --workflow build-hackerbox-firmware.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --exit-status
```

Expected: run succeeds end to end (resolve → build → publish).

- [ ] **Step 4: Verify the release**

```bash
gh release view "$LATEST" --repo cyberglitchlabs/hackerboxes-meshtastic --json tagName,assets --jq '{tag: .tagName, assets: [.assets[].name]}'
```

Expected:
```json
{"tag":"<LATEST>","assets":["firmware-hackerboxes-esp32-io-<LATEST>.zip","firmware-hackerboxes-esp32c3-oled-<LATEST>.zip"]}
```

---

### Task 5: Verify dedup behavior (the daily-schedule safety net)

**Files:** none (verification only — this task proves Global Constraint "dedup" holds, using the release Task 4 just created).

**Interfaces:** none new.

- [ ] **Step 1: Re-run the workflow for the same tag that Task 4 just released**

```bash
LATEST=$(gh api repos/meshtastic/firmware/releases/latest --jq .tag_name)
gh workflow run build-hackerbox-firmware.yml -f tag="$LATEST"
sleep 10
RUN_ID=$(gh run list --workflow build-hackerbox-firmware.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --exit-status
```

Expected: run succeeds (this is a no-op run, not a failure).

- [ ] **Step 2: Verify `resolve` reported skip=true and downstream jobs did not run**

```bash
gh api "repos/cyberglitchlabs/hackerboxes-meshtastic/actions/runs/$RUN_ID/jobs" --jq '.jobs[] | {name, conclusion}'
```

Expected: `resolve` shows `conclusion: success`; `build` and `publish` show `conclusion: skipped` (not `success`, not `failure`) — confirming the `if: needs.resolve.outputs.skip == 'false'` gate worked and no duplicate/conflicting release was attempted.

- [ ] **Step 3: Confirm exactly one release still exists for that tag**

```bash
gh release list --repo cyberglitchlabs/hackerboxes-meshtastic --json tagName --jq '[.[] | select(.tagName == "'"$LATEST"'")] | length'
```

Expected: `1`.

- [ ] **Step 4: Commit final state (if any docs tweaks were made during verification)**

```bash
git status --short
```

If clean, nothing to commit — the pipeline is done and verified. If you touched the README/spec while debugging, commit those with a message describing what changed.

---

## Self-review notes

- **Spec coverage:** trigger (schedule + workflow_dispatch tag override) → Task 2; build via `gh-action-firmware` + OTA bundling → Task 3; publish as GitHub Release with matching tag → Task 4; dedup/idempotency → Task 5; README/hardware background/OLED-out-of-scope explanation → Task 1. All spec sections have a task.
- **Placeholder scan:** no TBD/TODO; every step has literal file content or literal commands with expected output.
- **Type/name consistency:** `resolve` job outputs `tag`/`skip` (Task 2) are referenced identically in Task 3 (`needs.resolve.outputs.tag`, `if: needs.resolve.outputs.skip == 'false'`) and Task 4 (`needs.resolve.outputs.tag`, same `if`). Artifact names `firmware-hackerboxes-esp32-io-<tag>` / `firmware-hackerboxes-esp32c3-oled-<tag>` are produced in Task 3's `upload-artifact` step and consumed verbatim in Task 4's `download-artifact` steps. Matrix values `hackerboxes-esp32-io`/`esp32` and `hackerboxes-esp32c3-oled`/`esp32c3` match the Global Constraints exactly.
