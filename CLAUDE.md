# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Install

```bash
make -j$(nproc)               # build mt7902e.ko
sudo make install              # install module + depmod
sudo make install_fw           # install firmware .bin
sudo modprobe mt7902e disable_rpm=1
```

### DKMS
```bash
sudo dkms add .
sudo dkms install mt7902e/git
```

### Fedora post-install
```bash
echo "blacklist mt7921e" | sudo tee /etc/modprobe.d/disable-mt76.conf
sudo dracut -f --kver $(uname -r)
```

### Uninstall
```bash
sudo make uninstall
sudo make uninstall_fw
```

## Kernel Version Compatibility

| Kernel | Status |
|--------|--------|
| ≤ 6.5 | Not supported upstream |
| 6.6 – 7.0 | **Requires this driver** — PCI ID `14c3:7902` missing from in-tree `mt7921e` |
| 7.1-rc1+ | PCI ID natively recognized, but RPM fix still beneficial |

kernel 7.1+ broke `struct ieee80211_mgmt` — removed nested union `.u` from `u.action`, moved `action_code` to `u.action.action_code`. Use `#if LINUX_VERSION_CODE >= KERNEL_VERSION(7, 1, 0)` guards for access.

## Architecture

4-layer hierarchy (all sources in `src/`):

```
mac80211 ops (mt7902/main.c)  ← .start, .stop, .sta_state, .hw_scan, etc.
  └─ chip-specific (mt7902/pci.c, pci_mac.c, pci_mcu.c, init.c, mac.c, mcu.c)
       └─ MT792x shared (mt792x_core.c, mt792x_mac.c, mt792x_dma.c)
            └─ mt76 core (mmio.c, dma.c, tx.c, mac80211.c, mt76.h)
```

Chip differences vs MT7921: TXQ idx 15 (not 12), no MCU-WA ring, 512-entry RX ring, 2 IRQ status regs (not 3), aggressive PCI RPM.

## Our Patch

`mt7902_pci_rpm.patch` — applied to `src/mt7902/pci.c`:
- `disable_rpm` (bool): `pci_disable_link_state()` + `pm_runtime_forbid()` → stays in L0, no latency
- `rpm_state` (int): 0=default, 1=auto, 2=force off
- Zeroes `wm2_complete_mask` on MT7902 (prevent spurious IRQs)
- Fixes uninitialized `irq_map` pointer

## CI

`.github/workflows/c-cpp.yml` builds the module on every push. Runs on `ubuntu-latest` with kernel headers installed via apt.
