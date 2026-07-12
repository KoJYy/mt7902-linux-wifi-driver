# mt7902-linux-wifi-driver

**Driver Wi-Fi out-of-tree untuk MediaTek MT7902 (Filogic 310) — PCI ID `14c3:7902`**

Driver ini adalah *patch overlay* dari [hmtheboy154/mt7902](https://github.com/hmtheboy154/mt7902) (branch `backport`) dengan tambahan **fix PCI Runtime Power Management (RPM)** yang menyebabkan latency spike pada Fedora, Kali Linux, dan distro lain.

---

### Masalah

**1. PCI ID tidak dikenal (kernel < 7.1)**

Chip MT7902 (`14c3:7902`) tidak ada di tabel PCI driver `mt7921e` sebelum kernel 7.1-rc1. Kernel tidak bisa mengikat driver ke perangkat — terdeteksi di `lspci` tapi tidak ada modul yang mengklaimnya.

**2. PCI Runtime PM → latency spike 50–300ms**

Bahkan saat driver termuat, mekanisme **PCI Express Runtime PM** (ASPM L1/D3hot) bawaan chip sangat agresif. Setelah idle beberapa detik, PCI link turun ke daya rendah. Saat tiba-tiba ada paket Wi-Fi:
1. PCI link harus *wake up* dari D3hot ke L0
2. Firmware MT7902 perlu sinkronisasi ulang
3. Butuh **50–300ms** — latency parah di SSH, video call, game

---

### Solusi

Menambahkan **parameter modul** `disable_rpm` dan `rpm_state`:

```
disable_rpm=1  → PCI link tetap L0 (aktif) — tanpa latency spike
rpm_state=2    → Runtime PM dimatikan paksa
```

Saat aktif, driver panggil:
```c
pci_disable_link_state(pdev, PCIE_LINK_STATE_L1);
pm_runtime_dont_use_autosuspend(&pdev->dev);
pm_runtime_forbid(&pdev->dev);
```

Perubahan lain:
- **Zeroing `wm2_complete_mask`** — mencegah interrupt palsu (MT7902 hanya punya 2 IRQ status register)
- **Fix uninitialized pointer** `struct mt792x_irq_map *map = NULL`

---

### Arsitektur Driver

4 lapisan:
```
mac80211 ops (src/mt7902/main.c)
  └─ chip-specific (pci.c, pci_mac.c, pci_mcu.c, init.c, mcu.c)
       └─ MT792x shared (src/mt792x_core.c, mt792x_mac.c, mt792x_dma.c)
            └─ mt76 core (src/mmio.c, dma.c, tx.c, mac80211.c)
```

Perbedaan hardware MT7902 vs MT7921:

| Aspek | MT7921 | MT7902 |
|-------|--------|--------|
| MCU-WM TXQ index | 12 | **15** |
| MCU-WA ring | Ada | **Tidak ada** |
| RX event ring | 8 entry | **512 entry** |
| Register IRQ status | 3 | **2** |
| Perilaku PCI RPM | Stabil | **Agresif → spike** |

---

### Kompatibilitas Kernel

| Kernel | Status |
|--------|--------|
| ≤ 6.5 | Tidak didukung upstream |
| 6.6 – 7.0 | **Butuh driver ini** |
| 7.1-rc1+ | PCI ID ada di `mt7921e`, tapi RPM fix tetap berguna |

---

### Instalasi

#### Prasyarat

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

### Parameter Modul

| Parameter | Tipe | Default | Deskripsi |
|-----------|------|---------|-----------|
| `disable_rpm` | bool | 0 | Matikan PCI runtime PM (hilangkan latency) |
| `rpm_state` | int | 0 | 0=default, 1=auto, 2=paksa mati |
| `disable_aspm` | bool | 0 | Matikan PCI ASPM |
| `disable_clc` | bool | 0 | Matikan CLC |

Contoh runtime:
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

### Kredit

- **MediaTek / Sean Wang** — Patch upstream MT7902 + firmware
- **Lorenzo Bianconi & Felix Fietkau** — Pengelola mt76 framework
- **hmtheboy154** — Backport tree untuk kernel 6.6+
- **KoJYy** — Patch PCI runtime PM fix

Driver asli: [hmtheboy154/mt7902](https://github.com/hmtheboy154/mt7902)
