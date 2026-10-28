# Xiaomi Book Pro 14 2026 - Linux Driver Fixes

This repository documents the fixes for running Linux (Fedora 44) on the **Xiaomi Book Pro 14 (2026)** with Intel Panther Lake platform.

[中文文档](README-cn.md)

## Hardware Overview

| Component | Device ID | Status |
|-----------|-----------|--------|
| GPU | Intel Panther Lake Xe (iGPU) | ✅ Working (with kernel param) |
| Audio | Intel Panther Lake `8086:e428` (SOF) | ✅ Speakers/mic; headphones need RT721 workaround below |
| Fingerprint | Goodix `27c6:6890` | ✅ Working |
| Keyboard | - | ✅ Working (with kernel param) |
| Camera | Shinetech `3277:00ff` | ✅ Working |
| Wi-Fi / BT | Intel BE201 | ✅ Working |

## Fix 1: GPU Screen Corruption & Keyboard Not Working

**Symptoms:**
- Screen flickering, artifacts, or blank display with Intel Xe graphics
- Built-in keyboard not responding

**Solution:** Add kernel boot parameters.

```bash
sudo grubby --update-kernel=ALL --args="xe.enable_psr=0 i8042.dumbkbd"
```

Or manually edit `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX="... xe.enable_psr=0 i8042.dumbkbd"
```

Then rebuild GRUB config:

```bash
# For UEFI systems:
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**Explanation:**
- `xe.enable_psr=0`: Disables Panel Self Refresh (PSR) on Intel Xe driver, fixing screen flicker/corruption on Panther Lake iGPU
- `i8042.dumbkbd`: Forces the i8042 keyboard controller into "dumb" mode, bypassing problematic ACPI/BIOS keyboard handling

## Fix 2: Fingerprint Sensor (Goodix 27c6:6890)

**Symptom:** `fprintd-enroll` reports "No devices available", `lsusb` shows the device but fprintd can't use it.

**Root Cause:** The installed libfprint (v1.94.10) does not include USB ID `27c6:6890` in its Goodix MOC driver ID table.

**Solution:** Build libfprint from git master.

```bash
# Install build dependencies
sudo dnf builddep libfprint

# Clone and build
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git
cd libfprint
meson setup builddir -Ddoc=false -Dprefix=/usr
ninja -C builddir
sudo ninja -C builddir install

# Restart fprintd
sudo systemctl restart fprintd

# Enroll fingerprint
fprintd-enroll
```

**Enable fingerprint for sudo/login:**

```bash
sudo authselect enable-feature with-fingerprint
sudo authselect apply-changes
```

## Fix 3: Built-in Audio and 3.5mm Headphones (Intel Panther Lake / RT721)

### 3.1 No built-in sound card

**Symptom:** `aplay -l` shows no sound cards, or only USB audio; the internal speakers and microphone do not work.

Intel Panther Lake (`8086:e428`) requires SOF with function topology support. The original speaker/microphone fix was tested on Fedora 44 with `7.1.4-204.fc44`; this does not establish headphone compatibility on every later kernel.

```bash
sudo dnf install kernel alsa-sof-firmware
# Save your work and reboot, then check the sound card
cat /proc/asound/cards
aplay -l
arecord -l
```

### 3.2 3.5mm headphones detected but silent (updated 2026-10-28)

**Observed on this laptop:** The same headphones work in Windows and through a USB-C adapter. Linux initially missed jack insertion, then detected the headphones but produced no sound. On Fedora 44 / `7.2.11-200.fc44.x86_64`, output selection, volume and mute were correct; direct ALSA playback with PipeWire stopped was also silent. Reboots and an older kernel did not resolve it, so a particular kernel update has not been established as the cause.

**Working workaround:** Use the RT721 codec driver at upstream commit [`e2eb4873b901`](https://github.com/torvalds/linux/commit/e2eb4873b901290adc56b76f1d599005e620a6de), `ASoC: rt721-sdca: Adjust blindwrite flow to fix type detection issue`, managed by DKMS. After temporarily loading this driver, both the system sound test and ordinary audio played through the headphones; the internal microphone device remained available. The evidence points toward driver initialization or type detection, but does not isolate a particular register change.

The tested [driver sources and DKMS configuration](audio/rt721-dkms/) are included. This replaces the codec module, not the entire kernel. For a **fresh installation**, run from this repository's root:

```bash
kver="$(uname -r)"
# Match the running kernel instead of pulling in another kernel through DKMS dependencies
sudo dnf install gcc make dkms "kernel-devel-${kver}" "kernel-devel-matched-${kver}"

sudo install -d -m 0755 /usr/src/xiaomi-rt721-fix-0.1
sudo install -m 0644 audio/rt721-dkms/rt721-sdca.c \
  audio/rt721-dkms/rt721-sdca.h \
  audio/rt721-dkms/rt721-sdca-sdw.c \
  audio/rt721-dkms/rt721-sdca-sdw.h \
  audio/rt721-dkms/rt-sdw-common.h \
  audio/rt721-dkms/Makefile audio/rt721-dkms/dkms.conf \
  /usr/src/xiaomi-rt721-fix-0.1/
sudo dkms add -m xiaomi-rt721-fix -v 0.1
sudo dkms build -m xiaomi-rt721-fix -v 0.1 -k "$kver"
sudo dkms install -m xiaomi-rt721-fix -v 0.1 -k "$kver"
```

If matching development packages have left the repositories, follow the [Fedora build archive instructions](audio/rt721-dkms/README.md#matching-fedora-development-packages) to install the matching two RPMs from official Koji storage first. Do not use development files for a different kernel. If `xiaomi-rt721-fix/0.1` is already installed, do not repeat `dkms add`.

Save your work and reboot, select headphones in system sound settings, and test both system and ordinary audio. Check:

```bash
uname -r
dkms status
modinfo -F filename snd_soc_rt721_sdca
```

On a supported kernel, DKMS should report `xiaomi-rt721-fix/0.1 ... installed`. The module path should contain `extra/` (observed on this Fedora system) or `updates/dkms/`. The distribution driver is retained for rollback.

**Scope and validation:**

- Actually tested: Fedora 44, `7.2.11-200.fc44.x86_64`, RT721 + RT1320, headphone system tests and ordinary playback.
- DKMS automatically builds for `7.2.x` / `7.3.x`; only `7.2.11` has been verified on this laptop. Kernel updates need matching `kernel-devel`; check `dkms status` afterward.
- No local override is built for `7.1.x` or `7.4+`; those use the distribution driver. This does not claim that `7.4+` works on this laptop.
- Secure Boot was disabled during testing. Systems with Secure Boot enabled must trust the DKMS module's signing key before loading it.
- Persistence across reboot, jack reinsertion, speaker switching and suspend/resume still need verification.

**Rollback:**

```bash
sudo dkms remove -m xiaomi-rt721-fix -v 0.1 --all
# Once complete, save your work and reboot to use the distribution driver
```

See [audio/rt721-dkms/README.md](audio/rt721-dkms/README.md) for source provenance, build configuration and archived development packages.

## Fix 4: Lid Close Suspend

**Symptom:** Closing the lid does not trigger suspend. `journalctl -b -k | grep PM:` shows:

```
bitland_mifs_wmi: PM: dpm_run_callback(): bitland_mifs_wmi_suspend returns -22
PM: Some devices failed to suspend, or early wake event detected
```

**Root Cause:** The `bitland_mifs_wmi` driver (Xiaomi WMI platform driver for fan/keyboard backlight/performance mode) has a broken suspend callback that returns `-EINVAL`, causing the entire suspend to fail.

**Solution:** Create a systemd-sleep hook to unbind the device before suspend and rebind after resume.

```bash
sudo tee /usr/lib/systemd/system-sleep/bitland-workaround.sh << 'SCRIPT'
#!/bin/bash
DEV="B60BFB48-3E5B-49E4-A0E9-8CFFE1B3434B-4"

case $1 in
  pre)
    echo -n "$DEV" > /sys/bus/wmi/drivers/bitland-mifs-wmi/unbind 2>/dev/null
    ;;
  post)
    echo -n "$DEV" > /sys/bus/wmi/drivers/bitland-mifs-wmi/bind 2>/dev/null
    ;;
esac
SCRIPT
sudo chmod +x /usr/lib/systemd/system-sleep/bitland-workaround.sh
```

**Note:** Only `s2idle` (default) works. `deep` (S3) causes the system to hang on resume due to BIOS bugs. Do NOT set `mem_sleep_default=deep`.

## System Summary

**Original test environment:** Fedora 44, kernel 7.1.4-204.fc44.x86_64

**Additional audio verification (2026-10-28):** Fedora 44, kernel 7.2.11-200.fc44.x86_64; headphone system tests and ordinary playback work after applying the RT721 DKMS workaround above.

**Working:**
- Display (with `xe.enable_psr=0`)
- Keyboard (with `i8042.dumbkbd`)
- Touchpad
- Wi-Fi / Bluetooth
- Speakers & Microphone
- Fingerprint Sensor
- Camera
- Suspend/Resume (s2idle only, with workaround)
