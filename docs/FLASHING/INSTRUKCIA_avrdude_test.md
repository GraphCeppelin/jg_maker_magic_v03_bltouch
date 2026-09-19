# Пошаговая инструкция: установка avrdude + read-only handshake тест

Дата: 18.09.2026
**Важно**: на каждом шаге — ТОЛЬКО read-only / установка ПО. **НИКАКОЙ записи в плату** до вашего явного «GO». Ни `-U flash:w:`, ни `make upload`, ни fuse, ни EEPROM.

Цель: получить рабочий `avrdude` (Windows или WSL) и прогнать ОДНУ безопасную команду `-n -v`, чтобы проверить, что bootloader ATmega2560 на COM4 отвечает.

---

## Шаг 0. Текущее состояние (подтверждено)

| Что | Статус |
|---|---|
| COM4 = USB-SERIAL CH340 (`VID_1A86&PID_7523`) | ✅ есть |
| `avrdude.exe` в Windows | ❌ нет |
| `avrdude` в WSL (Ubuntu-24.04) | ✅ `/usr/bin/avrdude` 7.1 |
| Arduino IDE в Windows | ❌ не найден |
| `usbipd-win` | ❓ не проверено (установим в Варианте Б) |

---

## ВАРИАНТ А — Windows (рекомендуется, проще)

### А1. Скачать официальный avrdude (Windows x64)
Официальный релиз: **avrdudes/avrdude**. Для Windows нужен **статический zip** (содержит `avrdude.exe` + `avrdude.conf` без зависимостей).

Актуальный релиз (по данным https://github.com/avrdudes/avrdude/releases): **v8.3**.

| Файл | Назначение |
|---|---|
| `avrdude-v8.3-windows-x64.zip` | **MSVC build, рекомендуется** (~7.3 МБ) |
| `avrdude-v8.3-windows-x86.zip` | 32-бит (7.23 МБ) |
| `avrdude-v8.3-windows_mingw-x64.zip` | MSYS2 mingw (4.84 МБ) |
| `avrdude-v8.3-windows-arm64.zip` | ARM64 (6.99 МБ) |

⚠️ **Известные ограничения MSVC build** (из release notes):
- `#1440` — **no support of CH341A**. У нас **CH340** (`PID 7523`), а не CH341A — это **разные** чипы. Для `-c wiring` (serial/STK500v2) драйвер не нужен — avrdude работает по стандартному COM-порту. **Но** если MSVC build не увидит CH340-порт, переключитесь на mingw build.

Скачать (одна из команд):
```powershell
# Через winget (если доступен)
winget install avrdudes.avrdude

# ИЛИ вручную через PowerShell (скачивает zip в $env:TEMP)
Invoke-WebRequest -Uri "https://github.com/avrdudes/avrdude/releases/download/v8.3/avrdude-v8.3-windows-x64.zip" -OutFile "$env:TEMP\avrdude-v8.3-windows-x64.zip"
```

### А2. Распаковать в удобную папку (например `C:\tools\avrdude`)
```powershell
$dst = "C:\tools\avrdude"
New-Item -ItemType Directory -Force -Path $dst | Out-Null
Expand-Archive -Path "$env:TEMP\avrdude-v8.3-windows-x64.zip" -DestinationPath $dst -Force
# В $dst должен оказаться: avrdude.exe  и  avrdude.conf
Get-ChildItem $dst
```

### А3. Проверить, что avrdude запускается (read-only)
```powershell
$env:Path += ";C:\tools\avrdude"
avrdude --version
# Ожидается: avrdude version 8.x, ...

# Проверить, что programmer "wiring" поддерживается (read-only)
avrdude -c wiring -p atmega2560 --part-info 2>&1 | Select-Object -First 5
# ИЛИ просто список программаторов:
avrdude -c ?type 2>&1 | Select-String -Pattern "wiring|stk500"
```

### А4. Убедиться, что COM4 свободен (read-only)
```powershell
# Если кто-то держит порт — закрыть программу (Arduino IDE, PuTTY, монитор)
Get-CimInstance Win32_SerialPort | Where-Object Name -eq "COM4"
```

### А5. **БЕЗОПАСНЫЙ ТЕСТ** (read-only, `-n -v` — ничего не пишет)
```powershell
avrdude -C C:\tools\avrdude\avrdude.conf `
        -p atmega2560 `
        -P COM4 `
        -c wiring `
        -b 115200 `
        -n -v
```
**Что искать в выводе**: строка `Device signature = 0x1e 0x98 0x01`.
- Появилась → bootloader отвечает ✅
- `0x00 0x00 0x00` / timeout → нужен ручной RESET (раздел E) или CH340-проблема.

---

## ВАРИАНТ Б — WSL (если Windows-вариант не сработал)

Нужен **проброс CH340 из Windows в WSL** через `usbipd-win`.

### Б1. Установить usbipd-win (Windows, админ-PowerShell)
```powershell
winget install usbipd.usbipd-win
# ИЛИ:  wsl --install  (если WSL ещё не готов)
usbipd --version
```

### Б2. Начать сервис и найти устройство (Windows)
```powershell
usbipd start
usbipd list
# Ищем строку: ... USB-SERIAL CH340 (COM4) ... state: Available
# Запомним BusID (первое поле, например 1-3)
```

### Б3. Привязать и отдать устройство в WSL (Windows)
```powershell
# Привязать по VID:PID (наш CH340 = 1A86:7523)
usbipd bind --wsl --vid-pid 1A86:7523
# ИЛИ по BusID (из шага Б2, подставьте свой):
# usbipd bind --wsl --busid 1-3

# Проверить
usbipd list
```

### Б4. Подключить в WSL (внутри Ubuntu-24.04)
```bash
# В WSL:
sudo modprobe usbip_host
sudo usbip list --remote  # (если нужно) — иначе просто:
sudo usbip attach --wsl 1A86:7523
# Увидеть новое устройство:
lsusb | grep -i 1a86
# Порт должен появиться как /dev/ttyUSB0 (или /dev/ttyACM0)
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

### Б5. **БЕЗОПАСНЫЙ ТЕСТ** из WSL
```bash
wsl -d Ubuntu-24.04 -- bash -c "
  avrdude -C /etc/avrdude.conf \
          -p atmega2560 \
          -P /dev/ttyUSB0 \
          -c wiring \
          -b 115200 \
          -n -v
"
```
**Ищем**: `Device signature = 0x1e 0x98 0x01`.

### Б6. Откат (отключить из WSL, вернуть в Windows)
```bash
# В WSL:
sudo usbip detach 1A86:7523
```
```powershell
# В Windows (если нужно):
usbipd unbind --wsl
usbipd list
```

---

## Раздел E. RESET (безопасный ручной, только эксперимент)

Если `-n -v` даёт timeout / `0x00 0x00 0x00`:

1. **Авто-reset не подтверждён** (нет локальной схемы JGMaker). CH340 сам по себе **не генерирует** DTR-сброс MCU без внешнего RC+диодного контура.
2. Безопасный приём (без записи):
   - Поставьте палец на RESET-кнопку платы (если есть).
   - Запустите `-n -v`.
   - Когда avrdude начнёт инициализацию (первые строки), **нажмите и удерживайте** RESET.
   - Отпустите через ~1–2 секунды.
   - Если `Device signature = 0x1e 0x98 0x01` появился — ✅.
3. Если 3 попытки безрезультатны → CH340 на этой плате, вероятно, **не соединён** с RESET (или нет кнопки). Тогда нужен внешний ISP-программатор (USBasp/USBtiny) — **не** в рамках этой сессии.

---

## Раздел G. Что означает успех

| Строка вывода | Значение |
|---|---|
| `Device signature = 0x1e 0x98 0x01` | ✅ **УСПЕХ** — ATmega2560 ответил (подпись из `/etc/avrdude.conf:9280`) |
| `Device signature = 0x1e 0x97 0x04` | ❌ Это ATmega1280, не 2560 |
| `Device signature = 0x1e 0x98 0x02` | ❌ Это ATmega2561 |
| `Device signature = 0x00 0x00 0x00` | ❌ Bootloader не в boot-режиме (нужен RESET) |
| `can't open device "COM4"` | ❌ Порт занят / неверный номер |
| timeout | ❌ CH340 не отвечает / нет DTR-сброса |

**Signature ≠ версия bootloader.** Это жёстко зашитые 3 байта ID чипа.

---

## Итог

- **Вариант А (Windows)** — 6 команд, проще. Рекомендую начать с него.
- **Вариант Б (WSL)** — если А не сработал, 4–5 команд + проброс USB.
- Оба варианта заканчиваются **ОДНОЙ** read-only командой `-n -v`.
- **Плата НЕ прошивается** на этом этапе. Запись `Marlin.hex` — только отдельным вашим «GO» после успешного handshake и резервного чтения flash.
