# Verify the Google AIY Voice HAT V1

This guide applies to the original **Voice HAT V1**, used with the Pi 3. The Voice Bonnet V2 uses a different audio codec and driver setup. Do not use V2 `aiy-voicebonnet-soundcard-dkms` instructions for this board.

## 1. Assemble with power disconnected

Shut down and unplug the Pi before attaching the HAT. Align the HAT's connector across the complete 40-pin header, without offsetting it by a row or column. Use the kit's standoffs. Connect its speaker and button to their labelled HAT connectors following Google's [Voice Kit V1 assembly guide](https://aiyprojects.withgoogle.com/voice-v1/). Keep loose wires from shorting neighbouring contacts.

The photo supplied during preparation shows the Pi 3 Model B+ without the HAT mounted. It confirms the printed model name; it does not verify the HAT revision, wiring, or operation.

## 2. Check the audio overlay exists

**Pi:**

```sh
ls /boot/firmware/overlays/googlevoicehat-soundcard.dtbo
dtoverlay -h googlevoicehat-soundcard
```

The Raspberry Pi kernel's [6.18 overlay reference](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/arch/arm/boot/dts/overlays/README) includes `googlevoicehat-soundcard`, as does its [6.12 reference](https://github.com/raspberrypi/linux/blob/rpi-6.12.y/arch/arm/boot/dts/overlays/README). This is source-level evidence, not a completed playback test on your installed image.

If the file is missing after the normal OS updates and reboot, stop here and inspect the installed kernel packages. Do not download a random overlay or assume the V2 driver is a replacement.

## 3. Enable V1 audio at boot

**Pi:** Back up and open the boot configuration:

```sh
sudo cp -n /boot/firmware/config.txt /boot/firmware/config.txt.before-voice-hat
sudo nano /boot/firmware/config.txt
```

In a section applying to all boards, such as `[all]`, add this line once:

```ini
dtoverlay=googlevoicehat-soundcard
```

If `dtparam=audio=on` exists, change it to `dtparam=audio=off` to disable the Pi's onboard analog audio and simplify selection. Keep other unrelated settings. Do not add a second conflicting audio overlay. In nano, save with Ctrl+O, Enter, and exit with Ctrl+X. Then:

```sh
sudo reboot
```

Google's [V1 driver and pinout reference](https://github.com/google/aiyprojects-raspbian/blob/aiyprojects/docs/voice.md#voice-hat-voice-kit-v1) names this overlay. Its older `/boot/config.txt` example predates the modern `/boot/firmware/config.txt` path. The historical full AIY system image includes software we do not need; the tracker requires the button and speaker, not Assistant or speech recognition.

## 4. Identify the sound card and test the speaker

**Pi:**

```sh
aplay -l
aplay -L
id
```

Find the Google Voice HAT playback card in the list. Record its actual ALSA card identifier and playback device number privately. Do not assume it is always card 0. A device string will look like `plughw:CARD=<actual-card-id>,DEV=0`.

Replace `YOUR_CARD_ID` below with the actual identifier printed by ALSA, and use the actual device number if it differs. Start at a comfortable speaker distance:

```sh
speaker-test -D 'plughw:CARD=YOUR_CARD_ID,DEV=0' -c 2 -r 48000 -t sine -f 440 -l 1
```

This is a stereo, 48 kHz sine test. It may be loud; press Ctrl+C to stop. The V1 kernel codec declares stereo 48 kHz playback, so future generated WAV cues should use a compatible format. [Raspberry Pi V1 codec source](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/sound/soc/bcm/googlevoicehat-codec.c).

The HAT's amplifier may not expose a hardware mixer volume control. `alsamixer` is a diagnostic tool, not proof that volume can be changed there. The tracker will scale generated audio samples for configurable volume and play cues through a selected ALSA device using an ordered background queue.

If normal-user playback reports permission denied, check `/dev/snd` ownership and membership in `audio`. If needed, run `sudo usermod -aG audio "$USER"`, then log out and back in. Do not solve this by running the tracker as root.

## 5. Verify the button separately from tracker gestures

Google's [V1 pinout](https://github.com/google/aiyprojects-raspbian/blob/aiyprojects/docs/voice.md#voice-hat-voice-kit-v1) maps the button to **BCM GPIO 23, physical header pin 16**. Google's [button implementation](https://github.com/google/aiyprojects-raspbian/blob/aiyprojects/src/aiy/board.py) uses an active-low input with a pull-up by default. BCM 23 is not physical pin 23.

**Pi:** Check permissions:

```sh
ls -l /dev/gpiochip*
id
```

If those GPIO device files belong to the `gpio` group and your user lacks it, run `sudo usermod -aG gpio "$USER"`, then log out and reconnect. Check device ownership instead of changing device files to world-writable.

Activate the environment from the setup guide, then run this temporary diagnostic:

```sh
GPIOZERO_PIN_FACTORY=lgpio python - <<'PY'
from gpiozero import Button
from signal import pause

button = Button(23, pull_up=True, bounce_time=0.05)
button.when_pressed = lambda: print("pressed", flush=True)
button.when_released = lambda: print("released", flush=True)
print("Press and release the HAT button. Ctrl+C stops this check.", flush=True)
try:
    pause()
finally:
    button.close()
PY
```

Each physical press and release should print the corresponding line. This diagnostic does not test single/double-tap semantics or save study sessions. The future tracker will implement its own configurable 400 ms double-tap window, delayed single confirmation, and tested debounce handling.

## If a check fails

For audio, inspect `sudo journalctl -k -b` and check for Voice HAT, I2S, or sound-card errors. Ensure the overlay is under the correct configuration section, rebooted, and the speaker cable is correctly connected. Confirm no other program is holding the sound device. Avoid publishing full logs if they include personal information.

For GPIO, check the kit connector, pin numbering, group membership, and whether another process owns GPIO 23. A button check passing does not imply audio works, and an audio check passing does not imply gesture detection is correct.

If current Trixie cannot initialise the V1 sound card, capture the kernel version and error. Bookworm Legacy Lite 64-bit is a documented secondary candidate, not a guaranteed fix. Before rewriting, preserve your configuration and any future data. If both fail, keep development in simulation while investigating the driver; do not claim hardware integration is complete.
