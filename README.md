# mt7902-linux-wifi-driver

**Out-of-tree Linux Wi-Fi driver for MediaTek MT7902 (Filogic 310) — PCI ID `14c3:7902`**

This is a *patch overlay* of [hmtheboy154/mt7902](https://github.com/hmtheboy154/mt7902) (branch `backport`) with a **PCI Runtime Power Management (RPM) fix** that eliminates latency spikes on Fedora, Kali Linux, and other distributions.

---

### The Problem

**1. Unrecognized PCI ID (kernel < 7.1)**

The MT7902 chip (`14c3:7902`) is missing from the `mt7921e` driver's PCI device table on kernels prior to 7.1-rc1. The device shows up in `lspci` but no driver claims it.

**2. PCI Runtime PM → 50–300ms latency spikes**

Even when the driver loads, the chip's **PCI Express Runtime PM** (ASPM L1/D3hot) is overly aggressive. After a few seconds of idle traffic, the PCI link drops to a low-power state. When a Wi-Fi packet suddenly needs to be sent:

1. PCI link must wake from D3hot → L0
2. MT7902 firmware needs to resynchronize
3. This takes **50–300ms** — noticeable lag in SSH, video calls, gaming; can even drop the connection

---

### The Solution

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

### Driver Architecture

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

### Kernel Compatibility

| Kernel | Status |
|--------|--------|
| ≤ 6.5 | Not supported upstream |
| 6.6 – 7.0 | **Requires this driver** |
| 7.1-rc1+ | PCI ID present in in-tree `mt7921e`, but RPM fix still beneficial |

---

### Installation

#### Prerequisites

```bash
# Fedora
sudo dnf install kernel-headers kernel-devel gcc make

# Debian/Ubuntu
sudo apt install linux-headers-$(uname -r) build-essential

# Arch
sudo pacman -S linux-headers base-devel
```

#### Build & Install

```bash
git clone https://github.com/KoJYy/mt7902-linux-wifi-driver
cd mt7902-linux-wifi-driver
make -j$(nproc)
sudo make install
sudo make install_fw
sudo modprobe mt7902e disable_rpm=1
```

#### Fedora — Post-Install

```bash
echo "blacklist mt7921e" | sudo tee /etc/modprobe.d/disable-mt76.conf
sudo dracut -f --kver $(uname -r)
```

#### DKMS

```bash
sudo dkms add .
sudo dkms install mt7902e/git
```

---

### Module Parameters

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

### Uninstall

```bash
sudo make uninstall
sudo make uninstall_fw
sudo rm /etc/modprobe.d/disable-mt76.conf
sudo dracut -f --kver $(uname -r)
```

---

### Credits

- **MediaTek / Sean Wang** — Upstream MT7902 patches + firmware
- **Lorenzo Bianconi & Felix Fietkau** — mt76 framework maintainers
- **hmtheboy154** — Backport tree for kernel 6.6+
- **KoJYy** — PCI runtime PM fix

Upstream driver: [hmtheboy154/mt7902](https://github.com/hmtheboy154/mt7902)
