# JG Maker Magic V1.1 — DEFINITIVE PIN MAP (ФИНАЛЬНАЯ КАРТА ПИНОВ)

> **Статус:** USER-CONFIRMED 2026-09-21. Этот документ — единичный источник истины (single source of truth) по проводке и пинам.
> **MCU:** ATmega2560 TQFP-100 · **Board:** `BOARD_RAMPS_14_EFB` = 1020 · **Firmware:** Marlin 2.0.5.4 "Magic v0.3.3"

---

## 1. Физическая проводка (подтверждена владельцем)

### Разъём Z− (RAMPS Z-min port, сигнал = D18)

| Провод | Контакт разъёма | Назначение |
|---|---|---|
| **чёрный** | G | GND |
| **белый** | S (Signal) | **BLTouch TRIGGER** → D18 |
| (5V) | V | +5V (питание BLTouch) |

### Разъём Z+ (RAMPS Z-max port, сигнал = D19)

| Провод | Контакт разъёма | Назначение |
|---|---|---|
| **жёлтый** | S (Signal) | **BLTouch SERVO (управление)** → D19 |
| **зелёный** | G | GND |
| **красный** | V | +5V |

### Разъём X− (RAMPS X-min port, сигнал = D3)

| Пин чипа | Назначение |
|---|---|
| **D3 = PE5 = TQFP pin 7** | **X- концевик (endstop)** — подтверждено владельцем «раз 50» и прозвоном (pin 7 ≈ 1 Ω) |

### Разъём X+ (RAMPS X-max port, сигнал = D4)

| Пин чипа | Назначение |
|---|---|
| **D4 = PE6 = TQFP pin 8** | **Filament runout sensor** |

---

## 2. Итоговая карта «функция → пин → прошивка»

| Функция | Arduino-пин | Порт чипа | TQFP-100 pin | Прошивка |
|---|---|---|---|---|
| X- endstop | **D3** | PE5 | **7** | `X_MIN_PIN 3` (явный define, = default RAMPS) |
| Filament runout | **D4** | PE6 | **8** | `FIL_RUNOUT_PIN 4` |
| BLTouch trigger | **D18** | PD3 | **46** | `Z_MIN_PIN 18` + `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` |
| BLTouch servo | **D19** | PD4 | **47** | `SERVO0_PIN 19` |

**Конфликтов нет**: D3 ≠ D4 ≠ D18 ≠ D19 — все четыре пина разные, каждый один раз.

---

## 3. Отзываются (RETRACTED) старые утверждения

| Старое утверждение | Где было | Вердикт |
|---|---|---|
| «X− endstop = D2 (PE4)» | git-комментарий `X_MIN_PIN 2` | **ЛОЖЬ** — реальный пин PE5 = **D3** |
| «J1-S (BLTouch servo) = D3 / pin 7» | старые отчёты, `SERVO0_PIN 3` | **ЛОЖЬ** — D3 = X− endstop |
| «SERVO0_PIN = 15, виртуальный, без провода» | отчёты от 2026-09-18/20 | **ЛОЖЬ** — servo-провод жёлтый, реально на **D19** |
| «X_min застревает TRIGGERED — физика» | диагностика 2026-09-21 | было следствием **неверного пина** (D2 вместо D3): прошивка читала мёртвый пин |
| «INVERT_X_DIR true» | тестовый прогон 2026-09-21 | **НЕ НУЖНО** — стоковое `false` подтверждено правильным (X− едет влево к упору) |

---

## 4. Актуальные defines в `Configuration.h` (2026-09-21)

```c
#define X_MIN_PIN 3            // D3 = PE5 = TQFP pin 7 — X- endstop (USER-CONFIRMED)
#define FIL_RUNOUT_PIN 4       // D4 = PE6 = TQFP pin 8 — runout sensor
#define SERVO0_PIN 19          // D19 = PD4 = TQFP pin 47 — BLTouch servo (жёлтый)
// Z_MIN_PIN = 18 (default pins_RAMPS.h) — D18 = PD3 = TQFP pin 46 — BLTouch trigger (белый)
#define X_MIN_ENDSTOP_INVERTING true
#define INVERT_X_DIR false     // СТОК, ПРАВИЛЬНО (X- → влево, к упору)
#define Z_SERVO_ANGLES { 10, 90 }
```

Временно выключено для тестов (вернуть после проверки):
```c
//#define NO_MOTION_BEFORE_HOMING   // OFF during bring-up (X endstop validation)
```

---

## 5. История изменений (2026-09-21, хронология)

1. **утро** — X двигался «не туда». Тесты показали: стоковое `INVERT_X_DIR false` — **правильное направление**.
2. **день** — итеративное накатывание изменений. `SERVO0_PIN 15` + `FIL_RUNOUT_PIN 4` → X «сломался» → откат → снова работал. Причина: servo-команды уходили на D15 (куда BLTouch не подключён) — BLTouch не управлялся.
3. **вечер** — владелец подтвердил реальную проводку: Z− (чёрный GND / белый trigger → D18), Z+ (жёлтый servo → D19), X− = **PE5 = D3**.
4. **финал** — `X_MIN_PIN 3`, `SERVO0_PIN 19`, `FIL_RUNOUT_PIN 4`. Сборка 183580 bytes. **Прошивка ожидала USB-адаптер CH340 (COM4 пропал — требуется переподключение).**
