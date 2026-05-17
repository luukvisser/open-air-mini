# Open AIR Mini — ESPHome firmware

[![Validate configs](https://github.com/luukvisser/open-air-mini/actions/workflows/validate.yml/badge.svg)](https://github.com/luukvisser/open-air-mini/actions/workflows/validate.yml)
[![Build & Release Firmware](https://github.com/luukvisser/open-air-mini/actions/workflows/build-firmware.yml/badge.svg)](https://github.com/luukvisser/open-air-mini/actions/workflows/build-firmware.yml)

ESPHome firmware for the **Open AIR Mini** ventilation controller. This repository
follows the [Made for ESPHome](https://esphome.io/guides/made_for_esphome/) program and
ships a GitHub Actions pipeline that builds, releases, and publishes a per-device OTA
manifest so fielded devices auto-discover new firmware versions.

## Install

The simplest path:

1. Open the project's [GitHub Pages site](https://luukvisser.github.io/open-air-mini/)
   in Chrome or Edge on a desktop.
2. Plug the device in over USB and click **Install**.
3. After the device boots, join its `Open AIR Mini Setup` Wi-Fi network (or use the
   Improv flow in Home Assistant) to provide your Wi-Fi credentials.

## Automatic updates

The firmware exposes a `Firmware Update` entity (via ESPHome's
[`update.http_request` platform](https://esphome.io/components/update/http_request/))
that polls the per-device manifest every 6 hours:

```
https://luukvisser.github.io/open-air-mini/open-air-mini/manifest.json
```

When a new stable release is published, the entity surfaces in Home Assistant (and the
device's web UI) and the user can install with one click. Pre-releases are **not**
auto-published to Pages — they cut a GitHub Release only.

## Local development

1. `cp example.secrets.yaml secrets.yaml` and fill in your Wi-Fi etc. `secrets.yaml` is
   git-ignored — never commit real credentials.
2. Install ESPHome at the version this repo pins:
   ```sh
   uv sync          # uses pyproject.toml
   esphome compile open-air-mini.yaml
   esphome run open-air-mini.yaml
   ```
   (Plain `pip install esphome==<pinned-version>` works equivalently.)

## Release pipeline

Releases are device-scoped tags of the form `<slug>/v<semver>`:

```sh
# 1. Bump version in two places (kept in sync):
#      substitutions.config_version
#      esphome.project.version
# 2. Commit, then tag:
git tag open-air-mini/v1.0.1
git push origin open-air-mini/v1.0.1
```

The [`build-firmware.yml`](.github/workflows/build-firmware.yml) workflow then:

1. Parses the tag, looks the device up in [`devices.yaml`](devices.yaml), and verifies
   `esphome.project.version` matches the tag.
2. Compiles the firmware with the pinned ESPHome version.
3. Uploads `*.factory.bin`, `*.ota.bin`, and an `*.ota.md5` as GitHub Release assets.
4. For **stable** releases (no `-rc.N` etc. suffix), publishes a per-device
   `manifest.json` and a copy of the OTA + factory binaries to GitHub Pages, and
   regenerates the landing page so the new version is advertised.

Pre-releases (`open-air-mini/v1.0.0-rc.1`) build and create a Release but skip the Pages
publish — fielded devices won't be prompted to install them.

Pushes to `main` / pull-requests run [`validate.yml`](.github/workflows/validate.yml),
which runs `esphome config` against every entry in `devices.yaml` with both the pinned
and the latest unpinned ESPHome (the latter as an early-warning, non-blocking check).

### Adding a device

Add an entry to [`devices.yaml`](devices.yaml) and drop the YAML in the repo root. The
validate matrix picks it up automatically; release with a tag using the new slug.

## Repo layout

```
.github/workflows/   # CI: validate on PR, build & publish on tag
devices.yaml         # Source of truth for what gets built/released
open-air-mini.yaml   # Main device firmware
disconnected-mode-*.yaml  # Includable scripts toggled in open-air-mini.yaml
example.secrets.yaml # Template — copy to secrets.yaml locally
pyproject.toml       # Pins ESPHome version for reproducible builds
scripts/             # Manifest + landing-page generators (called by CI)
```

## Choosing the disconnected-mode behaviour

When Home Assistant cannot be reached, a _disconnected mode_ keeps the fan running. Two
variants are shipped; pick one by editing the `script:` block in
[`open-air-mini.yaml`](open-air-mini.yaml):

- **Without humidity sensor** — runs at a single fixed speed
  ([`disconnected-mode-without-humidity.yaml`](disconnected-mode-without-humidity.yaml)).
  Speed is set via `disconnected_default_fan_speed` (0–100).
- **With humidity sensor** — varies fan speed by humidity using the `disconnected_*`
  globals
  ([`disconnected-mode-with-humidity.yaml`](disconnected-mode-with-humidity.yaml)).
  Requires a humidity sensor with `id: air_humidity`.

Only one of the two `!include` lines may be active at a time.

## Sensor add-ons

Sensors are added by appending platform entries at the bottom of `open-air-mini.yaml`.
See the original sensor cookbook in this repo's git history for SHT-31 / SHT-4x / SCD-40
/ SGP-41 / Senseair S8 / SHT-20 snippets — copy the snippet that matches your hardware.
When using multiple boards, replace the `x` in the example sensor names with a unique
letter/number so Home Assistant can tell them apart.
