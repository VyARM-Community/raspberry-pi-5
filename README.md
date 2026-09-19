# VyOS for Raspberry Pi 5

> Unofficial community build of **VyOS Rolling** for the **Raspberry Pi 5**.

[![GitHub Release](https://img.shields.io/github/v/release/VyARM-Community/raspberry-pi-5?style=for-the-badge)](https://github.com/VyARM-Community/raspberry-pi-5/releases)
[![GitHub Downloads](https://img.shields.io/github/downloads/VyARM-Community/raspberry-pi-5/total?style=for-the-badge)](https://github.com/VyARM-Community/raspberry-pi-5/releases)
[![Build Status](https://img.shields.io/github/actions/workflow/status/frogro/vyos-arm64-board-builder/build-board-candidate.yml?branch=main-test&style=for-the-badge)](https://github.com/frogro/vyos-arm64-board-builder/actions/workflows/build-board-candidate.yml)
[![Donate with PayPal](https://img.shields.io/badge/Donate-PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/FGrootens)


## Table of Contents

- [Quick Start](#quick-start)
- [First Boot and Login](#first-boot-and-login)
- [Optional Helper Scripts](#optional-helper-scripts)
- [Supported Hardware](#supported-hardware)
- [Updates](#updates)
- [Releases](#releases)
- [Changelog](CHANGELOG.md)
- [Contributing](CONTRIBUTING.md)
- [Security](SECURITY.md)
- [License and Trademarks](#license-and-trademarks)
- [Support the Project](#️-support-the-project)

This repository provides an unofficial VyOS image for the Raspberry Pi 5 by combining:

- the VyOS ARM64 userspace and configuration system;
- a board-adapted VyOS kernel, modules and firmware, with native Raspberry Pi firmware, kernel and Device Tree loaded from the FAT boot partition;
- first-boot networking helpers and additional network, Wi-Fi and cellular modem drivers and firmware.

**Upstream attribution:** VyOS provides the routing userspace and configuration framework; Armbian provides board integration references; Raspberry Pi provides the native firmware and designs and documents the hardware.

> [!WARNING]
> This is an independent community project built from components provided by **VyOS**, **Armbian**, and **Raspberry Pi** ecosystems. It is not produced, supported, sponsored, certified, or endorsed by the VyOS project, Sentrium S.L., Armbian, Armbian d.o.o., Raspberry Pi, or Raspberry Pi Ltd. Rolling releases may contain regressions and should be tested before production use.

## Quick Start

### 1. Download the ready-to-use image

Open the [Releases page](https://github.com/VyARM-Community/raspberry-pi-5/releases) and download:

```text
vyos-VERSION-raspberry-pi-5-network.img.xz
vyos-VERSION-raspberry-pi-5-network.img.xz.sha256
```

Verify the download on Linux:

```bash
sha256sum -c vyos-VERSION-raspberry-pi-5-network.img.xz.sha256
```

Use a target drive larger than the uncompressed image and a boot medium supported by the Raspberry Pi 5. Replace `VERSION` in all examples with the version from the selected release.

### 2. Flash with balenaEtcher

[balenaEtcher](https://etcher.balena.io/) is available for Linux, Windows, and macOS.

1. Start balenaEtcher.
2. Select `vyos-VERSION-raspberry-pi-5-network.img.xz` directly.
3. Select the target boot medium.
4. Click **Flash**.
5. Wait for flashing and verification to finish.

> [!CAUTION]
> Flashing destroys all data on the selected target drive. Verify the destination carefully.

### 3. Flash from Linux with `dd`

Replace `/dev/sdX` with the complete target device, not a partition such as `/dev/sdX1`.

#### Option A: write the compressed image directly

```bash
sudo umount /dev/sdX?* 2>/dev/null || true
xz -dc vyos-VERSION-raspberry-pi-5-network.img.xz | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
sync
sudo eject /dev/sdX
```

#### Option B: extract first, then write

This may be faster on slower systems because decompression and writing do not occur simultaneously.

```bash
xz -dk vyos-VERSION-raspberry-pi-5-network.img.xz
sudo umount /dev/sdX?* 2>/dev/null || true
sudo dd if=vyos-VERSION-raspberry-pi-5-network.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
sudo eject /dev/sdX
```

---

## First Boot and Login

1. Connect the Raspberry Pi 5 Ethernet port to a network that provides DHCP.
2. Insert or attach the flashed boot drive.
3. Power on the Raspberry Pi 5.
4. Allow approximately 60–90 seconds for first-boot configuration.
5. Find the assigned address in your router or DHCP server.
6. Connect over SSH.

Default credentials for this image:

```text
Username: vyos
Password: vyos
```

Example:

```bash
ssh vyos@192.168.1.100
```

Replace the example address with the address assigned to your Raspberry Pi 5.

> [!IMPORTANT]
> Change the default password immediately after the first login. For stronger security, configure SSH key authentication and stop using password-based login.

Change the password:

```text
configure
set system login user vyos authentication plaintext-password 'YOUR_NEW_PASSWORD'
commit
save
exit
```

Check Ethernet locally from the HDMI or serial console:

```bash
ip -4 -br addr show eth0
```

<details>
<summary><strong>First-boot diagnostics</strong></summary>

```bash
cat /config/dhcp-wan-firstboot-wrapper.log
cat /config/dhcp-wan-ssh-setup.log
systemctl status dhclient@eth0.service --no-pager -l
```

The first-boot marker is:

```text
/config/.dhcp-wan-ssh-firstboot-done
```

</details>

---

## Optional Helper Scripts

The image includes helper scripts under `/usr/local/share/vyos-arm64-firstboot/`, with convenience links in `/home/vyos`. Start from the `vyos` account and use the command shown for each helper below.

### Configure locale, time and regional settings

Run this helper as the `vyos` user, without `sudo`:

```bash
/home/vyos/set-locales.sh
```

It guides you through the time zone, console keyboard layout, wireless regulatory country, DNS servers and NTP servers. It also persists the system locale as `C.UTF-8`, restarts Chrony and checks time synchronization. Review the proposed settings before applying them. Committing wireless settings may briefly restart an active access point.

### Configure a wireless access point

Run this helper as the `vyos` user, **without `sudo`**. The script invokes `sudo` internally where required.

```bash
/home/vyos/ap-dhcp-wan-setup.sh
```

This is separate from the automatic wired DHCP and SSH setup. Run it only when an access point, DHCP server, DNS forwarding, and NAT are required.

### Configure a modem

Run this helper from the `vyos` account **with `sudo`**. Unlike the AP and locale helpers, the modem setup script explicitly requires root privileges.

```bash
sudo /home/vyos/modem-connect.sh
```

Modem support depends on the modem, transport, drivers, firmware, carrier, and APN.

---

## Supported Hardware

Peripheral results below are project-level references, not confirmation that every listed device has been tested on this exact Raspberry Pi 5 image. Consult the release notes for board-specific validation.

### Wi-Fi adapters

`ap-dhcp-wan-setup.sh` does not hard-code a specific chipset. It enumerates every `phy` under `/sys/class/ieee80211`, reads supported interface modes from the kernel with `iw phy <phy> info`, and only offers devices that report AP mode support.

#### Hardware reported working with the project

- MediaTek MT7921-class M.2/PCIe Wi-Fi 6 adapters
- Realtek RTL8852-class M.2/PCIe Wi-Fi 6 adapters
- MediaTek MT7612U-based USB adapters

#### Expected to work

- Other Linux `mac80211` adapters that advertise AP mode

Adapters whose drivers only support client or station mode cannot be used by the AP helper.

Useful diagnostics:

```bash
ip link show
iw dev
dmesg
```

### Cellular modems

`modem-connect.sh` supports PCIe- and USB-attached modems, preferring native VyOS WWAN configuration where supported and using helpers for device-specific initialization and the AT/RNDIS fallback path.

#### Tested with the project

- Fibocom FM350-GL (Revision: 81600.0000.00.29.24.02,  SVN: 10), including automatic FCC unlock over the AT port

#### Expected to work with compatible drivers and firmware

- Quectel RM505Q
- Intel XMM7560-based modems
- Other QMI- or MBIM-capable modems supported by ModemManager

Actual connectivity also depends on the SIM carrier, APN, regional firmware, and supported bands.

---

## Updates

This Raspberry Pi 5 image currently supports **fresh installation and recovery using the `.img.xz` image**. Keep a configuration backup and your previous boot medium before testing a new image.

The Pi boots through its EEPROM/Boot ROM and native firmware, loading the kernel, initramfs and Device Tree from the FAT boot partition. Safe in-place updates require synchronization of those files with the selected VyOS system image. That update path is not yet released for this board.

Do not use `add system image latest` or the builder's experimental ISO for this image yet. No automatic update channel is configured. This section will be updated once the native update path has been implemented and hardware-tested.

---

## Releases

Prebuilt images are published on the [GitHub Releases page](https://github.com/VyARM-Community/raspberry-pi-5/releases).

Each release should normally contain:

```text
vyos-VERSION-raspberry-pi-5-network.img.xz
vyos-VERSION-raspberry-pi-5-network.img.xz.sha256
```

Use the compressed `.img.xz` for initial installation. Only initial-installation images are published here until the Pi update path is ready.

See [CHANGELOG.md](CHANGELOG.md) for the changes in each release.

The published VyOS Rolling is a reference; package versions may differ because the image is built later against the rolling package repository.

---

## License and Trademarks

The repository contains or builds software from multiple upstream projects. Their respective licenses remain in effect. Review the license and copyright files included in the repository and generated image.

“VyOS” and associated marks are trademarks of their respective owner. “Armbian” and associated marks are trademarks of Armbian d.o.o. “Raspberry Pi” and associated marks are trademarks of their respective owner.

These names are used solely to identify compatibility, upstream software, boot-chain and kernel components, and supported hardware. No affiliation, sponsorship, certification, or endorsement is claimed.

This repository does not redistribute third-party logo artwork.

---

## ❤️ Support the Project

If this project saved you time or made it easier to run VyOS on the Raspberry Pi 5, please consider supporting its development.

Contributions help cover hardware, testing, maintenance, and development time.

[![Donate with PayPal](https://img.shields.io/badge/Donate-PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/FGrootens)

Thank you for your support. ☕
