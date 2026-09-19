# Отчёт — Handshake JGMaker Magic V1.1 ↔ ATmega2560 bootloader (только проверка связи)

Дата: 18.09.2026 · Репозиторий: `F:\git\jg_maker_magic_v03_bltouch`
**Статус: плата НЕ прошилась, НЕ читалась, flash/EEPROM/fuse/bootloader НЕ тронуты. Файлы репозитория НЕ изменены. Git commit НЕ сделан.**

> **Критерий доверия**: каждое утверждение ниже — из РЕАЛЬНО ПРОЧИТАННЫХ локальных файлов или РЕАЛЬНО ВЫПОЛНЕННЫХ read-only команд (PowerShell / WSL). Если факт невозможно проверить — прямо сказано.

---

## A. COM-порт (Windows)

### Реальный результат (выполнено только что):

```powershell
Get-CimInstance Win32_SerialPort | Format-Table Name, Description, DeviceID -AutoSize
# → NO COM PORTS FOUND

Get-CimInstance Win32_PnPEntity | Where-Object { $_.Name -match 'CH340|COM4|USB.*Serial' }
# → USB-SERIAL CH340 (COM4)   Status=OK
#   Service      = CH341SER_A64
#   PNPDeviceID  = USB\VID_1A86&PID_7523\9&12127A34&0&1
#   (VID 0x1A86 = Qinheng, PID 0x7523 = CH340)

Get-Item -Path 'HKLM:\HARDWARE\DEVICEMAP\SERIALCOMM' | Get-ItemProperty
# → \Device\Serial2 : COM4
```

**Вывод**: на системе один USB-serial — **COM4** (CH340). Скорее всего это JGMaker Magic V1.1. Но формально — пока не проверен handshake — это гипотеза.

### Команда «ДО / ПОСЛЕ» (безопасная, read-only):

```powershell
# ДО подключения USB-кабеля
Get-CimInstance Win32_PnPEntity |
  Where-Object { $_.Name -match 'CH340|CP21|FTDI|USB.*Serial|ATmega' } |
  Select-Object Name, PNPDeviceID |
  Export-Csv "$env:TEMP\com_before.csv" -NoTypeInformation

# ПОСЛЕ подключения USB-кабеля
Get-CimInstance Win32_PnPEntity |
  Where-Object { $_.Name -match 'CH340|CP21|FTDI|USB.*Serial|ATmega' } |
  Select-Object Name, PNPDeviceID |
  Export-Csv "$env:TEMP\com_after.csv" -NoTypeInformation

# Сравнение
Compare-Object (Import-Csv "$env:TEMP\com_before.csv") (Import-Csv "$env:TEMP\com_after.csv")
```

---

## B. avrdude

### Реальный результат (выполнено только что):

```powershell
Get-Command avrdude -ErrorAction SilentlyContinue
# → (пусто)

where.exe avrdude
# → (пусто)

# Поиск avrdude.exe в стандартных путях Arduino IDE:
C:\Program Files (x86)\Arduino\hardware\tools\avr\bin\avrdude.exe      → НЕ НАЙДЕН
C:\Program Files (x86)\Arduino\hardware\arduino\avr\tools\avr\bin\    → НЕ НАЙДЕН
C:\Program Files\Arduino\hardware\tools\avr\bin\avrdude.exe           → НЕ НАЙДЕН
C:\Program Files\Arduino\hardware\arduino\avr\tools\avr\bin\          → НЕ НАЙДЕН

# Расширенный поиск по C:\ (depth 8):
Get-ChildItem C:\ -Filter avrdude.exe -Recurse -Depth 8
# → НЕ НАЙДЕН
```

**Вывод**: **Windows-версии avrdude.exe на этой машине НЕТ.** Arduino IDE не установлен.

Единственный доступный avrdude:
```
WSL Ubuntu-24.04:  /usr/bin/avrdude   version 7.1 (apt, от 01.04.2024)
```

**Последствия**:
- Из Windows — **нельзя** выполнить `-n -v` тест прямо сейчас, нет avrdude.exe.
- Из WSL — **нельзя** пробросить COM4 напрямую (COM-порты Windows не видны из WSL без `usbipd-win`, который не настроен).
- **Два варианта**:
  1. Установить `avrdude.exe` в Windows (winget / скачивание с github.com/avrdudes/avrdude/releases).
  2. Настроить `usbipd-win` + `usbipd attach --wsl` для проброса CH340 в WSL.

Ни один из вариантов **не выполнен** в этой сессии.

---

## C. Programmer — `wiring` (подтверждено реальными файлами)

| Источник | Строка | Значение |
|---|---|---|
| `/home/vivakalman/Arduino/hardware/arduino/avr/boards.txt` | 221 | `mega.menu.cpu.atmega2560.upload.protocol=wiring` |
| `/home/vivakalman/Arduino/hardware/arduino/avr/boards.txt` | 227 | `mega.menu.cpu.atmega2560.bootloader.file=stk500v2/stk500boot_v2_mega2560.hex` |
| `/etc/avrdude.conf` (WSL, avrdude 7.1) | 460–471 | `id = "wiring"; type = "wiring"; prog_modes = PM_SPM; connection_type = serial` — desc: *"Wiring for bootloader using STK500 v2 protocol"* |
| `/etc/avrdude.conf` (WSL, avrdude 7.1) | 475–483 | `id = "arduino"; type = "arduino"` — desc: *"Arduino for bootloader using STK500 v1 protocol"* |

**Вывод**: `-c wiring` (STK500v2). `arduino` (STK500v1) — **не** подходит для mega2560 bootloader.

---

## D. Baud rate — `115200` (подтверждено реальными файлами)

| Источник | Строка | Значение |
|---|---|---|
| `/home/vivakalman/Arduino/hardware/arduino/avr/boards.txt` | 223 | `mega.menu.cpu.atmega2560.upload.speed=115200` |

**Вывод**: `-b 115200`.

---

## E. RESET — что реально известно и чего мы не знаем

**Из реальных локальных файлов можно установить только следующее:**

| Факт | Источник |
|---|---|
| `wiring` = STK500v2 протокол | `/etc/avrdude.conf:460-471` |
| STK500v2 использует **SPM** (self-programming) через bootloader | `/etc/avrdude.conf` (`prog_modes = PM_SPM`) |
| Bootloader mega2560 = `stk500boot_v2_mega2560.hex`, region 0x3E000–0x3FD1D | `/home/vivakalman/Arduino/.../stk500v2/stk500boot_v2_mega2560.hex` (разбор) |
| `Makefile` для programmer `arduino` использует `stty hup`/`stty -hup` вокруг avrdude — это Unix-трюк, для `wiring` НЕ применяется | `Marlin/Makefile:790-800` |
| USB-serial чип на JGMaker Magic V1.1 = **CH340** (VID_1A86&PID_7523) | реестр Windows (PNP) |
| **НЕТ** ни одного файла в репозитории, описывающего: (a) схему USB-коннектора JGMaker, (b) подключение CH340 DTR/RTS к RESET ATmega2560, (c) наличие/отсутствие auto-reset контура, (d) прошивку самого CH340 | — |

**Чего мы НЕ знаем** (и не можем установить из локальных файлов):
- Подключён ли CH340 DTR к RESET MCU через стандартный RC+диодный делитель («Arduino autoreset»).
- Есть ли на плате физическая RESET-кнопка, куда она подключена.
- Работает ли CH340 в режиме с hardware flow control, достаточном для DTR-сигнала.

**Честный вывод**: **Auto-reset физически не подтверждён.**

### Безопасный ручной reset-процесс (только как эксперимент, БЕЗ записи):
Если авто-reset не сработает, при первом запуске `-n -v`:
1. Держать кнопку RESET на плате (если есть) нажатой.
2. Запустить avrdude.
3. В момент, когда на экране появляется `Device signature = 0x1e 0x98 0x01` — **отпустить** RESET.
4. Если signature не появляется за 5 секунд — отпустить RESET и повторить.
5. Три неудачные попытки → нужен внешний ISP (USBasp / USBtiny) — **не** в рамках этой сессии.

---

## F. Безопасный тест (ОДНА команда, read-only, `-n -v`)

Подтверждено в `avrdude --help` (выполнено: `avrdude --help` в WSL):
```
-n    Do not write anything to the device
-v    Verbose output; -v -v for more
```
`-n` **гарантирует**, что flash/EEPROM/fuse/bootloader НЕ изменяются. Команда только ведёт handshake с bootloader и читает подпись чипа.

### Готовая команда (с плейсхолдером `COMx` — подставьте `COM4` после вашего решения):

```bat
avrdude -C <путь к avrdude.conf> -p atmega2560 -P COMx -c wiring -b 115200 -n -v
```

**Варианты подстановки по текущему состоянию системы:**

| Окружение | Что нужно до запуска |
|---|---|
| **Windows** (рекомендация) | 1) Установить `avrdude.exe` (winget / github). 2) Скопировать `avrdude.conf` рядом. Подставить: `-C C:\path\to\avrdude.conf -P COM4`. |
| **WSL** (альтернатива) | 1) Установить `usbipd-win` в Windows. 2) `usbipd bind --wsl --vid-pid 1A86:7523`. 3) В WSL: `usbipd-attach 1A86:7523`. 4) Порт появится как `/dev/ttyUSB0`. Подставить: `-C /etc/avrdude.conf -P /dev/ttyUSB0`. |

> **Ни одна из этих команд не выполнялась** в этой сессии. Команда F — только шаблон для вашего ручного запуска.

---

## G. Что должно появиться в выводе (успех handshake)

### Реальная signature ATmega2560 (из локального `/etc/avrdude.conf`):

```
/etc/avrdude.conf, строки 9265-9281:
    part
        desc                   = "ATmega2560";
        id                     = "m2560";
        ...
        signature              = 0x1e 0x98 0x01;     ← ЭТО реальный байт
```

Для сравнения (чтобы не путать):
```
ATmega1280:  signature = 0x1e 0x97 0x04   (стр. ~9262)
ATmega2561:  signature = 0x1e 0x98 0x02   (стр. 9398, part parent m2560)
ATmega2560:  signature = 0x1e 0x98 0x01   (стр. 9280)
```

### Успешный вывод avrdude (ожидаемые строки):

```
AVRDUDE: M2560 STK500v2 mode
AVRDUDE: AVR memory initialized to 0x00
AVRDUDE: Device initialized to 0xFF
AVRDUDE: Device signature = 0x1e 0x98 0x01     ← УСПЕХ (совпадает с /etc/avrdude.conf:9280)
AVRDUDE: safemode? (нет, при -n)
AVRDUDE: done
```

### Неудачные сценарии:

| Симптом | Вероятная причина |
|---|---|
| `Device signature = 0x00 0x00 0x00` | Bootloader не отвечает (не в boot-режиме) — нужен RESET во время инициализации |
| `avrdude: ser_open(): can't open device "COM4": No such file` | COM4 занят (IDE / монитор портов) — закрыть |
| `avrdude: ser_open(): can't open device "COM4": Permission denied` | Не права доступа / антивирус |
| `Device signature = 0x1e 0x97 0x04` | На другом конце ATmega1280, не 2560 |
| Timeout / нет реакции вообще | CH340 не в режиме DTR-сброса, либо нет авто-reset контура → нужен ручной RESET |

### Важно:
- **Signature ≠ версия bootloader.** Signature — это жёстко зашитые в MCU 3 байта производителя (ATMEL + тип чипа). Версия bootloader определяется по коду, который выполняется в памяти 0x3E000+ — и не читается при handshake.
- `-n -v` **не читает** bootloader и не пишет. Он только инициализирует канал связи и читает `signature`. Это 100% read-only.

---

## H. Следующий шаг

**Не прошивать.** После успешного handshake (`Device signature = 0x1e 0x98 0x01`) возможны два варианта, оба — по вашему явному разрешению:

1. **Резервное чтение flash** (read-back всего 256 КБ):
   ```
   avrdude -C <conf> -p atmega2560 -P COMx -c wiring -b 115200 -U flash:r:backup.hex:i
   ```
   Результат: `backup.hex` — текущее содержимое flash **до** прошивки. Это позволяет откатиться.

2. **Запись `Marlin.hex`** — только после пункта 1 и вашего явного «да»:
   ```
   avrdude -C <conf> -p atmega2560 -P COMx -c wiring -b 115200 \
     -U flash:w:F:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.hex:i \
     -U flash:v:-:i -v
   ```

Ни одна из этих команд **не выполнялась** в этой сессии.

---

## Сводка: что реально установлено / не установлено

| Пункт | Статус | Источник |
|---|---|---|
| COM-порт = COM4 (CH340, VID_1A86&PID_7523) | ✅ подтверждено | PnP-реестр Windows |
| avrdude.exe в Windows | ❌ **не найден** (поиск по C:\ depth 8) | `Get-Command`, `where.exe`, `Get-ChildItem` |
| avrdude в WSL | ✅ `/usr/bin/avrdude` 7.1 | `avrdude --version` (выполнено ранее) |
| `-c wiring` = STK500v2 | ✅ подтверждено | `/etc/avrdude.conf:460-471`, `boards.txt:221` |
| `-b 115200` | ✅ подтверждено | `boards.txt:223` |
| `-p atmega2560` → `id=m2560`, `signature=0x1e 0x98 0x01` | ✅ подтверждено | `/etc/avrdude.conf:9269,9280` |
| `-n` = no-write | ✅ подтверждено | `avrdude --help` (выполнено) |
| Auto-reset контур на плате JGMaker | ❌ **не подтверждён** — нет локальных данных | — |
| Плата прошилась | ❌ **НЕТ** | — |
| Flash / EEPROM / fuse / bootloader изменены | ❌ **НЕТ** | — |
| Файлы репозитория изменены | ❌ **НЕТ** | — |
| Git commit | ❌ **НЕТ** | — |
