# Xiaomi Book Pro 14 2026 - Linux Driver Fixes

[English](#english) | [中文](#chinese)

---

## English

This repository documents the fixes and workarounds for running Linux (Fedora 44) on the **Xiaomi Book Pro 14 (2026)** laptop with Intel Panther Lake platform.

### Hardware Overview

| Component | Device ID | Status |
|-----------|-----------|--------|
| GPU | Intel Panther Lake Xe (iGPU) | ✅ Working (with kernel param) |
| Audio | Intel Panther Lake `8086:e428` (SOF) | ✅ Working (kernel ≥ 7.1) |
| Fingerprint | Goodix `27c6:6890` | ✅ Working |
| Keyboard | - | ✅ Working (with kernel param) |
| Webcam | Shinetech `3277:00ff` | ⚠️ Untested |
| Wi-Fi / BT | Intel BE201 | ✅ Working |

### Fix 1: GPU Screen Corruption & Keyboard Not Working

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

### Fix 2: Fingerprint Sensor (Goodix 27c6:6890)

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

### Fix 3: Speakers & Microphone (Intel Panther Lake Audio)

**Symptom:** `aplay -l` shows no sound cards (or only USB audio). Built-in speakers and microphone not working.

**Root Cause:** Intel Panther Lake (`8086:e428`) audio requires:
1. Sound Open Firmware (SOF) with function topology support (Linux ≥ 7.1)
2. SOF drivers must NOT be blacklisted

**Solution:**

```bash
# Upgrade to kernel ≥ 7.1.4
sudo dnf install kernel-7.1.4-204.fc44

# Ensure SOF firmware is installed
sudo dnf install alsa-sof-firmware

# Remove any SOF blacklist (if previously created)
sudo rm -f /etc/modprobe.d/blacklist-sof*.conf

# Rebuild initramfs and reboot
sudo dracut -f
sudo reboot
```

After reboot, verify:
```bash
cat /proc/asound/cards   # Should show Intel sound card
aplay -l                  # Should list playback devices
arecord -l                # Should list capture devices
```

### Fix 4: GNOME Keyring After Fingerprint Login

**Symptom:** After logging in with fingerprint, GNOME Keyring prompts for password to unlock.

**Solution:** Set the keyring password to blank.

1. Open **Passwords and Keys** (seahorse) from GNOME app menu
2. Right-click **Default keyring** → **Change Password**
3. Enter current password, leave new password blank, confirm

Alternatively, from command line:
```bash
# Install seahorse if not present
sudo dnf install seahorse
```

### System Summary

**Tested on:** Fedora 44, kernel 7.1.4-204.fc44.x86_64

**Working:**
- Display (with `xe.enable_psr=0`)
- Keyboard (with `i8042.dumbkbd`)
- Touchpad
- Wi-Fi / Bluetooth
- Speakers & Microphone
- Fingerprint Sensor
- Webcam (likely, untested)
- Suspend/Resume

---

## 中文

本文档记录了在**小米笔记本 Pro 14 (2026)** 上运行 Linux (Fedora 44) 所需的各种驱动修复方案。

### 硬件概览

| 组件 | 设备 ID | 状态 |
|------|-----------|------|
| 核显 | Intel Panther Lake Xe | ✅ 正常（需内核参数） |
| 声卡 | Intel Panther Lake `8086:e428` | ✅ 正常（内核 ≥ 7.1） |
| 指纹 | Goodix `27c6:6890` | ✅ 正常 |
| 键盘 | - | ✅ 正常（需内核参数） |
| 摄像头 | Shinetech `3277:00ff` | ⚠️ 未测试 |
| 无线网卡 | Intel BE201 | ✅ 正常 |

### 修复 1：核显花屏 & 键盘不可用

**症状：**
- Intel Xe 核显出现闪屏、花屏或黑屏
- 笔记本自带键盘无法使用

**解决：** 添加内核启动参数。

```bash
sudo grubby --update-kernel=ALL --args="xe.enable_psr=0 i8042.dumbkbd"
```

或手动编辑 `/etc/default/grub`：

```
GRUB_CMDLINE_LINUX="... xe.enable_psr=0 i8042.dumbkbd"
```

然后重建 GRUB 配置：

```bash
# UEFI 系统：
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**说明：**
- `xe.enable_psr=0`：关闭 Intel Xe 驱动的 Panel Self Refresh，解决 Panther Lake 核显闪屏/花屏问题
- `i8042.dumbkbd`：强制 i8042 键盘控制器进入简化模式，绕过有问题的 ACPI/BIOS 键盘处理逻辑

### 修复 2：指纹传感器 (Goodix 27c6:6890)

**症状：** `fprintd-enroll` 报告 "No devices available"，`lsusb` 能看到设备但 fprintd 无法使用。

**根因：** 系统自带的 libfprint (v1.94.10) 的 Goodix MOC 驱动设备 ID 表中不包含 `27c6:6890`。

**解决：** 从 git master 构建 libfprint。

```bash
# 安装构建依赖
sudo dnf builddep libfprint

# 克隆并构建
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git
cd libfprint
meson setup builddir -Ddoc=false -Dprefix=/usr
ninja -C builddir
sudo ninja -C builddir install

# 重启 fprintd 服务
sudo systemctl restart fprintd

# 录入指纹
fprintd-enroll
```

**启用指纹登录/sudo：**

```bash
sudo authselect enable-feature with-fingerprint
sudo authselect apply-changes
```

### 修复 3：扬声器 & 麦克风（Intel Panther Lake 声卡）

**症状：** `aplay -l` 显示无声卡（或仅有 USB 音频设备），自带扬声器和麦克风不可用。

**根因：** Intel Panther Lake (`8086:e428`) 音频需要：
1. 支持功能拓扑 (function topology) 的 SOF 驱动（Linux ≥ 7.1）
2. SOF 驱动不能被屏蔽

**解决：**

```bash
# 升级内核到 ≥ 7.1.4
sudo dnf install kernel-7.1.4-204.fc44

# 确保 SOF 固件已安装
sudo dnf install alsa-sof-firmware

# 删除之前的 SOF 黑名单（如有）
sudo rm -f /etc/modprobe.d/blacklist-sof*.conf

# 重建 initramfs 并重启
sudo dracut -f
sudo reboot
```

重启后验证：
```bash
cat /proc/asound/cards   # 应显示 Intel 声卡
aplay -l                  # 应列出播放设备
arecord -l                # 应列出录音设备
```

### 修复 4：指纹登录后 GNOME Keyring 解锁

**症状：** 使用指纹登录系统后，GNOME Keyring 弹出密码框要求输入密码解锁。

**解决：** 将密钥环密码设为空。

1. 在 GNOME 应用菜单中打开「**密码和密钥**」(Passwords and Keys / seahorse)
2. 右键「**默认密钥环**」(Default keyring) → **更改密码**
3. 输入当前密码，新密码留空，确认

或命令行安装：
```bash
sudo dnf install seahorse
```

### 系统总结

**测试环境：** Fedora 44, kernel 7.1.4-204.fc44.x86_64

**已确认正常：**
- 屏幕显示（需 `xe.enable_psr=0`）
- 键盘（需 `i8042.dumbkbd`）
- 触摸板
- Wi-Fi / 蓝牙
- 扬声器 & 麦克风
- 指纹传感器
- 摄像头（大概率正常，未测试）
- 睡眠/唤醒
