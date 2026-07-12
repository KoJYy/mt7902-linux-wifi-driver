# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

This repo is a **patch overlay** over the upstream MediaTek MT7902 out-of-tree driver. The actual driver source lives at:

https://github.com/hmtheboy154/mt7902 (branch: `backport`)

This repo contains:
- `mt7902_pci_rpm.patch` — adds PCI runtime PM control module parameters (`disable_rpm`, `rpm_state`) to fix latency spikes on Fedora
- No driver source files — they come from the upstream repo

## Build (requires upstream source cloned)

```bash
# Clone upstream
git clone https://github.com/hmtheboy154/mt7902 /tmp/mt7902
cd /tmp/mt7902

# Apply our patch (if applicable — upstream may already include RPM fix)
# git am /path/to/mt7902_pci_rpm.patch

# Build driver
make -j$(nproc)

# Install driver + firmware
sudo make install
sudo make install_fw

# Post-install (Fedora)
# Blacklist in-tree mt7921e to avoid symbol conflict
echo "blacklist mt7921e" | sudo tee /etc/modprobe.d/disable-mt76.conf
sudo dracut -f --kver $(uname -r)
```

### Module parameters (added by our patch)
```bash
# Check/disable PCI runtime PM
sudo modprobe mt7902e disable_rpm=1
# Runtime control
echo 2 | sudo tee /sys/module/mt7902e/parameters/rpm_state
```

## Our patch

`mt7902_pci_rpm.patch` — fixes PCI runtime PM causing 50–300ms latency spikes on MT7902 by:

1. Adding `disable_rpm` bool param → calls `pci_disable_link_state()` + `pm_runtime_forbid()`
2. Adding `rpm_state` int param → runtime override (0=default, 1=auto, 2=off)
3. Zeroing `wm2_complete_mask` on MT7902 to prevent spurious IRQs
4. Fixing uninitialized `struct mt792x_irq_map *map = NULL`

## Kernel version compatibility

| Kernel | Status |
|--------|--------|
| ≤ 6.6 | Not supported by upstream |
| 6.6 – 7.0 | Needs this out-of-tree driver + patch |
| ≥ 7.1-rc1 | PCI ID `14c3:7902` natively in `mt7921e`, but RPM fix still beneficial |

## Architecture (upstream driver)

4-layer hierarchy:
1. **mac80211 ops** → `src/mt7902/main.c` (`.start`, `.stop`, `.sta_state`, `.hw_scan`...)
2. **Chip-specific** → `src/mt7902/pci.c` (probe/remove), `pci_mac.c`, `pci_mcu.c`, `init.c`, `mac.c`, `mcu.c`
3. **MT792x shared** → `src/mt792x_core.c`, `mt792x_mac.c`, `mt792x_dma.c`
4. **mt76 core** → `src/mmio.c`, `dma.c`, `tx.c`, `mac80211.c`

Key hardware diffs MT7902 vs MT7921: MCU-WM TXQ at index 15 (not 12), no MCU-WA ring, 512-entry RX ring, only 2 IRQ status regs.
