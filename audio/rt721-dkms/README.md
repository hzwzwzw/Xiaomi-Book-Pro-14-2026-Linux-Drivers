# RT721 DKMS workaround

This directory contains the exact driver sources and build configuration used to restore 3.5mm headphone playback on a Xiaomi Book Pro 14 (2026), Fedora 44, kernel `7.2.11-200.fc44.x86_64`, on 2026-10-28. System sound tests and ordinary playback were confirmed; reboot persistence, jack reinsertion, speaker switching and suspend/resume were not yet verified.

Installation and rollback: [English](../../README.md#32-35mm-headphones-detected-but-silent-updated-2026-10-28) / [中文](../../README-cn.md).

## Source provenance

The following files are unchanged copies from Linux commit [`e2eb4873b901290adc56b76f1d599005e620a6de`](https://github.com/torvalds/linux/commit/e2eb4873b901290adc56b76f1d599005e620a6de):

- `rt721-sdca.c`
- `rt721-sdca.h`
- `rt721-sdca-sdw.c`
- `rt721-sdca-sdw.h`

This is the driver snapshot at the accepted Realtek patch **ASoC: rt721-sdca: Adjust blindwrite flow to fix type detection issue**, including its earlier upstream dependencies. It is not that patch applied in isolation to the older 7.2.11 driver. The patch does not apply cleanly to 7.2.11 by itself.

`rt-sdw-common.h` is unchanged from [Linux v7.2.11](https://github.com/gregkh/linux/blob/v7.2.11/sound/soc/codecs/rt-sdw-common.h). The kernel's existing `snd-soc-rt-sdw-common` supplies the corresponding symbols; this package builds only `snd-soc-rt721-sdca`.

`Makefile` and `dkms.conf` provide the local module build and DKMS registration. `AUTOINSTALL` is enabled only for `7.2.x` and `7.3.x`; other versions use the distribution driver. That restriction is a packaging choice, not a claim that every supported version has been tested or that newer kernels are known to work on this laptop.

The upstream source files retain their copyright notices and `GPL-2.0-only` SPDX identifiers. The Linux GPL-2.0 license text is included in `COPYING`; the local build files use the same license.

To verify the bundled files:

```bash
cd audio/rt721-dkms
sha256sum -c SHA256SUMS
```

## Matching Fedora development packages

`dkms` on Fedora depends on `kernel-devel-matched`. Installing it without a version constraint can pull in a different kernel and trigger Fedora's old-kernel retention limit. The main instructions request both development packages for the running kernel explicitly.

If they are no longer in the enabled repositories, obtain them from [official Fedora Koji storage](https://kojipkgs.fedoraproject.org/packages/kernel/). For Fedora **x86_64** kernels with a version such as `7.2.11-200.fc44.x86_64`:

```bash
kver="$(uname -r)"
version="${kver%%-*}"
release="${kver#*-}"
release="${release%.x86_64}"
pkgdir="$(mktemp -d)"
base_url="https://kojipkgs.fedoraproject.org/packages/kernel/${version}/${release}/x86_64"

curl -fL "${base_url}/kernel-devel-${kver}.rpm" \
  -o "${pkgdir}/kernel-devel.rpm"
curl -fL "${base_url}/kernel-devel-matched-${kver}.rpm" \
  -o "${pkgdir}/kernel-devel-matched.rpm"
# Inspect package identities and the transaction before accepting installation.
rpm -qp --qf '%{NAME} %{VERSION}-%{RELEASE} %{ARCH}\n' \
  "${pkgdir}/kernel-devel.rpm" "${pkgdir}/kernel-devel-matched.rpm"
sudo dnf install gcc make dkms \
  "${pkgdir}/kernel-devel.rpm" "${pkgdir}/kernel-devel-matched.rpm"
```

Both package versions must match `uname -r`; the matching `kernel-core` must already be installed. These two archive packages were used for the tested 7.2.11 system. Then continue with the source copy and `dkms add/build/install` steps in the main README.

## Operational notes

- DKMS installation preserves the distribution module and installs an override. On the tested Fedora installation, `modinfo -F filename snd_soc_rt721_sdca` resolves to `.../extra/snd-soc-rt721-sdca.ko.xz`.
- Fedora's DKMS post-transaction hook regenerated the initramfs during installation; allow it to finish. No manual driver-file replacement or kernel boot argument is required.
- Kernel updates need matching development packages. `kernel-devel-matched` and DKMS's kernel-install hook provide the normal Fedora integration, but a failed build still needs attention; check `dkms status` after updates.
- Secure Boot was disabled during testing. DKMS signed the installed module with its generated key; enabling Secure Boot requires enrolling/trusting the key separately.
- Avoid unloading only the RT721 codec on a running system. Partial re-probes in this driver stack can lose the internal microphone route; use a normal reboot to activate or roll back the module.
