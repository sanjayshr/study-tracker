# Preparation verification record

Checked on 2026-10-04. This record describes documentation and tool preparation, not finished application acceptance tests.

## Available workspace

The repository initially contained only its title README and one initial commit. No Git remote was configured. The available machine is Ubuntu 24.04.5 LTS on x86_64, with glibc 2.39, Python 3.12.3, Node.js 24.19.0, and npm 11.17.0. There is no Pi GPIO device or attached Voice HAT here. No physical Pi connection was discovered or used.

The supplied photo clearly shows the board marking “Raspberry Pi 3 Model B+”. The HAT is not mounted in that photo. The Pi's installed OS, microSD contents, and kit wiring remain unknown.

## Checks completed locally

- Installed exact Wrangler 4.147.0 locally and generated a dependency lockfile.
- Confirmed Wrangler prints its version on the available Node 24.19.0 installation.
- Confirmed `pages project create --help` supports the explicit project name and `--production-branch` option.
- Confirmed `pages deploy --help` supports `--project-name` and `--branch`.
- Confirmed the installed workerd executable starts and reports `2026-10-01`.
- Confirmed Python imports SQLite and the `Asia/Kolkata` timezone.
- Checked all six Markdown documents for valid local links and balanced code fences; all 19 shell examples passed Bash syntax checks.
- Checked matching Wrangler pins, Git ignore coverage for common private/generated files, and whitespace errors. A targeted credential-pattern scan found no matches in the prepared files.
- Confirmed the official checksum manifest contains the documented ARM64 Node archive. The ARM64 archive was not installed or executed here.

The npm install reported script-policy warnings for esbuild and workerd; the CLI and runtime checks passed. No Cloudflare login, token, project creation, or deployment was performed.

## Official documentation findings

The [official OS catalogue](https://www.raspberrypi.com/software/operating-systems/) lists Raspberry Pi OS Lite 64-bit Trixie for the 3B+, and Bookworm Legacy Lite as another available release. The proposed primary image is Trixie Lite 64-bit.

Google's [V1 hardware reference](https://github.com/google/aiyprojects-raspbian/blob/aiyprojects/docs/voice.md#voice-hat-voice-kit-v1) identifies BCM GPIO 23 / physical pin 16 for the button and the `googlevoicehat-soundcard` audio overlay. Its [button implementation](https://github.com/google/aiyprojects-raspbian/blob/aiyprojects/src/aiy/board.py) confirms the default pull-up, falling/active-low input convention.

The Raspberry Pi [6.18 overlay reference](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/arch/arm/boot/dts/overlays/README) and [V1 codec source](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/sound/soc/bcm/googlevoicehat-codec.c) provide current source-level support evidence. Installed-image support and actual playback still require the documented checks.

Cloudflare's [Wrangler requirements](https://developers.cloudflare.com/workers/wrangler/install-and-update/), [runtime requirements](https://github.com/cloudflare/workerd#running-workerd), [Pages Direct Upload instructions](https://developers.cloudflare.com/pages/get-started/direct-upload/), and [Pages limits](https://developers.cloudflare.com/pages/platform/limits/) were reviewed. The installed npm package metadata requires Node >=22 and publishes a Linux ARM64 runtime package. None of these establish that our pins work on this specific Pi without running them there.

## Still to verify on the actual devices

- Successful flash, boot, network connection, and non-root SSH login.
- Actual OS release, userland architecture, kernel, glibc, and CPU features.
- Time synchronisation, supply health, and free storage.
- V1 board identity, button wiring, permission checks, and press/release events.
- V1 sound-card initialisation, actual ALSA device identifier, and speaker playback.
- Node/Wrangler startup on ARM64 and memory use during a real upload.
- Cloudflare account setup, token scope, dashboard preview, and chosen access policy.

State-machine tests, gesture semantics, SQLite recovery, dashboard layouts, retry behaviour, and systemd startup will be tested during application implementation. They have not been implemented or tested yet.
