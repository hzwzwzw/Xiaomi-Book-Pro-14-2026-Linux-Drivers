# 小米笔记本 Pro 14 2026 - Linux 驱动修复

本文档记录了在**小米笔记本 Pro 14 (2026)**（Intel Panther Lake 平台）上运行 Linux (Fedora 44) 所需的驱动修复方案。

[English Documentation](README.md)

## 硬件概览

| 组件 | 设备 ID | 状态 |
|------|-----------|------|
| 核显 | Intel Panther Lake Xe | ✅ 正常（需内核参数） |
| 声卡 | Intel Panther Lake `8086:e428` | ✅ 正常（内核 ≥ 7.1） |
| 指纹 | Goodix `27c6:6890` | ✅ 正常 |
| 键盘 | - | ✅ 正常（需内核参数） |
| 摄像头 | Shinetech `3277:00ff` | ✅ 正常 |
| 无线网卡 | Intel BE201 | ✅ 正常 |

## 修复 1：核显花屏 & 键盘不可用

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
- `xe.enable_psr=0`：关闭 Intel Xe 驱动的 Panel Self Refresh（面板自刷新），解决 Panther Lake 核显闪屏/花屏问题
- `i8042.dumbkbd`：强制 i8042 键盘控制器进入简化模式，绕过有问题的 ACPI/BIOS 键盘处理逻辑

## 修复 2：指纹传感器 (Goodix 27c6:6890)

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

## 修复 3：扬声器 & 麦克风（Intel Panther Lake 声卡）

**症状：** `aplay -l` 显示无声卡（或仅有 USB 音频设备），自带扬声器和麦克风不可用。

**根因：** Intel Panther Lake (`8086:e428`) 音频需要功能拓扑 (function topology) 支持的 SOF 驱动（Linux ≥ 7.1）。

**解决：**

```bash
# 升级内核到 ≥ 7.1.4
sudo dnf install kernel-7.1.4-204.fc44

# 确保 SOF 固件已安装
sudo dnf install alsa-sof-firmware

# 重启
sudo reboot
```

重启后验证：
```bash
cat /proc/asound/cards   # 应显示 Intel 声卡
aplay -l                  # 应列出播放设备
arecord -l                # 应列出录音设备
```

## 系统总结

**测试环境：** Fedora 44, kernel 7.1.4-204.fc44.x86_64

**已确认正常：**
- 屏幕显示（需 `xe.enable_psr=0`）
- 键盘（需 `i8042.dumbkbd`）
- 触摸板
- Wi-Fi / 蓝牙
- 扬声器 & 麦克风
- 指纹传感器
- 摄像头
- 睡眠/唤醒
