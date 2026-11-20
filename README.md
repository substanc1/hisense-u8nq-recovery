# Hisense 65U8NQ — USB Firmware Recovery

> 🇺🇦 [Українська](#українська) | 🇷🇺 [Русский](#русский) | 🇬🇧 [English](#english)

---

## English

### The Problem

The TV shows a **black screen** after booting (or a boot loop). The remote only works for power on/off. No menu, no picture.

**Root cause:** `/APPS/pkgs/tv.vidaa.app.ui/packageinfo.json` is corrupt or empty.  
This kills VIDAAcommon → app manager can't register the UI → black screen.

The TV is **NOT fully bricked** — bootloader, kernel and init all load fine. Only the app layer is broken. This is fully recoverable at home with a USB drive.

---

### TV Specs

| Field | Value |
|-------|-------|
| Rear label | 65U8NQ |
| Internal model | 65U85LEVS |
| Project | 76931 |
| SoC | MediaTek MT9618 (Pentonic 1000) |
| OS | VIDAA 6.x |
| Firmware tested | V0000.09.61B / E006 (O0410) |

---

### The Fix — USB Firmware Recovery

#### What you need
- USB flash drive, **≤32 GB** (8–16 GB ideal), USB 2.0 type
- Formatted **FAT32, MBR** (not GPT, not exFAT, not NTFS)
- The correct firmware package (see below)

#### Step 1 — Get the firmware

Download `76931_65U85LEVS_Hisense_E006_O0410.zip` (~854 MB) from a Hisense firmware mirror.

The ZIP contains `MTK_9618_EU_G_pkg_20240410.tar.gz` which contains `usb_MTK_9618_EU_G.bin`.

#### Step 2 — Prepare the USB drive (macOS)

```bash
# Replace diskN with your actual disk number (check with: diskutil list)
diskutil eraseDisk FAT32 HISENSE MBRFormat /dev/diskN

# Prevent macOS metadata files
sudo mdutil -i off /Volumes/HISENSE
touch /Volumes/HISENSE/.metadata_never_index

# Extract firmware directly to USB
unzip -p ~/Downloads/76931_65U85LEVS_Hisense_E006_O0410.zip \
  | tar -xzOf - usb_MTK_9618_EU_G.bin \
  > /Volumes/HISENSE/usb_MTK_9618_EU_G.bin

# Verify (must be exactly 894841488 bytes)
stat -f %z /Volumes/HISENSE/usb_MTK_9618_EU_G.bin

# ⚠️ CRITICAL STEP — rename the file
mv /Volumes/HISENSE/usb_MTK_9618_EU_G.bin /Volumes/HISENSE/Forcedupgrade_001.bin

# Remove macOS hidden files, then eject
dot_clean -m /Volumes/HISENSE
sync && diskutil eject /dev/diskN
```

> **Why rename?** The bootloader looks for `Forcedupgrade_001.bin`. With the original name the TV completely ignores the USB drive and boots normally — no error, no message.

#### Step 3 — Flash the TV

1. Insert USB into the **left USB port** (USB 2.0 — white color, not the blue USB 3.0)
2. Power on the TV normally with the remote
3. The TV shows a **blue screen: "USB Upgrade — Checking."** with a progress bar

![USB Upgrade screen](img/usb_upgrade_screen.jpg)

4. **Do NOT power off. Do NOT remove the USB.** Wait 10–20 minutes.
5. TV reboots automatically → Hisense logo → first-time setup wizard
6. Remove USB after the logo appears
7. Go to Settings → Support → Factory Reset to clean up and verify

---

### What We Tried That Did NOT Work

This section is for people who already tried the obvious things.

#### ❌ USB with original filename (`usb_MTK_9618_EU_G.bin`)
Bootloader silently ignores it. TV boots to the broken screen as if no USB is inserted. No error. Renaming to `Forcedupgrade_001.bin` is mandatory.

#### ❌ 62 GB USB drive
Large FAT32 volumes are unreliable with MTK bootloaders. Use ≤32 GB.

#### ❌ Right USB port (blue USB 3.0)
Bootloader may not enumerate USB 3.0 drives. Use the white USB 2.0 port (left side on this model).

#### ❌ Physical button hold during power-on
On the 65U8NQ the physical button does not trigger USB recovery mode via hold. It simply boots the TV directly (bypasses Hisense logo, goes straight to whatever screen loads first).

#### ❌ UART via HDMI DDC pins (pins 15 and 16)
UART is disabled in production firmware on MTK9618. Tested all standard baud rates: 921600, 460800, 230400, 115200, 57600. Zero output on all.

#### ❌ UART via 3.5mm SERVICE port
Same result — no data at any baud rate. UART disabled in production.

#### ❌ MQTT commands (port 36669)
Port accepts connections but VIDAAcommon is dead when `packageinfo.json` is corrupt. All commands are ignored. Tested with both static credentials and dynamically generated tokens.

#### ❌ DNS spoof for OTA recovery
VIDAAcommon not running → HTTP client processes never start → TV makes zero TCP connections to external servers. DNS queries run but no TCP follows.

#### ❌ AirPlay recovery (port 7000)
Returns 403 on all endpoints. No recovery path.

#### ❌ DIAL (port 3366/3367)
Discovery XML responds but app launch returns 404 — VIDAAcommon is not registering apps.

---

### Hardware We Bought That Wasn't Needed (for reference)

We spent time and money on UART hardware before finding the USB solution:

| Item | Purpose | Result |
|------|---------|--------|
| FT232RL USB-UART adapter (3.3V) | UART console via HDMI or SERVICE port | UART disabled in firmware — no output |
| HDMI Male Plug with screw terminals | Connect to HDMI DDC pins without soldering | Tried pins 15/16 — no UART signal |
| Dupont F-F wires | Connections | N/A |

**Lesson:** Try USB firmware recovery with the correct filename *first*, before buying any UART hardware.

---

### Technical Background

#### eMMC partition layout (MTK9618, 42 partitions)
```
p20:  rootfs squashfs ~36 MB  (read-only system)
p37:  /usr/local       245 MB
p38:  /opt             665 MB  (MQTT broker: /opt/iot/mosquitto)
p39:  /var/local       348 MB  (writable, boot hooks)
p41:  /APPS            614 MB  (ext4 — target partition)
p42:  /OAD             921 MB  (OTA staging — DO NOT TOUCH)
```

#### The broken file
```
/APPS/pkgs/tv.vidaa.app.ui/packageinfo.json
```

When empty or corrupt → VIDAA app manager cannot register the main UI package → VIDAAcommon fails → black screen.

---

### If USB Recovery Still Doesn't Work

1. **Different USB drive** — try a different brand/size (8–16 GB), some drives don't enumerate correctly
2. **UART via mainboard test pads** — 3-pad group near MT9618 SoC, 115200 baud 8N1 3.3V. Use U-boot shell to directly edit the broken JSON file on eMMC
3. **eMMC ISP** — repair shops with Easy-JTAG / UFI box can read/write eMMC in-circuit without desoldering (~$20–60 USD)

---

### Contributing

If this worked for you on a different Hisense model, open a PR with:
- TV model and internal name
- Firmware filename and project number
- Which USB port worked
- Any differences from the steps above

---

## Українська

### Проблема

Телевізор показує **чорний екран** після ввімкнення. Пульт реагує тільки на кнопку живлення. Меню немає.

**Причина:** файл `/APPS/pkgs/tv.vidaa.app.ui/packageinfo.json` порожній або пошкоджений → VIDAAcommon не запускається → чорний екран.

Телевізор **не повністю зламаний** — завантажувач і ядро Linux працюють нормально. Відновлення можливе вдома з флешкою.

### Рішення — коротко

1. Флешка **≤32 ГБ**, FAT32, MBR
2. Скопіюй `usb_MTK_9618_EU_G.bin` з архіву прошивки (`76931_65U85LEVS_Hisense_E006_O0410.zip`)
3. **Перейменуй** файл у `Forcedupgrade_001.bin` — це обов'язково
4. Вставте у **лівий USB порт** (білий, не синій)
5. Увімкни телевізор пультом як звичайно
6. Має з'явитись синій екран **"USB Upgrade — Checking."** з прогрес-баром
7. Чекай 10–20 хвилин, не вимикай

**Без перейменування** завантажувач ігнорує флешку і телевізор завантажується у зламаний стан — без будь-яких повідомлень про помилку.

### Що точно не спрацює

- Оригінальна назва файлу `usb_MTK_9618_EU_G.bin` — ігнорується
- Флешка >32 ГБ — ненадійна з MTK завантажувачем
- Правий USB порт (синій USB 3.0) — може не визначатись
- Утримання фізичної кнопки на корпусі — не запускає recovery режим
- UART через HDMI піни 15/16 (DDC) — UART вимкнено у production прошивці
- MQTT команди — VIDAAcommon мертвий, всі команди ігноруються

---

## Русский

### Проблема

Телевизор показывает **чёрный экран** после включения. Пульт реагирует только на кнопку питания. Меню нет.

**Причина:** файл `/APPS/pkgs/tv.vidaa.app.ui/packageinfo.json` пустой или повреждён → VIDAAcommon не запускается → чёрный экран.

Телевизор **не полностью сломан** — загрузчик и ядро Linux работают нормально. Восстановление возможно дома с флешкой.

### Решение — кратко

1. Флешка **≤32 ГБ**, FAT32, MBR
2. Скопируй `usb_MTK_9618_EU_G.bin` из архива прошивки (`76931_65U85LEVS_Hisense_E006_O0410.zip`)
3. **Переименуй** файл в `Forcedupgrade_001.bin` — это обязательно
4. Вставь в **левый USB порт** (белый, не синий)
5. Включи телевизор пультом как обычно
6. Должен появиться синий экран **"USB Upgrade — Checking."** с прогресс-баром
7. Жди 10–20 минут, не выключай

**Без переименования** загрузчик игнорирует флешку и телевизор загружается в сломанное состояние — без каких-либо сообщений об ошибке.

### Что точно не работает

- Оригинальное имя файла `usb_MTK_9618_EU_G.bin` — игнорируется загрузчиком
- Флешка >32 ГБ — ненадёжна с MTK загрузчиком  
- Правый USB порт (синий USB 3.0) — может не определяться
- Удержание физической кнопки на корпусе при подаче питания — не запускает recovery
- UART через HDMI пины 15/16 (DDC) — UART отключён в production прошивке
- MQTT команды через порт 36669 — VIDAAcommon мёртв, все команды игнорируются
- DNS-спуфинг для OTA — TV не делает TCP-соединений когда VIDAAcommon не работает

### Купленное оборудование, которое не понадобилось

Мы потратили время и деньги на UART-оборудование, прежде чем нашли решение с USB:

- **FT232RL USB-UART адаптер** (3.3V) — UART отключён в прошивке, данных нет
- **HDMI Male Plug с клеммниками** (без пайки) — пробовали пины 15/16, сигнала нет
- **Провода Dupont** — не понадобились

**Вывод:** сначала попробуй USB recovery с правильным именем файла, прежде чем покупать любое железо.
