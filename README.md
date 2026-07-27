# Xiaomi Book Pro 14 2026 - Linux Driver Fixes

This repository documents the fixes for running Linux (Fedora 44) on the **Xiaomi Book Pro 14 (2026)** with Intel Panther Lake platform.

[中文文档](README-cn.md)

## Hardware Overview

| Component | Device ID | Status |
|-----------|-----------|--------|
| GPU | Intel Panther Lake Xe (iGPU) | ✅ Working (with kernel param) |
| Audio | Intel Panther Lake `8086:e428` (SOF) | ✅ Working (kernel ≥ 7.1) |
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

## Fix 3: Speakers & Microphone (Intel Panther Lake Audio)

**Symptom:** `aplay -l` shows no sound cards (or only USB audio). Built-in speakers and microphone not working.

**Root Cause:** Intel Panther Lake (`8086:e428`) audio requires Sound Open Firmware (SOF) with function topology support, which was introduced in Linux 7.1.

**Solution:**

```bash
# Upgrade to kernel ≥ 7.1.4
sudo dnf install kernel-7.1.4-204.fc44

# Ensure SOF firmware is installed
sudo dnf install alsa-sof-firmware

# Reboot
sudo reboot
```

After reboot, verify:
```bash
cat /proc/asound/cards   # Should show Intel sound card
aplay -l                  # Should list playback devices
arecord -l                # Should list capture devices
```

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

**Tested on:** Fedora 44, kernel 7.1.4-204.fc44.x86_64

**Working:**
- Display (with `xe.enable_psr=0`)
- Keyboard (with `i8042.dumbkbd`)
- Touchpad
- Wi-Fi / Bluetooth
- Speakers & Microphone
- Fingerprint Sensor
- Camera
- Suspend/Resume (s2idle only, with workaround)
