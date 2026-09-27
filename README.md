# MediaTek MT7927 (MT6639) Bluetooth-Only DKMS Driver

Custom packaging for enabling the Bluetooth controller (`0489:e13a`) on the **ASUS ROG Crosshair X870E Hero** and companion MediaTek MT7927 Wi-Fi 7 / Bluetooth 5.4 modules on Arch Linux / CachyOS.

This setup builds **only** the `btusb` and `btmtk` out-of-tree kernel modules via DKMS, leaving the native, stable in-tree `mt7925e` PCIe Wi-Fi driver untouched.

---

## 1. Prerequisites

Ensure base development tools, DKMS, and your kernel headers are installed:

```bash
sudo pacman -S --needed base-devel git dkms linux-cachyos-headers
```
*(Replace `linux-cachyos-headers` with `linux-headers` or `linux-zen-headers` if using another kernel).*

---

## 2. Installation & Build

Clone this repository and compile the package:

```bash
git clone [https://github.com/OmegaAlpha/mediatek-mt7927-dkms.git](https://github.com/OmegaAlpha/mediatek-mt7927-dkms.git)
cd mediatek-mt7927-dkms
makepkg --skipinteg -si
```

---

## 3. Persistent Firmware Symlink

The firmware blob `BT_RAM_CODE_MT6639_2_1_hdr.bin` installs to `/usr/lib/firmware/mediatek/mt7927/`, but some kernel driver lookups check directly in `/usr/lib/firmware/mediatek/`. 

To ensure this persists across `linux-firmware` updates, configure a systemd `tmpfiles.d` rule:

```bash
echo "L+ /usr/lib/firmware/mediatek/BT_RAM_CODE_MT6639_2_1_hdr.bin - - - - /usr/lib/firmware/mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin" | sudo tee /etc/tmpfiles.d/mediatek-bt-firmware.conf
sudo systemd-tmpfiles --create
```

---

## 4. Verification

After installation, verify that the controller is recognized and active:

```bash
# Verify USB enumeration (ID 0489:e13a)
lsusb | grep -i "0489"

# Verify Bluetooth radio state
rfkill list

# Inspect controller status
bluetoothctl show
```

---

## 5. Troubleshooting: USB State Lockup (Cold Capacitor Drain)

On modern AMD AM5 motherboards, the Bluetooth controller connects through an internal USB bus (Bus 8, Port 6) on the chipset. If a driver crash, hard hang, or improper firmware handshake occurs, the controller onboard microcontroller can enter a fault state and become unresponsive to USB descriptors (`error -110` / `error -71` in `dmesg`).

Because motherboards supply continuous **+3.3VSB (standby power)** to the M.2 slot even while shut down or rebooting, standard soft reboots **will not clear this state**.

### Hardware Reset Procedure:
1. Shut down the system: `sudo poweroff`
2. **Switch the physical power supply (PSU) rocker switch to OFF** (or disconnect the AC power cord).
3. **Press and hold the case power button for 10–15 seconds** to completely discharge residual energy in the motherboard capacitors.
4. Flip the PSU rocker back to **ON** and power on the system.
