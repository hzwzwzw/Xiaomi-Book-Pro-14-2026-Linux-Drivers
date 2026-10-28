# 小米笔记本 Pro 14 2026 - Linux 驱动修复

本文档记录了在**小米笔记本 Pro 14 (2026)**（Intel Panther Lake 平台）上运行 Linux (Fedora 44) 所需的驱动修复方案。

[English Documentation](README.md)

## 硬件概览

| 组件 | 设备 ID | 状态 |
|------|-----------|------|
| 核显 | Intel Panther Lake Xe | ✅ 正常（需内核参数） |
| 声卡 | Intel Panther Lake `8086:e428` | ✅ 扬声器/麦克风；耳机需 RT721 修复（见下文） |
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

## 修复 3：内置音频与 3.5mm 耳机（Intel Panther Lake / RT721）

### 3.1 内置声卡完全不可用

**症状：** `aplay -l` 显示无声卡（或仅有 USB 音频设备），自带扬声器和麦克风不可用。

Intel Panther Lake (`8086:e428`) 需要带功能拓扑 (function topology) 支持的 SOF 驱动。原方案在 Fedora 44、内核 `7.1.4-204.fc44` 上验证了扬声器和麦克风；这不代表所有后续内核的耳机功能都正常。

```bash
sudo dnf install kernel alsa-sof-firmware
# 保存工作后重启，再检查声卡
cat /proc/asound/cards
aplay -l
arecord -l
```

### 3.2 能识别 3.5mm 耳机，但没有声音（2026-10-28 更新）

**本机现象：** Windows 中耳机正常，USB-C 转接也可用；Linux 下曾检测不到插入，后来能识别耳机但仍无声。Fedora 44、`7.2.11-200.fc44.x86_64` 下，默认输出、音量和静音设置正常；停止 PipeWire 后直接用 ALSA 播放也无声。重启及加载旧内核没有解决，因此尚不能确认是某次内核升级引入的回归。

**验证有效的方案：** 将 RT721 codec 驱动换为上游提交 [`e2eb4873b901`](https://github.com/torvalds/linux/commit/e2eb4873b901290adc56b76f1d599005e620a6de) 对应版本（`ASoC: rt721-sdca: Adjust blindwrite flow to fix type detection issue`），并用 DKMS 管理。临时加载后，已确认系统声音测试和日常音频可以通过耳机播放；内置麦克风通道仍存在。证据指向驱动初始化或类型检测流程，但尚未定位到具体单项寄存器变化。

本仓库提供了相同的[驱动源码与 DKMS 配置](audio/rt721-dkms/)，不是替换整个内核。以下是**首次安装**步骤，在本仓库根目录执行：

```bash
kver="$(uname -r)"
# 精确匹配当前内核，避免 DKMS 依赖顺带安装另一版内核
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

如果当前内核的开发包已从软件源移除，先按[开发包归档获取方法](audio/rt721-dkms/README.md#matching-fedora-development-packages)从 Fedora 官方 Koji 下载匹配的两个 RPM，再安装。不要混用其他版本的开发包。已经安装 `xiaomi-rt721-fix/0.1` 的机器无需重复 `dkms add`。

保存工作后重启，在系统声音设置中选择耳机并测试，再播放日常音频。检查：

```bash
uname -r
dkms status
modinfo -F filename snd_soc_rt721_sdca
```

当前适用内核应显示 `xiaomi-rt721-fix/0.1 ... installed`，模块路径应包含 `extra/`（本机 Fedora）或 `updates/dkms/`。系统原驱动保留，可回退。

**适用范围与验证限制：**

- 已实际验证：Fedora 44、内核 `7.2.11-200.fc44.x86_64`、RT721 + RT1320、3.5mm 耳机系统测试及日常播放。
- DKMS 配置对 `7.2.x` / `7.3.x` 启用自动构建；只有 `7.2.11` 已在本机验证。后续更新需要对应的 `kernel-devel`，可用 `dkms status` 检查是否成功构建。
- `7.1.x` 及 `7.4+` 不启用本地覆盖，使用系统自带驱动；不代表 `7.4+` 已在本机验证正常。
- 本机 Secure Boot 关闭；启用 Secure Boot 的系统需要让内核信任 DKMS 模块签名后才能加载。
- 重启后的持久加载、拔插、扬声器切换及休眠恢复尚待后续验证。

**回退：**

```bash
sudo dkms remove -m xiaomi-rt721-fix -v 0.1 --all
# 完成后保存工作并重启，使用系统原驱动
```

源码来源、构建配置及归档包获取方法见 [audio/rt721-dkms/README.md](audio/rt721-dkms/README.md)。

## 修复 4：合盖休眠

**症状：** 合盖后无法触发休眠。`journalctl -b -k | grep PM:` 显示：

```
bitland_mifs_wmi: PM: dpm_run_callback(): bitland_mifs_wmi_suspend returns -22
PM: Some devices failed to suspend, or early wake event detected
```

**根因：** `bitland_mifs_wmi` 驱动（小米 WMI 平台驱动，负责风扇/键盘背光/性能模式等）的挂起回调有 bug，返回 `-EINVAL`，导致整个休眠流程失败。

**解决：** 创建 systemd-sleep hook，休眠前解绑该设备，唤醒后重新绑定。

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

**注意：** 仅 `s2idle`（默认）可用。`deep`（S3 深度休眠）因 BIOS bug 会睡死无法唤醒，切勿设置 `mem_sleep_default=deep`。

## 系统总结

**原测试环境：** Fedora 44, kernel 7.1.4-204.fc44.x86_64

**音频补充验证（2026-10-28）：** Fedora 44, kernel 7.2.11-200.fc44.x86_64；应用上述 RT721 DKMS 修复后，3.5mm 耳机系统测试和日常播放正常。

**已确认正常：**
- 屏幕显示（需 `xe.enable_psr=0`）
- 键盘（需 `i8042.dumbkbd`）
- 触摸板
- Wi-Fi / 蓝牙
- 扬声器 & 麦克风
- 指纹传感器
- 摄像头
- 睡眠/唤醒（仅 s2idle，需 workaround）
