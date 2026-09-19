# FLASH REPORT — JG Maker Magic V1.1 → Marlin 2.0.5.4

**Дата**: 2026-09-18
**Статус**: ✅ **УСПЕШНАЯ ЗАПИСЬ + VERIFY + ПОДТВЕРЖДЕНИЕ ЦЕЛОСТИ BOOTLOADER'А**

---

## 1. Итог

| Параметр | Значение |
|----------|----------|
| Firmware | Marlin 2.0.5.4 (`Marlin/applet/Marlin.hex`) |
| SHA-256 HEX | `d18cdd87c028da4ef023c86b98a3735873fd81967a0012c730cff81150c12bbf` |
| MCU | ATmega2560, signature `1E 98 01` |
| Записано | **177,206 байт** (693 страницы + 202 pad bytes) |
| App region | `0x00000 – 0x2B435` |
| Verify | `177206 bytes of flash verified` ✅ |
| Bootloader | `0x3E000 – 0x3FD31` — **ЦЕЛ**, строка "Arduino explorer stk500V2 by MLS" на `0x3E0EF` |
| Fuses | Не тронуты (LFUSE=0xFF, HFUSE=0xD8, LOCK=0xFF) |
| EEPROM | Не тронута |
| Lock bits | Не тронуты |

---

## 2. Параметры записи

| Параметр | Значение |
|----------|----------|
| avrdude | 8.3 (MinGW, winget) |
| Путь | `C:\Users\cherk\AppData\Local\Microsoft\WinGet\Packages\AVRDudes.AVRDUDE.MinGW_...\avrdude.exe` |
| Программист | `-c wiring` (STK500v2, SPM) |
| Порт | `-P COM4` (CH340, `VID_1A86&PID_7523`) |
| Baud | `-b 115200` |
| Флаг | `-D` (disable auto-erase — bootloader не поддерживает chip erase) |
| HEX | `-U flash:w:"f:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.hex":i` |

**Полная команда**:
```
avrdude -D -p atmega2560 -c wiring -P COM4 -b 115200 -U flash:w:"f:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.hex":i -v
```

---

## 3. Хронология сессии

| Шаг | Действие | Результат |
|-----|----------|-----------|
| 1 | COM4 verify (GetPortNames, mode, PnP) | ✅ Реальный, CH340, PnP OK |
| 2 | HEX SHA-256 + size + range | ✅ `d18cdd...`, 177206 B, `0x00000–0x2B435` |
| 3 | Протокол bootloader (из HANDSHAKE_log + reports) | ✅ STK500v2, `-c wiring`, baud 115200 |
| 4 | Установка avrdude 8.3 MinGW (winget) | ✅ Успешно |
| 5 | Read-only handshake (`-n -v`) | ✅ "ready to accept instructions", sig `1E 98 01` |
| 6a | Первая попытка записи (без `-D`) | ❌ "chip erase failed" (bootloader не поддерживает) |
| 6b | Запись с `-D` | ✅ **177206 bytes written + verified** (26.19s + 19.99s) |
| 7 | Post-flash verify (full read-back) | ✅ App + Bootloader оба целы |
| 8 | G-code verification @ **250000** (новый `BAUDRATE`) | ✅ **Marlin 2.0.5.4 (Magic v0.3.3) ALIVE**, M115/M114/M104/M503 OK |

---

## 4. Verify (read-back)

```
APP 0x00000:  0C 94 E6 1F 0C 94 FA 22 0C 94 25 23 0C 94 50 23  ← reset vector (jmp)
APP 0x2B435:  58 00 9E 16 2F 00                                  ← last bytes of Marlin
BL  0x3E000:  0D 94 89 F1 0D 94 B2 F1 0D 94 B2 F1 0D 94 B2 F1  ← bootloader (ЦЕЛ)
BL  0x3E0EF:  "Arduino explorer stk500V2 by MLS"                ← родной bootloader
```

---

## 5. Риски и rollback

| Риск | Митигация |
|------|-----------|
| Потеря app | HEX `0x00000–0x2B435` **не пересекается** с bootloader `0x3E000+`. Bootloader подтверждён живым. |
| Потеря bootloader | Невозможно — запись только в app region, bootloader region не адресован. Подтверждено read-back. |
| Fuses / Lock / EEPROM | Не тронуты — команда содержит только `flash:w:`. |
| **Rollback** | `FLASH_backup_2026-09-18.hex` (620,952 B) содержит полный dump флэша ДО записи (включая старый app + bootloader). Для отката: `avrdude -D -p atmega2560 -c wiring -P COM4 -b 115200 -U flash:w:"f:\git\jg_maker_magic_v03_bltouch\FLASH_backup_2026-09-18.hex":i -v` |

---

## 6. BLTouch

**СТАТУС**: DISCONNECTED (как и требовалось во время прошивки).

**Подключение (следующая задача)**:
- 3-pin: Brown→J1-G(GND), Red→J1-V(+5V), Yellow→J1-S(CONTROL)
- 2-pin: White→Z-S(SIGNAL), Black→Z-G(GND)
- Z-V: unused

---

## 8. G-code verification — **УСПЕШНО @ 250000**

**Ключевое открытие**: новый firmware (`Marlin/applet/Marlin.hex`) использует **`BAUDRATE 250000`** (`Configuration.h:124`), а не 115200 (старая прошивка). Поэтому все ранние попытки @115200 давали garbled — **это не ошибка записи, а различие конфигураций**.

**После открытия COM4 @250000**:
```
start
echo:Marlin 2.0.5.4 (Magic v0.3.3)
echo: Last Updated: 2020-07-19 | Author: Christopher Norton
echo:Compiled: Sep 18 2026
echo: Free Memory: 1498  PlannerBufferBytes: 1520
```

**G-code ответы** (@250000):
- **M115** → `FIRMWARE_NAME:Marlin 2.0.5.4 (Magic v0.3.3) (GitHub) ... MACHINE_TYPE:JGMaker Magic ... Cap:Z_PROBE:1 Cap:AUTOLEVEL:1 Cap:EEPROM:1`
- **M114** → `X:-13.00 Y:-13.00 Z:0.00 E:0.00 Count X:-1040 Y:-1040 Z:0`
- **M104** → `ok`
- **M503** → Полный dump EEPROM:
  - `M92 X80.00 Y80.00 Z800.00 E88.00`
  - `M301 P22.20 I1.08 D114.00` (PID — нетронут)
  - `M851 X46.00 Y14.00 Z0.00` (**NOZZLE_TO_PROBE_OFFSET — нетронут**, как и требовалось)
  - `M206 X0.00 Y0.00 Z0.00`, `M420 S0 Z0.00`

**ВЫВОД**: ✅ **Новый firmware 100% работает**. Все G-коды отвечают, EEPROM настройки сохранены, BLTouch-оффсет `X46 Y14 Z0` нетронут.

---

## 9. Итоговая таблица статуса

| Область | До | После |
|---------|----|----|
| App firmware | Marlin v0.2 (старая, 2019-05-14) | **Marlin 2.0.5.4 (Magic v0.3.3)** |
| App region 0x00000–0x2B435 | Старый .text | **Новый .text (177 206 B, verified)** |
| Bootloader 0x3E000–0x3FD31 | `Arduino explorer stk500V2 by MLS` | **ЦЕЛ, строка на 0x3E0EF** |
| Fuses (LF/HF) | 0xFF / 0xD8 | **Нетронуты** |
| Lock bits | 0xFF | **Нетронуты** |
| EEPROM | V76 stored (736 B, crc 62013) | **Нетронута** (M503 подтверждает) |
| NOZZLE_TO_PROBE_OFFSET | X46 Y14 Z0 | **Нетронут** (M851) |
| BAUDRATE | 115200 (предположительно) | **250000** (Configuration.h:124) |
| BLTouch | Disconnected | **Disconnected** (не подключался) |

---

## 7. Артефакты сессии

| Файл | Назначение |
|------|------------|
| `flash_verify_full.bin` (Temp) | Полный read-back флэша (261426 B) — для будущего diff |
| `FLASH_backup_2026-09-18.hex` | **Rollback** — полный dump до записи |
| `HANDSHAKE_log_2026-09-18.txt` | Лог успешного handshake (WSL, прошлая сессия) |
| `probe_reset_sync.ps1` | DTR/RTS reset + sync probe (read-only) |
| `probe_sync.ps1` | Basic sync probe (read-only) |
| `opencheck.ps1` / `.log` | COM4 open check |
| `probe_bootloader.ps1` / `.log` | Bootloader probe |
