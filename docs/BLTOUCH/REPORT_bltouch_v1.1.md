# Полный отчёт — BLTouch / JGMaker Magic V1.1 (Marlin 2.0.5.4)

Дата: 18.09.2026 · Репозиторий: `F:\git\jg_maker_magic_v03_bltouch` · Статус: **сборка успешна, commit/прошивка НЕ выполнялись**

---

## A. Что подтверждено реальными файлами

### Цепочка включения (реальная)

| Шаг | Файл:строка | Что делает |
|---|---|---|
| 1 | `src/inc/MarlinConfigPre.h:35` | `#include "../../Configuration.h"` — **наш Configuration.h включается ПЕРВЫМ** |
| 2 | `src/inc/MarlinConfig.h` | → `pins/pins.h` → `Conditionals_post.h` → `SanityCheck.h` |
| 3 | `src/pins/pins.h:71` | Для `MOTHERBOARD=1020` → `#include "ramps/pins_RAMPS.h"` |
| 4 | `Marlin/Configuration.h` | **явный `#define SERVO0_PIN 19`** (D19, разъём Z+, жёлтый servo-провод) — override вступает в силу ДО `pins_RAMPS.h` (USER-CONFIRMED 2026-09-21; «virtual/15» **ОТЗЫВАЕТСЯ**) |
| 5 | `src/pins/ramps/pins_RAMPS.h:96-98` | `#ifndef X_MIN_PIN` → `3` — **действует** (override `X_MIN_PIN 2` удалён — X− = D3) |
| 6 | `src/pins/ramps/pins_RAMPS.h:106` | `#ifndef Z_MIN_PIN` → `18` — **сработает** (мы не override'им) |
| 7 | `src/inc/Conditionals_LCD.h:534-540` | BLTOUCH: **принудительно** `Z_MIN_ENDSTOP_INVERTING false` + `Z_MIN_PROBE_ENDSTOP_INVERTING false` |

### Ключевые определения (файл:строка)

| Определение | Значение | Файл:строка |
|---|---|---|
| `MOTHERBOARD` | `BOARD_RAMPS_14_EFB` = **1020** | `Configuration.h:131` + `boards.h:40` |
| `BLTOUCH` | ✅ | `Configuration.h:888` |
| `Z_PROBE_SERVO_NR` | `0` | `Configuration.h:882` |
| `Z_SERVO_ANGLES` | `{10, 90}` | `Configuration.h:891` |
| `SERVO0_PIN` | **19** (D19/PD4/pin 47, разъём Z+) | `Configuration.h` override — **REAL WIRE (жёлтый)** (BLTouch servo) |
| `X_MIN_PIN` | **3** (D3/PE5) | `pins_RAMPS.h` default (override `X_MIN_PIN 2` **removed**) |
| `Z_MIN_PIN` | **18** (D18/PD3) | `pins_RAMPS.h:106` (default, совпал с измерением) |
| `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` | ✅ | `Configuration.h:832` |
| `Z_MIN_ENDSTOP_INVERTING` | `true` в `Configuration.h:657` → **`false`** после `Conditionals_LCD.h:539` | — |
| `Z_MIN_PROBE_ENDSTOP_INVERTING` | `true` в `Configuration.h:661` → **`false`** после `Conditionals_LCD.h:536` | — |
| `ENDSTOPPULLUPS` | ✅ | `Configuration.h:628` |
| `AUTO_BED_LEVELING_BILINEAR` | ✅ | `Configuration.h:1208` |
| `Z_SAFE_HOMING` | ✅ | `Configuration.h:1360` |
| `NOZZLE_TO_PROBE_OFFSET` | `{46, 14, 0}` | `Configuration.h:960` (без изменений) |

### BLTouch-логика (подтверждено по коду)

- **`src/feature/bltouch.cpp:93-99`** — `triggered()`:
  ```cpp
  #if ENABLED(Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN)
    READ(Z_MIN_PIN) != Z_MIN_ENDSTOP_INVERTING
  #else
    READ(Z_MIN_PROBE_PIN) != Z_MIN_PROBE_ENDSTOP_INVERTING
  #endif
  ```
  → В нашем случае: `READ(Z_MIN_PIN) != false` → **HIGH = TRIGGERED**

- **`src/inc/Conditionals_LCD.h:534-540`** — BLTOUCH принудительно:
  ```c
  // Always disable probe pin inverting for BLTouch
  #undef Z_MIN_PROBE_ENDSTOP_INVERTING
  #define Z_MIN_PROBE_ENDSTOP_INVERTING false
  #if ENABLED(Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN)
    #undef Z_MIN_ENDSTOP_INVERTING
    #define Z_MIN_ENDSTOP_INVERTING false
  #endif
  ```

- **`src/module/endstops.cpp:521`** — общий endstop: `READ(pin) != _ENDSTOP_INVERTING` — та же логика

- **`src/inc/SanityCheck.h:1175-1176`** — требование:
  ```c
  #elif DISABLED(BLTOUCH_SET_5V_MODE) && NONE(ONBOARD_ENDSTOPPULLUPS, ENDSTOPPULLUPS, ENDSTOPPULLUP_ZMIN, ENDSTOPPULLUP_ZMIN_PROBE)
    #error "BLTOUCH without BLTOUCH_SET_5V_MODE requires ENDSTOPPULLUPS..."
  ```
  → Удовлетворяется: `Configuration.h:628` `#define ENDSTOPPULLUPS` ✅

- **`src/inc/Conditionals_post.h:1206-1209`** — при `ENDSTOPPULLUPS` + `USE_ZMIN_PLUG` → `#define ENDSTOPPULLUP_ZMIN`

- **`src/module/endstops.cpp:140-141`** — `SET_INPUT_PULLUP(Z_MIN_PIN)` при `ENDSTOPPULLUP_ZMIN` ✅

- **`src/module/servo.cpp:41`** — `servo[0].attach(SERVO0_PIN)` — подтверждено, что `SERVO0_PIN` используется для `Z_PROBE_SERVO_NR 0`

### Tone.cpp (подтверждено)

| Факт | Файл:строка |
|---|---|
| `LIB_CXXSRC` НЕ содержит `Tone.cpp` | `Makefile:613` |
| `find src -name '*.cpp'` автоматически подхватывает | `Makefile:746-747` |
| `buzzer.cpp` вызывает `::tone(BEEPER_PIN, ...)` | `src/libs/buzzer.cpp:71` |
| `tone()` для AVR определена в | `src/libs/Tone.cpp:243` |
| DUE-вариант `Tone.cpp` не конфликтует | `src/HAL/DUE/Tone.cpp:28` — `#ifdef ARDUINO_ARCH_SAM` (не компилируется на AVR) |

### HARDWARE_MOTHERBOARD (подтверждено)

| Факт | Файл:строка |
|---|---|
| Default в Makefile | `Makefile:60` — `HARDWARE_MOTHERBOARD ?= 11` (устаревший) |
| Reальный MOTHERBOARD в Configuration.h | `Configuration.h:131` — `BOARD_RAMPS_14_EFB` = **1020** |
| Командная строка переопределяет `?=` | безопасно, без правки Makefile |

### Прошивка (подтверждено по core 1.8.3)

| Факт | Файл:строка |
|---|---|
| Mega 2560 upload speed | `boards.txt:223` — `upload.speed=115200` |
| Mega 2560 upload protocol | `boards.txt:221` — `upload.protocol=wiring` |
| Mega 2560 bootloader file | `boards.txt:227` — `stk500v2/stk500boot_v2_mega2560.hex` |
| Makefile upload rate (default) | `Makefile:74` — `UPLOAD_RATE ?= 57600` (для mega1280) |
| Makefile programmer | `Makefile:75` — `AVRDUDE_PROGRAMMER ?= arduino` |
| Makefile upload port | `Makefile:77` — `UPLOAD_PORT ?= /dev/ttyUSB0` |

---

## B. Что подтверждено физическими измерениями

| Сигнал | TQFP pin | Порт.пин | Pin Arduino | Назначение |
|---|---|---|---|---|
| Z-S | 46 | PD3 | **D18** | BLTouch PROBE (= Z-min) |
| X-S | 7 | PE5 | **D3** | X endstop (X−) |
| X+ | 1 | PG5 | **D4** | filament runout |
| J1-V / Z-V | — | — | +5V | питание |
| J1-G / Z-G | — | — | GND | масса |

> **ОТЗЫВАЕТСЯ** (2026-09-21): старая запись «J1-S → D3 = BLTouch SERVO» **неверна** — D3 это концевик **X−** (подтверждено пользователем: BLTouch = разъёмы Z−/Z+, X− = xmin, X+ = filament). «Серво-команды» BLTouch эмулируются на линии D18 (`Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN`), физический провод серво **нет**.

Конфликтов нет. Не пересматривается.

---

## C. Полярность `Z_MIN_PROBE_ENDSTOP_INVERTING`

**Итог: `true` в `Configuration.h` НЕ имеет значения — Marlin 2.0.5.4 принудительно переопределяет в `false` для BLTOUCH.**

**Доказательство:**
- `src/inc/Conditionals_LCD.h:536` — `#define Z_MIN_PROBE_ENDSTOP_INVERTING false` (внутри `#if ENABLED(BLTOUCH)`)
- `src/inc/Conditionals_LCD.h:539` — `#define Z_MIN_ENDSTOP_INVERTING false` (внутри `#if ENABLED(Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN)`)

**Что это значит физически:**
- BLTouch при срабатывании **вытягивает вход вверх** (HIGH)
- При `Z_MIN_ENDSTOP_INVERTING = false`: `READ(D18) != false` → HIGH = TRIGGERED ✅
- Это корректно для BLTouch (open-drain + pull-up)

**Не нужно менять `Configuration.h:657` и `Configuration.h:661`** — их значения будут переопределены.

---

## D. Конфигурация BLTouch — финальный набор

Все необходимые `#define` **уже активны**:

```c
// Configuration.h (существующие, без изменений):
#define MOTHERBOARD BOARD_RAMPS_14_EFB       // 131
#define BLTOUCH                               // 888
#define Z_PROBE_SERVO_NR 0                    // 882
#define Z_SERVO_ANGLES { 10, 90 }            // 891
#define Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN   // 832
#define ENDSTOPPULLUPS                        // 628
#define AUTO_BED_LEVELING_BILINEAR            // 1208
#define Z_SAFE_HOMING                         // 1360
#define NOZZLE_TO_PROBE_OFFSET { 46, 14, 0 }  // 960

// pins_RAMPS.h (default, действуют):
#define Z_MIN_PIN 18       // pins_RAMPS.h:106 (D18, Z− разъём, белый триггер)
#define X_MIN_PIN 3        // pins_RAMPS.h default (явный define в Configuration.h, D3)
// Configuration.h (наши overrides):
#define SERVO0_PIN 19      // D19, Z+ разъём, жёлтый servo-провод (USER-CONFIRMED 2026-09-21)
#define FIL_RUNOUT_PIN 4   // X+ порт → D4
```

**Ничего не нужно добавлять или менять.**

---

## E. Конфликты пинов

| Pin | Назначение | Кто использует в прошивке | Конфликт |
|---|---|---|---|
| **D3** | X− endstop | `X_MIN_PIN` (явный define, = default RAMPS) | ❌ Нет |
| **D4** | filament runout | `FIL_RUNOUT_PIN` (`Configuration.h`) | ❌ Нет |
| **D19** | BLTouch SERVO (жёлтый) | `SERVO0_PIN` (override 19) | ❌ Нет |
| **D18** | BLTouch TRIGGER (белый) | `Z_MIN_PIN` (`pins_RAMPS.h:106`) | ❌ Нет |

**Проверка по `pins_RAMPS.h` (полный список):**
- D3 — `X_MIN_PIN`, совпадает с X− пользователем ✅
- D4 — `X_MAX_PIN` (не используется) / `FIL_RUNOUT_PIN` ✅
- D19 — `SERVO0_PIN` (override 19, жёлтый servo-провод) ✅
- D18 — только `Z_MIN_PIN` ✅
- `BEEPER_PIN` = 37 (`pins_RAMPS.h:461`) — не конфликтует ✅

**Подтверждено, что `SERVO0_PIN` используется именно как servo для `Z_PROBE_SERVO_NR 0`:**
- `src/module/servo.cpp:41` — `servo[0].attach(SERVO0_PIN)`
- `src/feature/bltouch.cpp:42` — `MOVE_SERVO(Z_PROBE_SERVO_NR, cmd)` → `servo[0]` = `SERVO0_PIN` ✅

---

## F. Tone.cpp

**Решение: ОСТАВИТЬ как есть.**

| Вопрос | Ответ | Файл:строка |
|---|---|---|
| `Tone.cpp` в `LIB_CXXSRC`? | **Нет** | `Makefile:613` |
| Автоматически подхватывается? | **Да**, через `find src -name '*.cpp'` | `Makefile:747` |
| Нужен для линковки? | **Да** — `buzzer.cpp:71` вызывает `tone()` | `src/libs/buzzer.cpp:71` |
| Коллизия с DUE-`Tone.cpp`? | **Нет** — DUE-вариант под `#ifdef ARDUINO_ARCH_SAM` | `src/HAL/DUE/Tone.cpp:28` |
| Альтернатива (Makefile-строка)? | `LIB_CXXSRC += Tone.cpp` — **не требуется**, текущее работает | — |

---

## G. Сборка

**Команда (WSL Ubuntu-24.04):**
```bash
cd /mnt/f/git/jg_maker_magic_v03_bltouch/Marlin && rm -rf applet && make -j4 ARDUINO_INSTALL_DIR=/home/vivakalman/Arduino HARDWARE_MOTHERBOARD=1020
```

**Результат (финальный build, 2026-09-21):**
```
Program: 184252 bytes (70.3% Full)
Data:      6762 bytes (82.5% Full)
```
(исторический build от 2026-09-18: 177206 / 6681)

**Файлы:**
- `F:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.hex` (498 KB)
- `F:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.elf`

---

## H. Прошивка через USB

### Bootloader
- Плата ATmega2560 — **требует bootloader** (Arduino Mega 2560 / stk500v2)
- Bootloader занимает последние 4 KB flash (адрес 0x3F000–0x3FFFF)
- **Прошивка по USB = загрузка через bootloader** (не ISP)
- **Риск потерять bootloader при обычной USB-прошивке: НЕТ** — avrdude с `-c arduino` пишет только в main flash. Bootloader не затрагивается.

### Параметры (из `boards.txt` + Makefile)
| Параметр | Значение | Источник |
|---|---|---|
| MCU | `atmega2560` | `Makefile:508` |
| Programmer | `arduino` (stk500v1 → bootloader) | `Makefile:75` + `boards.txt:221` |
| Baud rate | **115200** | `boards.txt:223` |
| Port (Windows) | `COMx` (определить в Диспетчере устройств) | — |
| Port (WSL/Linux) | `/dev/ttyACM0` или `/dev/ttyUSB0` | — |
| Reset | **Не нужен** для `-c arduino` | — |
| Fuses | Не менять | — |

### Команда `avrdude` (НЕ выполнена, только показана)

**Вариант 1 — из Windows (PowerShell/CMD):**
```
avrdude -c arduino -p atmega2560 -b 115200 -P COM3 -U flash:w:F:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.hex:i
```

**Вариант 2 — из WSL (если порт проброшен):**
```
avrdude -c arduino -p atmega2560 -b 115200 -P /dev/ttyACM0 -U flash:w:/mnt/f/git/jg_maker_magic_v03_bltouch/Marlin/applet/Marlin.hex:i
```

> ⚠️ **`COM3` — пример. Подставить свой порт.**

### Как определить COM-порт в Windows
1. `devmgmt.msc` → Диспетчер устройств → Порты (COM и LPT)
2. Искать: «Arduino Mega» / «USB Serial» / «CH340» / «ATmega-USB»
3. Или: `wmic path win32_SerialPort get Description,DeviceID` (PowerShell)
4. Или: `Get-CimInstance Win32_SerialPort | Select Description, DeviceID`

### Нужен ли reset?
- **Нет.** Bootloader Arduino Mega 2560 (stk500v2) входит в режим ожидания при правильном baud (115200). avrdude с `-c arduino` автоматически посылает handshake.
- Если не работает: отключить USB на 3 сек → подключить → сразу запустить avrdude (у bootloader окно ~10 сек).

### Есть ли риск потерять bootloader?
- **Нет.** Флаг `-U flash:w:...` пишет в main flash только.
- Для перезаписи bootloader нужен отдельный `-U signature` или `-U efuse`/`-U hfuse` — мы **не** делаем этого.

---

## I. Что проверить непосредственно перед первой прошивкой

| # | Проверка | Как |
|---|---|---|
| 1 | BLTouch **отключён** от J1 | Визуально |
| 2 | X-endstop **отключён** или свободен | Визуально |
| 3 | Печатная плата не подключена к принтеру (если можно) | — |
| 4 | USB-кабель (данные, не только питание) | Проверить |
| 5 | COM-порт определён | Диспетчер устройств / `wmic` |
| 6 | avrdude установлен и в PATH | `avrdude --version` в PowerShell |
| 7 | Hex-файл существует | `dir F:\git\jg_maker_magic_v03_bltouch\Marlin\applet\Marlin.hex` |
| 8 | **НЕ** держать RESET при USB-подключении (bootloader не нужен) | — |
| 9 | После прошивки: подключить BLTouch → X-endstop → нагреть → `G28 Z` → проверить срабатывание | — |

---

## Изменённые файлы (итого)

| Файл | Статус | Изменение |
|---|---|---|
| `Marlin/Configuration.h` | **изменён (2026-09-21)** | удалён override `X_MIN_PIN 2` (теперь D3); **`SERVO0_PIN 19` явный override (D19, REAL WIRE, разъём Z+ — жёлтый)**; добавлен `FIL_RUNOUT_PIN 4` (D4); `FIL_RUNOUT_INVERTING true`; `X_MIN_ENDSTOP_INVERTING true` |
| `Marlin/src/libs/Tone.cpp` | **новый** | копия из Arduino core 1.8.3 (untracked) |
| `REPORT_bltouch_v1.1.md` | **новый** | этот отчёт |

**Не тронуты:** Makefile, `pins_RAMPS.h`, `bltouch.cpp`, `endstops.cpp`, E-steps, thermistor, PID, bed size, `NOZZLE_TO_PROBE_OFFSET`, экструзия.

**Commit: НЕ выполнялся.**
**Прошивка: ВЫПОЛНЕНА (2026-09-21)** — 184252 bytes прошиты на COM4, verified (SHA-256 `433E101CD54B29FEDDEAA79F39607BC0096AA8AAF5B0944F9A5E66A2BB9FD38B`; включает INVERT_X_DIR=true — см. `docs/FLASHING/FLASH_REPORT.md`).
