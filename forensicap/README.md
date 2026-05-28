# ForensicAP Kernel Overlay

This directory contains the **only** ForensicAP-specific additions on top of
the upstream `raspberrypi/linux` kernel.  Everything else in this fork stays
a clean mirror of `rpi-6.12.y` so upstream rebases remain trivial.

## Files

| File | Purpose |
|---|---|
| `slim.config` | Shared subsystem-exclusion overlay applied to both variants. Disables printers, audio, GPU acceleration, camera sensors, tape/optical drives, amateur radio, CAN bus, and exotic filesystems. Also disables in-kernel WiFi chipset drivers — ForensicAP uses DKMS packages for reliable monitor-mode and packet-injection support instead. |
| `v8.config` | Variant overlay merged on top of `bcm2711_defconfig` (Pi 4 / CM4 / Zero 2 W) |
| `v8-16k.config` | Variant overlay merged on top of `bcm2712_defconfig` (Pi 5 / CM5) |
| `version` | Monotonically incremented integer (`1`, `2`, …). Bumped when the overlay changes, *not* when upstream advances. Used as the `KDEB_PKGVERSION` suffix. |

## Overlay layering

```
bcmXXXX_defconfig
  └─ forensicap/slim.config    (subsystem exclusions, shared by both variants)
      └─ forensicap/v8.config  (variant-specific options + regulatory)
      └─ forensicap/v8-16k.config
```

## WiFi and DKMS

The slim overlay disables in-kernel Realtek (RTW88/RTW89/RTL8XXXU), Ralink
(RT2X00), MediaTek (MT7921U), Marvell (MWIFIEX), Intel (IWLWIFI), and Atheros
(ATH9K/ATH9K_HTC) WiFi drivers.  These are replaced by DKMS out-of-tree
packages on the ForensicAP image, which provide full monitor-mode and
packet-injection support:

| DKMS package | Chipsets | Representative adapter |
|---|---|---|
| `rtl88xxau` (aircrack-ng) | RTL8812AU, RTL8821AU | Alfa AWUS036ACH |
| `rtl8812bu` | RTL8812BU, RTL8822BU | Comfast CF-812AC |
| `mt7612u` | MT7612U | Alfa AWUS036ACHM |
| `rtl8814au` | RTL8814AU | Alfa AWUS1900 |

`CFG80211`, `MAC80211`, and `RFKILL` remain compiled in — DKMS drivers register
against these framework symbols.  `BRCMFMAC` is retained for the onboard Pi
WiFi chip.  The `linux-headers-*.deb` package (built by CI) must be installed
on the target system so DKMS can compile against the running kernel.

## Build

The CI workflow `.github/workflows/forensicap-release.yml` produces the
official ForensicAP kernel `.deb` packages.  Triggered by:

- `workflow_dispatch` (manual via GitHub UI)
- Tag push matching `kernel-v*` (e.g. `kernel-v6.12.90-forensicap.1`)

Output: three `.deb` packages per variant, attached to the release:

```
linux-image-<kver>_<pkgver>_arm64.deb
linux-headers-<kver>_<pkgver>_arm64.deb
linux-libc-dev_<pkgver>_arm64.deb
```

Consumed by [`ForensicAP/forensicap-image`](https://github.com/ForensicShark/ForensicAP)
via the `FORENSICAP_KERNEL_RELEASE_TAG` config key in `forensicap-image/config`.

## Local build

To reproduce a release build locally (e.g. for testing an overlay change
before tagging):

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
scripts/kconfig/merge_config.sh -m .config \
    forensicap/slim.config \
    forensicap/v8.config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    KDEB_PKGVERSION="$(cat forensicap/version).0" \
    -j"$(nproc)" bindeb-pkg
```

Resulting `linux-*.deb` files appear in the parent directory.

## Why config overlays (and not kernel-source patches)

All ForensicAP customisations are pure Kconfig changes — no code modifications.
Keeping them as config fragments means:

- Trivial upstream rebases — no merge conflicts anywhere in the source tree
- Reviewable in one place (this directory)
- Easy to test alternative settings without rebuilding patches
