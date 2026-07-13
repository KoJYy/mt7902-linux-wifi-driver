# mt7902 linux wifi driver

[![License: GPL v2](https://img.shields.io/badge/License-GPL_v2-blue.svg)](./LICENSE)
[![Kernel](https://img.shields.io/badge/Kernel-6.6–7.1%2B-blueviolet)](https://kernel.org)
[![Platform](https://img.shields.io/badge/Platform-Linux%20(Fedora%20%7C%20Debian%20%7C%20Arch)-green)](https://github.com/KoJYy/mt7902-linux-wifi-driver)
[![PCI](https://img.shields.io/badge/PCI-14c3%3A7902-orange)](https://devicehunt.com/view/type/pci/vendor/14C3/device/7902)
[![WiFi](https://img.shields.io/badge/WiFi-6%20(802.11ax)-informational)](https://en.wikipedia.org/wiki/Wi-Fi_6)
[![GitHub release](https://img.shields.io/github/v/release/KoJYy/mt7902-linux-wifi-driver)](https://github.com/KoJYy/mt7902-linux-wifi-driver/releases)
[![CI](https://github.com/KoJYy/mt7902-linux-wifi-driver/actions/workflows/c-cpp.yml/badge.svg)](https://github.com/KoJYy/mt7902-linux-wifi-driver/actions/workflows/c-cpp.yml)
[![GitHub stars](https://img.shields.io/github/stars/KoJYy/mt7902-linux-wifi-driver?style=social)](https://github.com/KoJYy/mt7902-linux-wifi-driver)

**Out-of-tree Linux Wi-Fi driver for MediaTek MT7902 (Filogic 310) — PCI ID `14c3:7902`**

This is a *patch overlay* of [hmtheboy154/mt7902](https://github.com/hmtheboy154/mt7902) (branch `backport`) with a **PCI Runtime Power Management (RPM) fix** that eliminates latency spikes on Fedora, Kali Linux, and other distributions.

---

## Table of Contents

- [Quick Start](#quick-start)
- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Driver Architecture](#driver-architecture)
- [Kernel Compatibility](#kernel-compatibility)
- [Installation](#installation)
- [Module Parameters](#module-parameters)
- [Troubleshooting](#troubleshooting)
- [Uninstall](#uninstall)
- [Credits](#credits)

---

## Quick Start

```bash
git clone https://github.com/KoJYy/mt7902-linux-wifi-driver
cd mt7902-linux-wifi-driver
make -j$(nproc)
sudo make install
sudo make install_fw
sudo modprobe mt7902e disable_rpm=1
```

```
# Verify it works
iw dev wlp2s0 link
ping -c 2 8.8.8.8
```

- **GitHub Actions CI** — automated build verification on every push. Note: CI runs on generic Ubuntu and performs a syntax/source build check only. Runtime testing (module loading, firmware, Wi-Fi association) requires matching kernel headers for your target kernel version.

---

## The Problem

### 1. Unrecognized PCI ID (kernel < 7.1)

The MT7902 chip (`14c3:7902`) is missing from the `mt7921e` driver's PCI device table on kernels prior to 7.1-rc1. The device shows up in `lspci` but no driver claims it.

### 2. PCI Runtime PM → 50–300ms latency spikes

Even when the driver loads, the chip's **PCI Express Runtime PM** (ASPM L1/D3hot) is overly aggressive. After a few seconds of idle traffic, the PCI link drops to a low-power state. When a Wi-Fi packet suddenly needs to be sent:

1. PCI link must wake from D3hot → L0
2. MT7902 firmware needs to resynchronize
3. This takes **50–300ms** — noticeable lag in SSH, video calls, gaming; can even drop the connection

---

## The Solution

We add **module parameters** `disable_rpm` and `rpm_state` to the `mt7902e` driver:

```
disable_rpm=1  → Forces PCI link to stay in L0 (active) — zero latency penalty
rpm_state=2    → Runtime PM forcibly disabled
```

When active, the driver calls:

```c
pci_disable_link_state(pdev, PCIE_LINK_STATE_L1);
pm_runtime_dont_use_autosuspend(&pdev->dev);
pm_runtime_forbid(&pdev->dev);
```

Additional fixes:

- **Zero `wm2_complete_mask`** on MT7902 — prevents spurious IRQs (this chip has only 2 IRQ status registers, not 3 like the MT7921)
- **Fix uninitialized pointer** `struct mt792x_irq_map *map = NULL`

---

## Driver Architecture

4-layer hierarchy inherited from MediaTek's mt76 driver family:

```
mac80211 ops (src/mt7902/main.c)
  └─ chip-specific (pci.c, pci_mac.c, pci_mcu.c, init.c, mcu.c)
       └─ MT792x shared (src/mt792x_core.c, mt792x_mac.c, mt792x_dma.c)
            └─ mt76 core (src/mmio.c, dma.c, tx.c, mac80211.c)
```

Hardware differences between MT7902 vs MT7921:

| Aspect | MT7921 | MT7902 |
|--------|--------|--------|
| MCU-WM TXQ index | 12 | **15** |
| MCU-WA ring | Present | **Absent** |
| RX event ring | 8 entries | **512 entries** |
| IRQ status registers | 3 | **2** |
| PCI RPM behavior | Stable | **Aggressive → spikes** |

---

## Kernel Compatibility

| Kernel | Status |
|--------|--------|
| ≤ 6.5 | Not supported upstream |
| 6.6 – 7.0 | **Requires this driver** |
| 7.1-rc1+ | PCI ID present in in-tree `mt7921e`, but RPM fix still beneficial |

---

## Installation

### Prerequisites

```bash
# Fedora
sudo dnf install kernel-headers kernel-devel gcc make

# Debian/Ubuntu
sudo apt install linux-headers-$(uname -r) build-essential

# Arch
sudo pacman -S linux-headers base-devel
```

### Build & Install

```bash
git clone https://github.com/KoJYy/mt7902-linux-wifi-driver
cd mt7902-linux-wifi-driver
make -j$(nproc)
sudo make install
sudo make install_fw
sudo modprobe mt7902e disable_rpm=1
```

### Fedora — Post-Install

```bash
echo "blacklist mt7921e" | sudo tee /etc/modprobe.d/disable-mt76.conf
sudo dracut -f --kver $(uname -r)
```

### DKMS (Recommended)

**Why DKMS?** Each kernel upgrade replaces your kernel headers and modules directory. Without DKMS, the `mt7902e` module silently disappears after an update — you'd have to rebuild and reinstall it manually. DKMS **automatically rebuilds** the driver for every new kernel you install.

```bash
# 1. Add this driver source to DKMS
sudo dkms add .

# 2. Build and install for the current kernel
sudo dkms install mt7902e/git

# 3. The module is now managed by DKMS.
#    Future kernel updates trigger an automatic rebuild.
#    You only need to re-run these commands after `git pull`.
```

To verify DKMS is tracking the module:

```bash
sudo dkms status | grep mt7902e
# Expected: mt7902e/git, ...: installed
```

> **Note:** After DKMS installation, the Fedora post-install steps (blacklist + dracut) are still required.

---

## Module Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `disable_rpm` | bool | 0 | Disable PCI runtime PM (eliminates latency) |
| `rpm_state` | int | 0 | 0=default, 1=allow, 2=force off |
| `disable_aspm` | bool | 0 | Disable PCI ASPM |
| `disable_clc` | bool | 0 | Disable CLC (Channel Location Certification) |

Runtime control:

```bash
sudo modprobe mt7902e disable_rpm=1
echo 2 | sudo tee /sys/module/mt7902e/parameters/rpm_state
```

---

## Troubleshooting

### Module fails to load — "Unknown symbol"

The in-tree `mt7921e` and out-of-tree `mt7902e` share kernel symbols. If both are loaded, you'll get duplicate symbol errors.

**Fix:** Blacklist the in-tree driver:

```bash
echo "blacklist mt7921e" | sudo tee /etc/modprobe.d/disable-mt76.conf
sudo dracut -f --kver $(uname -r)
```

### Wi-Fi interface not appearing after `modprobe`

Check `dmesg` for errors:

```bash
dmesg | grep mt79
```

If you see `Direct firmware load for mediatek/WIFI_RAM_CODE_MT7902_1.bin failed`, install firmware:

```bash
sudo make install_fw
```

### Latency spikes still present despite driver loaded

Verify the parameter took effect:

```bash
cat /sys/module/mt7902e/parameters/disable_rpm
# Should print Y
cat /sys/module/mt7902e/parameters/rpm_state
```

If `disable_rpm=0`, reload:

```bash
sudo modprobe -r mt7902e && sudo modprobe mt7902e disable_rpm=1
```

### Frequent disconnections

Try disabling ASPM as well:

```bash
sudo modprobe -r mt7902e
sudo modprobe mt7902e disable_rpm=1 disable_aspm=1
```

---

## Uninstall

```bash
sudo make uninstall
sudo make uninstall_fw
sudo rm /etc/modprobe.d/disable-mt76.conf
sudo dracut -f --kver $(uname -r)
```

---

## Credits

- **MediaTek / Sean Wang** — Upstream MT7902 patches + firmware
- **Lorenzo Bianconi & Felix Fietkau** — mt76 framework maintainers
- **hmtheboy154** — Backport tree for kernel 6.6+
- **KoJYy** — PCI runtime PM fix

Upstream driver: [hmtheboy154/mt7902](https://github.com/hmtheboy154/mt7902)
