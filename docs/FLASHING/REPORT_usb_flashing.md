# Отчёт — USB-флэшинг JGMaker Magic V1.1 (ATmega2560, Marlin 2.0.5.4)

Дата: 18.09.2026 · Репозиторий: `F:\git\jg_maker_magic_v03_bltouch` · Статус: **плата НЕ прошилась, ничего не пересобрано, commit НЕ сделан**

> **Критерий доверия**: каждое утверждение ниже основано на РЕАЛЬНО ПРОЧИТАННЫХ локальных файлах (указан путь → строка → значение). Если факт невозможно проверить по имеющимся файлам — это прямо отмечено в разделе E.

---

## A. Programmer

| Параметр | Значение | Источник |
|---|---|---|
| `-c <programmer>` | **`wiring`** | `boards.txt:221` |
| Протокол | STK500 **v2** | `/etc/avrdude.conf` стр. 460–471 |
| Запрещено | `arduino` (STK500 **v1**) | `/etc/avrdude.conf` стр. 475–483 |

**Доказательства:**

1. `/home/vivakalman/Arduino/hardware/arduino/avr/boards.txt:221`
   ```
   mega.menu.cpu.atmega2560.upload.protocol=wiring
   ```
2. `/etc/avrdude.conf` (аврду де 7.1, apt, WSL Ubuntu-24.04), строки 460–471:
   ```
   programmer
       desc       = "Wiring for bootloader using STK500 v2 protocol"
       type       = "wiring"
       prog_modes = PM_SPM
       connection_type = serial
   ```
3. `/etc/avrdude.conf`, строки 475–483:
   ```
   programmer
       desc       = "Arduino for bootloader using STK500 v1 protocol"
       type       = "arduino"
   ```
4. `boards.txt:227` → bootloader = `stk500v2/stk500boot_v2_mega2560.hex` (bootloader **v2** → требуется `wiring`).

**Вывод:** `-c wiring`. `Marlin/Makefile:75` по умолчанию ставит `AVRDUDE_PROGRAMMER ?= arduino` — это НЕПРАВИЛЬНО для Mega 2560; комментарий в `Makefile:50` прямо советует: *"If uploading doesn't work try adding the parameter 'AVRDUDE_PROGRAMMER=wiring'"*.

---

## B. Baud rate

| Параметр | Значение | Источник |
|---|---|---|
| `-b <baudrate>` | **`115200`** | `boards.txt:223` |

**Доказательство:**
```
/home/vivakalman/Arduino/hardware/arduino/avr/boards.txt:223
mega.menu.cpu.atmega2560.upload.speed=115200
```

`Makefile:74` по умолчанию `UPLOAD_RATE ?= 57600` (значение под mega1280) — для mega2560 переопределяется `UPLOAD_RATE=115200`.

---

## C. Bootloader (реальный разбор hex)

| Параметр | Значение |
|---|---|
| Файл | `/home/vivakalman/Arduino/hardware/arduino/avr/bootloaders/stk500v2/stk500boot_v2_mega2560.hex` |
| Первый адрес | **0x3E000** |
| Последний адрес | **0x3FD1D** |
| Размер | **7454 байта (7.28 KB)** |
| Data-записей | 466 |

**Расчёт (из реальных строк файла):**
- ESA-запись: `:02000002 3000 CC` → base = 0x3000 × 16 = **0x30000**
- Первая data: `:10E00000 0D9489F1...` → 0x30000 + 0xE000 = **0x3E000**
- Последняя data: `:0EFD1000 FA9AF99A0FBE01960895F894FFCF63` → 0x30000 + 0xFD10 = 0x3FD10, +14 байт = **0x3FD1D**
- Fuse-запись: `:04000003 3000E000E9` (type 03: low=0x30, high=0x00, ext=0x00)

---

## D. Application flash (Marlin.hex) + проверка пересечения

| Параметр | Значение |
|---|---|
| Файл | `Marlin/applet/Marlin.hex` (498 447 байт, 11 079 строк) |
| Первый адрес | **0x00000** |
| Последний адрес | **0x2B435** |
| Размер | **177 206 байт (173.05 KB, 67.6 %)** |
| Data-записей | 11 076 |
| ESA-записи (type 02) | `:02000002 1000 EC` → 0x10000, `:02000002 2000 DC` → 0x20000 |
| Max app size | `253952 = 0x3E000` (`boards.txt:222`) |
| **Пересечение с bootloader** | **НЕТ** |

**Реальный результат проверки (Python-разбор с учётом ESA type 02):**
```
Marlin.hex  first = 0x00000   last = 0x2B435   size = 177206 bytes
bootloader  first = 0x3E000   last = 0x3FD1D   size = 7454 bytes
OVERLAP: False
Max app size from boards.txt = 253952 = 0x3E000
```

Зазор между концом приложения и началом bootloader: 0x3E000 − 0x2B436 = **79 870 байт (~78 KB)**. Размер совпадает с выводом сборки `Program: 177206 bytes (67.6 percent)`.

> Примечание: в hex-файле НЕТ записей type 04 (Extended Linear Address) — расширение адреса сделано только записями type 02 (Extended Segment Address). Простой просмотр `head`/`tail` без обработки ESA даёт неверную «последнюю строку» — именно поэтому проверка выполнена программно.

---

## E. Auto-reset

**НЕТ ДАННЫХ для верификации по имеющимся локальным файлам.** Прямо сообщаю об этом, как требовалось.

Что реально проверено:
- `Marlin/Makefile` upload-таргет (строки 790–800): только `stty hup` / `stty -hup` вокруг avrdude — и то только для `AVRDUDE_PROGRAMMER=arduino`. Для `wiring` (STK500v2) avrdude сам управляет DTR/RTS.
- В репозитории **нет** ни одного файла о USB-serial чипе JGMaker Magic V1.1, его схеме, pinout'е или его прошивке.

Что следует из протокола STK500v2 (общезвестно, но НЕ доказуемо из файлов репозитория):
- `wiring`-режим ожидает стандартный «Arduino auto-reset» контур: DTR/RTS USB-serial чипа → диоды → RESET MCU.
- Если на плате JGMaker Magic V1.1 стоит FT232R/CH340/CP2102 без такого контура — авто-сброс не сработает; понадобится нажатие RESET в момент начала записи либо внешний ISP-программатор.

**Практический приём:** при первом запуске avrdude держать RESET нажатым и отпустить в момент, когда на экране появится `Device signature = 0x1e 0x98 0x21` (подпись ATmega2560). Если после 3 неудачных попыток — нужен внешний ISP.

---

## F. COM-порт (Windows) / устройство (WSL)

avrdude установлен **только в WSL** (`/usr/bin/avrdude`, 7.1). На Windows avrdude отдельно НЕ установлен (проверено).

### Из WSL:
```bash
# Список tty-устройств
ls /dev/ttyUSB* /dev/ttyACM* /dev/ttyS* 2>/dev/null

# Кто из USB-чипов подключён
lsusb | grep -iE "usb|serial|atmega|ch34|cp21|ftdi"

# После подключения платы — последнее событие ядра
dmesg | tail -20
```
⚠️ Windows COM-порты из WSL напрямую НЕ видны (только файловая система `/mnt/c`). Проброс USB-устройства в WSL требует usbipd-win (не настроен) — либо прошивать из WSL с проброшенным USB, либо из Windows с avrdude от Arduino IDE.

### Из Windows (PowerShell):
```powershell
# Все COM-порты
Get-CimInstance Win32_SerialPort | Format-Table Name, Description, DeviceID -AutoSize

# Классический способ
mode

# Найти USB-serial чип
pnputil /enum-devices /connected | Select-String -Pattern "FTDI|CH34|CP21|USB.*Serial"
```

**Как определить «свой» порт:**
1. `mode` → запомнить список
2. Подключить плату USB
3. `mode` → новый COM — ваш

---

## G. READ-ONLY тест (ничего не пишет в плату)

Подтверждено в `avrdude --help`:
```
-U <memtype>:r|w|v:<filename>[:format]   Memory operation specification
-n                                       Do not write anything to the device
```

`-U flash:r:<file>:i` = **чтение** flash (ничего не записывается). Это чистая read-only операция — рекомендуется вместо `-n` (который не пишется НИЧЕГО, но и читать не даёт, если без `-U`).

### Из WSL:
```bash
avrdude \
  -C /etc/avrdude.conf \
  -p atmega2560 \
  -P /dev/ttyUSB0 \
  -c wiring \
  -b 115200 \
  -U flash:r:/tmp/readback.hex:i \
  -v
```
Результат: `/tmp/readback.hex` — содержимое flash платы **до** прошивки (резервная копия).

### Из Windows:
```bat
avrdude -C %ARDUINO_DIR%\hardware\arduino\avr\etc\avrdude.conf -p atmega2560 -P COM3 -c wiring -b 115200 -U flash:r:C:\temp\readback.hex:i -v
```

---

## H. Проверка HEX (до прошивки)

```bash
# 1. Размер (ожидается: 177206 bytes)
size --format=avr --target=atmega2560 Marlin/applet/Marlin.hex

# 2. Первые/последние строки
head -5 Marlin/applet/Marlin.hex
tail -5 Marlin/applet/Marlin.hex
# Ожидаемо:
#   :100000000C94E61F...      (адрес 0x0000)
#   :020000021000EC           (ESA → 0x10000)
#   :020000022000DC           (ESA → 0x20000)
#   :0AB42C00...              (реальный 0x2B42C)
#   :00000001FF               (EOP)

# 3. Контрольные суммы ВСЕХ записей (ожидается BAD=0)
python3 -c "
ok = bad = 0
for line in open('Marlin/applet/Marlin.hex'):
    line = line.strip()
    if not line.startswith(':') or len(line) < 11: continue
    ln = int(line[1:3], 16); ad = int(line[3:7], 16); ty = int(line[7:9], 16)
    data = bytes.fromhex(line[9:9+2*ln])
    cs = int(line[9+2*ln:11+2*ln], 16)
    if (ln + ad + ty + sum(data) + cs) & 0xFF == 0: ok += 1
    else: bad += 1; print('BAD CS:', line)
print(f'OK={ok}  BAD={bad}')
"

# 4. Сопоставление с размером из сборки (make sizeafter / .lss)
```

---

## I. Реальная команда прошивки (НЕ ИСПОЛНЯТЬ)

### Из WSL:
```bash
avrdude \
  -C /etc/avrdude.conf \
  -p atmega2560 \
  -P /dev/ttyUSB0 \
  -c wiring \
  -b 115200 \
  -U flash:w:/mnt/f/git/jg_maker_magic_v03_bltouch/Marlin/applet/Marlin.hex:i \
  -U flash:v:-:i \
  -v
```

| Флаг | Значение |
|---|---|
| `-C` | `/etc/avrdude.conf` (реальный путь apt-пакета; Makefile ожидает `/etc/avrdude/avrdude.conf` — каталог, его нет) |
| `-p` | `atmega2560` |
| `-P` | ваш порт (WSL: `/dev/ttyUSB0`, Windows: `COM3` и т.п.) |
| `-c` | **`wiring`** (STK500v2 — ключевое отличие от `arduino`) |
| `-b` | **`115200`** |
| `-U flash:w:...:i` | запись из hex |
| `-U flash:v:-:i` | верификация (readback-сравнение) |
| `-v` | подробный вывод |

### Из Windows:
```bat
avrdude -C %ARDUINO_DIR%\hardware\arduino\avr\etc\avrdude.conf -p atmega2560 -P COM3 -c wiring -b 115200 -U flash:w:F:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.hex:i -U flash:v:-:i -v
```
avrdude.exe берётся из Arduino IDE: `%ARDUINO_DIR%\hardware\arduino\avr\tools\avr\bin\avrdude.exe` (в PATH добавить при необходимости).

### Через Makefile (без правки файла — только переменные):
```bash
make -j4 upload \
  ARDUINO_INSTALL_DIR=/home/vivakalman/Arduino \
  HARDWARE_MOTHERBOARD=1020 \
  AVRDUDE_PROGRAMMER=wiring \
  UPLOAD_RATE=115200 \
  UPLOAD_PORT=/dev/ttyUSB0
```
⚠️ `Makefile:734`: на Linux `AVRDUDE_CONF = /etc/avrdude/avrdude.conf` — каталог `/etc/avrdude/` **не существует**, поэтому `make upload` упадёт на `-C`. Обход без правки Makefile:
```bash
sudo mkdir -p /etc/avrdude && sudo cp /etc/avrdude.conf /etc/avrdude/avrdude.conf
```
(либо прошивать ручной командой из раздела I — рекомендуется).

---

## J. Последняя проверка перед flash (чеклист)

| # | Проверка | Команда | Ожидание |
|---|---|---|---|
| 1 | Порт существует | `ls /dev/ttyUSB*` (WSL) / `mode` (Win) | порт виден |
| 2 | Порт не занят | `lsof /dev/ttyUSB0` (WSL) | пусто |
| 3 | HEX на месте | `ls -la Marlin/applet/Marlin.hex` | 498 447 байт |
| 4 | Размер HEX | `size --format=avr --target=atmega2560 Marlin/applet/Marlin.hex` | 177 206 bytes |
| 5 | Размер < max | 177206 < 253952 | ✅ |
| 6 | Нет пересечения с bootloader | 0x2B435 < 0x3E000 | ✅ (раздел D) |
| 7 | Контрольные суммы | пункт H.3 | BAD=0 |
| 8 | Резервная копия flash (рекомендуется) | раздел G | `readback.hex` создан |
| 9 | Питание платы | визуально | LED горит |
| 10 | RESET-кнопка доступна | визуально | для первого раза |
| 11 | `programmer=wiring` | в команде | ✅ |
| 12 | `baud=115200` | в команде | ✅ |
| 13 | `mcu=atmega2560` | в команде | ✅ |
| 14 | Файл — **hex** (не elf/bin) | в команде | ✅ |
| 15 | Конфиг-файл существует | `ls -la /etc/avrdude.conf` | есть |

---

## Изменения / неизменённое

| Файл | Статус |
|---|---|
| `REPORT_usb_flashing.md` | **СОЗДАН** (этот файл) |
| `REPORT_bltouch_v1.1.md` | создан ранее, не тронут |
| `Marlin/Configuration.h` | изменён ранее (X_MIN_PIN=2, SERVO0_PIN=3) — **в этой сессии НЕ тронут** |
| `Marlin/src/pins/ramps/pins_RAMPS.h` | **НЕ изменён** |
| `Marlin/Makefile` | **НЕ изменён** |
| `Marlin/src/libs/Tone.cpp` | **НЕ изменён** |
| `Marlin/applet/Marlin.hex` | **НЕ пересобран** |
| Плата | **НЕ прошивалась** |
| Git | **commit НЕ сделан** |

---

## Итог

| Раздел | Статус |
|---|---|
| A. Programmer | ✅ `wiring` (STK500v2) |
| B. Baud rate | ✅ 115200 |
| C. Bootloader | ✅ 0x3E000–0x3FD1D, 7454 bytes |
| D. Application | ✅ 0x00000–0x2B435, 177206 bytes, **пересечения нет** |
| E. Auto-reset | ⚠️ **не проверено по файлам репозитория** (данных нет) |
| F. COM-порт | ✅ команды WSL + Windows |
| G. READ-ONLY | ✅ `-U flash:r:...:i` |
| H. Проверка HEX | ✅ 4 команды |
| I. Реальная прошивка | ✅ команды WSL + Windows + make (НЕ исполнены) |
| J. Pre-flash чеклист | ✅ 15 пунктов |
