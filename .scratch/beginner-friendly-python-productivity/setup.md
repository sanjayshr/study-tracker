# Install Raspberry Pi OS and prerequisites

This guide prepares the Pi before we write the tracker. Commands marked **Pi** run in its SSH session; commands marked **laptop** run on the computer beside it. Do not paste the whole guide at once. Finish each step and check its result.

## 1. Gather the parts

You need your Raspberry Pi 3 Model B+, 32 GB microSD card, a card reader, a laptop, an internet connection, and a reliable 5 V / 2.5 A micro-USB supply. Ethernet is convenient for the first boot; Wi-Fi also works. Keep the Voice HAT, its button, and its speaker ready for the later hardware check.

Writing the OS erases the selected card. Copy anything you need off it first. Check that the storage selected in Imager is the 32 GB microSD card, not another drive.

## 2. Choose the OS

Use **Raspberry Pi OS Lite (64-bit), Debian Trixie**, the current official Lite release when this guide was checked on 2026-10-04. The Raspberry Pi download page lists the 3B+ as compatible. Lite has no desktop and leaves more of the Pi's 1 GB RAM available. [Official OS downloads](https://www.raspberrypi.com/software/operating-systems/).

This is our setup candidate, not a claim that this particular HAT has been tested on it. The current Raspberry Pi kernel source documents the V1 audio overlay, but the installed kernel, wiring, permissions, and real playback must still pass [the Voice HAT checks](voice-hat.md). If they fail, investigate the reported error before changing OS. Current official Bookworm Legacy Lite 64-bit is a secondary candidate; its kernel also documents this overlay. Old Google AIY images are historical references, not our default OS.

## 3. Write the microSD card on the laptop

Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/) for your laptop's operating system and install it. Windows, macOS, and Linux versions are available. Use a current Imager; screens may differ slightly between releases.

1. Insert the microSD card into the laptop's reader.
2. Select **Raspberry Pi 3** as the device.
3. Select **Raspberry Pi OS Lite (64-bit)**. Depending on the Imager version, this is under **Raspberry Pi OS (other)**. Check that it is the Trixie Lite image, not Desktop or the 32-bit image.
4. Select the microSD card as storage.
5. Set the hostname to `study-pi`.
6. Create a non-root username, such as `student`, and a strong password. There is no assumed default `pi` login.
7. Set your Wi-Fi name, password, and actual country if using Wi-Fi. Use your real country even though the dashboard will show India time.
8. Set timezone to `Asia/Kolkata`, and choose your keyboard layout.
9. Enable SSH. Password authentication is simplest for a first setup; an SSH public key is preferable if you already have one. These settings are for local network access.
10. Write the card, allow verification to finish, and safely eject it.

Raspberry Pi's [getting-started instructions](https://www.raspberrypi.com/documentation/computers/getting-started.html) describe Imager customisation and headless boot.

## 4. Boot and connect

With power disconnected, insert the card into the slot underneath the Pi. Connect Ethernet if using it. Plug in the micro-USB power supply and allow a few minutes for first boot.

**Laptop:** Open Windows PowerShell, macOS Terminal, or your Linux terminal. If you selected `student`, connect with:

```sh
ssh student@study-pi.local
```

Use your chosen username if different. Accept the host key on this first connection and enter the password. Password typing shows no characters; that is normal.

If `.local` cannot be found, look up `study-pi` in your router's connected-device list and use its local IP, for example `ssh student@192.168.1.50`. You do not need port forwarding or a tunnel. If SSH is unavailable on the laptop, use an HDMI monitor and USB keyboard for the same Pi steps. Keep SSH confined to your local network.

## 5. Verify what actually booted

**Pi:** Run these commands before installing publishing tools:

```sh
cat /etc/os-release
uname -m
dpkg --print-architecture
getconf LONG_BIT
getconf GNU_LIBC_VERSION
uname -r
tr -d '\000' < /proc/device-tree/model
```

Expect Debian Trixie / Raspberry Pi OS, `aarch64`, `arm64`, and `64`. The model should identify a Pi 3 Model B+. `uname -m` describes the kernel; `dpkg --print-architecture` describes the installed userland. An `armhf` userland is 32-bit even if the CPU supports 64-bit.

If you accidentally installed a fresh 32-bit OS, rewrite the card with the 64-bit image before adding project data. Save the OS and kernel versions in a private setup note for troubleshooting. Never treat a CPU label alone as proof of compatibility.

## 6. Update and install the base tools

**Pi:**

```sh
sudo apt update
sudo apt full-upgrade
sudo apt install git python3 python3-venv python3-pip python3-gpiozero python3-lgpio alsa-utils sqlite3 curl ca-certificates xz-utils nano
sudo timedatectl set-timezone Asia/Kolkata
sudo timedatectl set-ntp true
sudo reboot
```

Reconnect over SSH after reboot. `git` downloads and versions the project; Python runs it; `venv` isolates its future dependencies. GPIO Zero and lgpio provide a candidate button interface. ALSA tools include `aplay` and `speaker-test`. The `sqlite3` command helps inspect and back up databases; the application will use Python's built-in SQLite module. `curl`, certificate files, and `xz` support downloading Node later; `nano` edits configuration.

Only package installation and OS configuration need `sudo`. Run application code as your normal user. Do not install Python libraries with `sudo pip` or use `--break-system-packages`.

## 7. Download the repository and prepare Python

**Pi:**

```sh
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/sanjayshr/study-tracker.git
cd study-tracker
python3 -m venv --system-site-packages .venv
. .venv/bin/activate
python -c 'import sqlite3, wave; from zoneinfo import ZoneInfo; print("SQLite", sqlite3.sqlite_version); print("Timezone", ZoneInfo("Asia/Kolkata"))'
python -c 'import gpiozero, lgpio; print("GPIO libraries available")'
```

The environment inherits the OS-installed GPIO libraries so we do not compile hardware libraries unnecessarily on a small Pi. Future project-only dependencies will be pinned separately. Opening a new SSH connection requires running `cd ~/projects/study-tracker` and `. .venv/bin/activate` again.

There is no tracker command or systemd service to start yet. The repository currently contains preparation documentation and publishing tooling. You do not need Flask, Django, a JavaScript framework, an AI SDK, Docker, or a cloud database.

## 8. Check the clock and storage

**Pi:**

```sh
timedatectl status
df -h /
free -h
vcgencmd get_throttled
```

Check that the timezone is India time and the clock synchronises while online. The Pi 3 does not have a battery-backed real-time clock. After an offline reboot, its wall clock may be wrong until synchronised. The future tracker must handle that honestly; timezone settings alone do not solve it. Sessions will store UTC timestamps and use monotonic elapsed timing.

`get_throttled=0x0` means no flags are set. A different result merits checking supply quality and temperature; consult [Raspberry Pi's hardware documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#undervoltage-warning). Use `sudo shutdown -h now` and wait for shutdown before removing power.

## 9. Verify the HAT, then prepare publishing

Continue with [Voice HAT V1 checks](voice-hat.md). Only after identifying the actual OS architecture, audio device, and button behaviour should you follow [publishing prerequisites](publishing.md).

You are ready for project development when SSH works, the OS is confirmed, Python/SQLite/timezone imports pass, the clock is sane, the button produces press/release events, and the HAT speaker plays a test sound. Wrangler on the Pi is an additional publishing check; a laptop fallback is available if it fails.

## Common first-boot problems

- **No SSH connection:** Check power, wait for boot, confirm both devices are on the same network, try Ethernet and the router's local IP. Guest Wi-Fi may isolate devices. Check that SSH was enabled in Imager.
- **Wrong password:** Use the username and password you created, not a guessed default login.
- **Host key changed after rewriting the card:** Verify that this is your freshly reimaged Pi, then remove its old entry with `ssh-keygen -R study-pi.local` on the laptop. Repeat for the IP if you connected that way.
- **Package download fails:** Confirm networking and system time, then rerun `sudo apt update`. Do not disable certificate verification.
- **Python reports an externally managed environment:** Activate `.venv` before adding Python packages.
- **Pi resets unexpectedly:** Check the micro-USB supply and cable. Repeated resets can corrupt the card.
