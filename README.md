# Hisense 65U8NQ (MTK9618) — USB Firmware Recovery Guide

**Problem:** TV shows black screen / boot loop after VIDAA app corruption.  
**Solution:** USB firmware recovery using `Forcedupgrade_001.bin`.  
**Confirmed working on:** Hisense 65U8NQ (65U85LEVS, project 76931, SoC MT9618, firmware V0000.09.61B / E006)

---

## Root Cause

The TV crashes because `/APPS/pkgs/tv.vidaa.app.ui/packageinfo.json` is corrupt or empty.  
This kills VIDAAcommon → black screen, no menu, remote only works for power.

The TV **is not fully bricked** — bootloader and kernel load fine. Only the app layer is broken.

---

## What You'll Need

- USB flash drive, **≤32 GB**, formatted **FAT32, MBR partition table** (not GPT, not exFAT)
- The correct firmware file for your model (see below)
- A Mac or Linux PC to prepare the drive

---

## Step 1 — Get the Firmware

Download the firmware package for your exact model from Hisense or a mirror:

| Field | Value |
|-------|-------|
| Model | 65U8NQ / 65U85LEVS |
| Project | 76931 |
| SoC | MT9618 |
| File inside ZIP | `usb_MTK_9618_EU_G.bin` |
| Package | `76931_65U85LEVS_Hisense_E006_O0410.zip` (~854 MB) |

The ZIP contains a `.tar.gz` which contains the `.bin` file.

---

## Step 2 — Prepare the USB Drive

On macOS (replace `diskN` with your actual disk number from `diskutil list`):

```bash
# Format the drive
diskutil eraseDisk FAT32 HISENSE MBRFormat /dev/diskN

# Disable Spotlight indexing
sudo mdutil -i off /Volumes/HISENSE
touch /Volumes/HISENSE/.metadata_never_index

# Extract firmware directly to USB
unzip -p ~/Downloads/76931_65U85LEVS_Hisense_E006_O0410.zip \
  | tar -xzOf - usb_MTK_9618_EU_G.bin \
  > /Volumes/HISENSE/usb_MTK_9618_EU_G.bin

# Verify size (must be exactly 894841488 bytes)
stat -f %z /Volumes/HISENSE/usb_MTK_9618_EU_G.bin

# CRITICAL: rename the file — the bootloader requires this exact name
mv /Volumes/HISENSE/usb_MTK_9618_EU_G.bin /Volumes/HISENSE/Forcedupgrade_001.bin

# Eject
sync && diskutil eject /dev/diskN
```

> **Why rename?** The bootloader on this model looks for `Forcedupgrade_001.bin`, not the original filename. Without renaming, the TV boots normally and ignores the USB drive entirely.

---

## Step 3 — Flash the TV

1. Insert the USB drive into the **left USB port** (USB 2.0, not the blue USB 3.0)
2. Power on the TV normally (remote or physical button)
3. The TV will show a **blue screen: "USB Upgrade — Checking."** with a progress bar
4. Wait **10–20 minutes** — do NOT power off, do NOT remove the USB
5. TV reboots automatically → Hisense logo → first-time setup wizard
6. Remove the USB drive after the logo appears

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| TV ignores USB, boots to broken screen | Check filename is exactly `Forcedupgrade_001.bin` |
| TV ignores USB with correct name | Try the other USB port; try a different ≤16 GB drive |
| File isn't found in TAR | Run `unzip -p file.zip \| tar -tzf -` to list contents |
| SHA256 mismatch | Re-download the firmware package |

---

## What Doesn't Work (and Why)

### USB with original filename (`usb_MTK_9618_EU_G.bin`)
The bootloader silently ignores it. No error, no message — TV just boots normally.

### Physical button hold on TV body
On this model, the physical button does not trigger a USB recovery mode via hold-on-power-on. It simply boots the TV directly (bypassing the Hisense logo on some firmware versions).

### UART via HDMI DDC pins (15/16)
UART is disabled in production firmware on MTK9618. No data on DDC Clock (pin 15) or DDC SDA (pin 16) at any baud rate (tested: 921600, 460800, 230400, 115200, 57600).

### MQTT commands
Port 36669 (MQTT) is open but VIDAAcommon is dead when `packageinfo.json` is corrupt — broker accepts connections but all commands are ignored.

### DNS spoof / OTA recovery
TV makes zero TCP connections when VIDAAcommon is not running. OTA update service never starts.

---

## Technical Details

### TV identification
```
Rear label:    65U8NQ
Internal name: 65U85LEVS
Project:       76931
SoC:           MediaTek MT9618 (Pentonic 1000)
VIDAA:         6.x (Tizen-based VIDAA OS)
```

### eMMC partition layout (MTK9618, 42 partitions)
```
p20:  rootfs squashfs ~36 MB  (read-only)
p37:  /usr/local       245 MB
p38:  /opt             665 MB  (MQTT broker at /opt/iot/mosquitto)
p39:  /var/local       348 MB  (writable, boot hooks)
p41:  /APPS            614 MB  (ext4 — corrupt packageinfo.json lives here)
p42:  /OAD             921 MB  (OTA staging — DO NOT TOUCH)
```

### The broken file
```
/APPS/pkgs/tv.vidaa.app.ui/packageinfo.json
```
When this file is empty or corrupt, the VIDAA app manager cannot register the main UI package.  
VIDAAcommon fails to start → no ports open → no network recovery possible.

---

## If USB Recovery Still Fails

1. **Try a smaller USB drive** (8–16 GB recommended, some bootloaders reject large FAT32 volumes)
2. **UART via mainboard test pads** — 3-pad group near the MT9618 SoC, 115200 baud 8N1 3.3V. Use U-boot to directly edit `/APPS/pkgs/tv.vidaa.app.ui/packageinfo.json`
3. **eMMC ISP** — phone repair shops with Easy-JTAG / UFI box can read/write eMMC in-circuit without desoldering (~800–2500 UAH / $20–60 USD)

---

## Contributing

If you recovered a different Hisense model using this method, please open a PR with:
- TV model and internal name
- Firmware filename that worked
- Which USB port worked
- Any differences from the steps above
