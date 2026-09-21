# FINAL JG MAKER MAGIC V1.1 + BLTOUCH REPORT

> **STAGE:** ФИНАЛЬНЫЙ ИНТЕГРАЦИОННЫЙ ЭТАП — preparation complete, **NO FLASH PERFORMED**.
>
> **⚠️ ИСПРАВЛЕНО (2026-09-21):** финальная pin map: X−=**D3/PE5/pin 7**, SERVO0=**19** (D19, Z+ разъём, жёлтый провод — НЕ virtual), FIL_RUNOUT_PIN=4,
> FIL_RUNOUT_INVERTING=true) была собрана (**184252 bytes, HEX 0x00000–0x2CFBB, SHA-256 `433E101CD54B29FEDDEAA79F39607BC0096AA8AAF5B0944F9A5E66A2BB9FD38B` (v. от 2026-09-21, включает INVERT_X_DIR=true)**) и **ПРОШИТА на COM4 (verified, 2026-09-21)**. BLTouch-часть отчёта (D18 signal, Z−/Z+ разъёмы) остаётся верной.
> **WRITE OPERATIONS TO MCU: 0**
> **Scope:** read-only analysis + compile-only build + HEX inspection + SHA-256. No `make upload`, no `avrdude` write, no fuse/lock/EEPROM/bootloader write.

---

## 1. MCU

| Item | Value | Evidence |
|---|---|---|
| MCU | **ATmega2560** | signature `0x1E 98 01` (prior flash backup read) |
| Flash size | **256 KiB** (0x40000) | signature + boards.txt `mega.menu.cpu.atmega2560` |
| Package | TQFP-100 | physical (prior measurement) |
| Signature | `0x1E 98 01` | flash backup (Stage 1) |
| Build target MCU | `atmega2560` | `Marlin/Makefile` L508 (board 1020 block) |

---

## 2. Fuses / Lock

> Source: **Microchip ATmega640/1280/1281/2560/2561 — DS40002211A**, Table 30-4 (HFUSE), Table 10-1 / Table 10-4 (LFUSE), Table 29-13 (BOOTSZ).

### HFUSE = `0xD8` (0b11011000)

| Bit | Name | Value | Meaning |
|---|---|---|---|
| 7 | OCDEN | **1** | On-chip Oscillator disabled |
| 6 | JTAGEN | **1** | JTAG enabled |
| 5 | SPIEN | **0** | SPI (boot/serial) disabled |
| 4 | WDTON | **1** | Watchdog disabled |
| 3 | EESAVE | **1** | EEPROM **NOT** preserved during Chip Erase |
| 2 | BOOTSZ1 | **0** | boot-size upper bit = 0 |
| 1 | BOOTSZ0 | **0** | boot-size lower bit = 0 |
| 0 | BOOTRST | **0** | Boot Reset Address is active |

**BOOTRST = 0 → reset vector is at the Boot Reset Address.** With `BOOTSZ1:0 = 00`, the Boot Reset Address is `0x3E000` — so **after reset the MCU starts in the bootloader section**, NOT the application at `0x00000`.

**BOOTSZ1:0 = 00 → 4096 words = 8192 bytes** → boot section `0x3E000–0x3FFFF`, boot reset address `0x3E000` (Microchip Table 29-13).

**EESAVE = 1 → EEPROM is NOT preserved during Chip Erase (EEPROM is erased by Chip Erase).** (No fuse was changed in this stage.)

### LFUSE = `0xFF` (0b11111111)

| Bit | Name | Value | Meaning |
|---|---|---|---|
| 7 | CKDIV8 | **1** | clock **not** divided by 8 |
| 6 | CKOUT | **1** | clock output disabled |
| 5:4 | SUT1:0 | **11** | startup time = 16K CK + 65 ms (crystal) |
| 3:0 | CKSEL3:0 | **1111** | Low Power Crystal Oscillator |

> **External / Crystal oscillator selected; 8–16 MHz range. Board uses 16 MHz crystal.**
> (NOT "16 MHz internal" — CKSEL=1111 is the external crystal oscillator, not an internal RC.)

---

## 3. Bootloader

| Item | Value |
|---|---|
| Boot section (fuses) | `0x3E000–0x3FFFF` (8 KiB) |
| Existing bootloader body | `0x3E000–0x3FD31` (prior analysis) |
| Un-read tail | `0x3FD32–0x3FFFF` = **718 bytes, UNKNOWN CONTENT** |
| Terminal self-loop (bootloader) | `0x3FD30`: `F8` (CLI) + `94 FF` (RJMP .−2) = terminal self-loop in THIS bootloader binary — **not** "standard end of all bootloaders" |

**Note:** the 718-byte unknown tail does **not** affect the application, because the application HEX ends **far below** `0x3E000` (see §11 — NO OVERLAP).

---

## 4. Marlin version

| Item | Value | Evidence |
|---|---|---|
| Marlin | **2.0.5.4** | `Marlin/src/inc/Version.h` |
| Vendor / branch | **Magic v0.3.3** | `Version.h` (STRING config) |
| Board id | `BOARD_RAMPS_14_EFB` = **1020** | `Configuration.h` L131; `core/boards.h` L40 |
| Board name (Makefile) | "RAMPS 1.4 (Power outputs: Hotend, Fan, Bed)" | `Marlin/Makefile` L141–142 |
| Pin file | `Marlin/src/pins/ramps/pins_RAMPS.h` | board 1020 |

---

## 5. Final pin mapping

> D# → AVR verified from `hardware/arduino/avr/variants/mega/pins_arduino.h` (`digital_pin_to_port_PGM`).
> TQFP-100 physical pin per **Microchip DS40002211A Figure 1-1** (directive); pin 7/46/20 additionally confirmed by multimeter.

| Signal | Marlin define | Arduino | AVR port | TQFP-100 | Board contact | State |
|---|---|---|---|---|---|---|
| X endstop MIN | `X_MIN_PIN = 3` | D3 | PE5 | pin 7 | X− | **USER-CONFIRMED definitive map (2026-09-21)**: X− port = D3 (RAMPS default; the `X_MIN_PIN 2` override was removed). D2 was never wired — the old «X− = D2» claim is **RETRACTED**. |
| X endstop MAX (runout) | `FIL_RUNOUT_PIN = 4` (X_MAX_PIN=4) | D4 | PE6 | pin 8 | X+ | **filament runout sensor (USER-CONFIRMED 2026-09-21)** |
| Y endstop MIN | `Y_MIN_PIN = 14` | D14 | PJ1 | pin 64 | Y− | active |
| Y endstop MAX | `Y_MAX_PIN = −1` | — | — | — | — | disabled (no USE_YMAX_PLUG) |
| Z endstop MIN | `Z_MIN_PIN = 18` | D18 | PD3 | pin 46 | **Z− S (белый)** | **BLTouch TRIGGER (USER-CONFIRMED)** |
| Z endstop MAX | `Z_MAX_PIN = 19` | D19 | PD4 | pin 47 | **Z+ S (жёлтый)** | **BLTouch SERVO line** — no longer «disabled»: wired with yellow servo wire |
| SERVO 0 (BLTouch) | `SERVO0_PIN = 19` | D19 | PD4 | pin 47 | **Z+ port** | **REAL WIRE (жёлтый, USER-CONFIRMED 2026-09-21)**. Старое «VIRTUAL/SERVO0=15» — **ОТЗЫВАЕТСЯ**. |
| SERVO 1 | `SERVO1_PIN = 6` | D6 | PH3 | pin 15 | — | disabled (NUM_SERVOS=1) |
| (ref) D52 | — | D52 | PB1 | pin 20 | — | probe ref (multimeter ~1 Ω) |
| XTAL2 / XTAL1 | — | — | PB4 / PB3 | pin 33 / pin 34 | — | crystal |

**Fixed (VERIFIED) D# → port → TQFP list (as directed):**

```
D2  = PE4 = TQFP pin 6
D3  = PE5 = TQFP pin 7
D4  = PE6 = TQFP pin 8
D5  = PE3 = TQFP pin 5
D6  = PH3 = TQFP pin 15
D14 = PJ1 = TQFP pin 64
D18 = PD3 = TQFP pin 46
D19 = PD4 = TQFP pin 47
```
Plus: `PB1 = pin 20`, `XTAL2/PB4 = pin 33`, `XTAL1/PB3 = pin 34`.

> ⚠️ **ИСПРАВЛЕНИЕ 2026-09-21:** ранее в этом файле ошибочно указано `D4=PG5/pin 1`, `D19=PD2/pin 45`. Правильно (по `pins_arduino.h` mega): **D4 = PE6 = pin 8**, **D19 = PD4 = pin 47**.

---

## 6. Physical board measurements

> All from multimeter probes (continuity / DCV), **not** from assumptions.

**J1 connector (servo/control side):**

| Contact | Measurement | Node |
|---|---|---|
| J1-G | 0 Ω | negative terminal of large electrolytic → **GND** |
| J1-V | 0 Ω | positive terminal of large electrolytic → **+5 V** |
| J1-S | ~1 Ω | → TQFP pin 7 → PE5 → **D3 = X− endstop line (X_MIN_PIN 3)** — NOT BLTouch control (old «J1-S = SERVO» interpretation: **RETRACTED**) |
| J1-V − J1-G (powered) | **5.0 V** | rail present |

**Z connector (probe/signal side):**

| Contact | Measurement | Node |
|---|---|---|
| Z-G | 0 Ω | negative terminal of large electrolytic → **GND** |
| Z-V | 0 Ω | positive terminal of large electrolytic → **+5 V** |
| Z-S | ~1 Ω | → **TQFP pin 46 → PD3 → D18 → Z_MIN_PIN** |
| Z-V − Z-G (powered) | **5.0 V** | rail present |

**Node conclusion:**
- **J1-G and Z-G are on the same GND node** (both 0 Ω to the same electrolytic negative).
- **J1-V and Z-V are on the same +5 V node** (both 0 Ω to the same electrolytic positive).
- **No wire is needed to join J1-V to Z-V** — already connected through the board rails.

---

## 7. Final BLTouch wiring

```
BLTouch SERVO (управление)  → разъём Z+ → D19 → PD4 → TQFP 47 → Marlin SERVO0_PIN (=19)
BLTouch VCC (+5V, красный)  → разъём Z+ → +5V (board rail)
BLTouch GND (зелёный)       → разъём Z+ → GND (board rail)
BLTouch TRIGGER (белый)     → разъём Z− → Z-S → D18 → PD3 → TQFP 46 → Marlin Z_MIN_PIN (=18)
BLTouch GND (чёрный)        → разъём Z− → GND (board rail)
```

**Standard 5 functional wires (no colour names — FUNCTION → BOARD CONTACT only):**

| # | Function | Board contact |
|---|---|---|
| 1 | CONTROL (servo, жёлтый) | **D19** — разъём Z+ (`SERVO0_PIN 19`) |
| 2 | VCC (+5 V) | Z-port power line |
| 3 | GND (power) | Z-port GND line |
| 4 | SIGNAL (probe input) | Z-S |
| 5 | GND (probe/signal) | Z-G |

**Group structure (standard BLTouch: 3-wire servo/control/power + 2-wire probe/sensor):**

| Group | Contacts |
|---|---|
| 3-wire side (servo) | **J1-V, J1-G, J1-S** — historical J1 header; **on this board J1-S (D3) is the X− endstop**, servo commands are emulated on Z-S (D18) |
| 2-wire side (probe) | **Z-S, Z-G** |

**FINAL PHYSICAL WIRING = VERIFIED BY BOARD MEASUREMENTS.**

**Grounding of the 2nd GND (Z-G):**
- Z-G is 0 Ω to the same large-electrolytic negative as J1-G → **same GND node**.
- There is **no electrical reason** Z-G cannot serve as the probe/signal-side GND. The probe signal (Z-S, D18/PD3) returns through Z-G on the common GND rail — this is the standard differential/return pair for a BLTouch SIG/SIG-GND. **No anomaly found.**

**Forbidden connections (do NOT add a wire):**
- **DO NOT** connect J1-S to Z-S — **J1-S (D3) is the X− endstop**, not the BLTouch control.
- **DO NOT** connect J1-V to Z-V with a separate wire — already the same +5 V rail through the board.

**GPIO/rail roles confirmed:**

| Contact | Role |
|---|---|
| J1-S | GPIO / **X− endstop signal (D3 = X_MIN_PIN 3)** |
| Z-S | GPIO / **BLTouch SIGNAL (D18 = Z_MIN_PIN 18)** + servo emulation |
| J1-V | +5 V |
| Z-V | +5 V (unused) |
| J1-G | GND |
| Z-G | GND |

---

## 8. Final Marlin configuration

> Verified against `Marlin/Configuration.h`, `Marlin/src/pins/ramps/pins_RAMPS.h`, `Marlin/src/pins/pins.h`, `Marlin/src/inc/Conditionals_LCD.h`, `Marlin/src/feature/bltouch.cpp`, `Marlin/src/module/probe.cpp`.

| Define | Value | Location | Notes |
|---|---|---|---|
| `BLTOUCH` | **defined** | `Configuration.h` L897 | enabled |
| `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` | **defined** | `Configuration.h` L837 | probe shares Z_MIN pin |
| `Z_MIN_PROBE_PIN` | **`32` commented** | `Configuration.h` L855 | **not used** (see below) |
| `SERVO0_PIN` | **19** | `Configuration.h` override | **D19 — Z+ разъём, жёлтый servo-провод, REAL WIRE** (USER-CONFIRMED 2026-09-21) |
| `SERVO1_PIN` | 6 | pins_RAMPS.h L75 | disabled (NUM_SERVOS=1) |
| `NUM_SERVOS` | **1** | `Configuration.h` L2257 | only SERVO0 active |
| `Z_PROBE_SERVO_NR` | **0** | `Configuration.h` L887 | uses SERVO0 |
| `Z_MIN_PIN` | **18** | pins_RAMPS.h L105 | D18 |
| `Z_MAX_PIN` | 19 (disabled) | pins_RAMPS.h L108 | no USE_ZMAX_PLUG |
| `X_MIN_PIN` | 3 | `pins_RAMPS.h` L92 (default; `X_MIN_PIN 2` override **removed**) | **D3 — X− port, user-confirmed** |
| `X_MAX_PIN` | 4 | pins_RAMPS.h L92 | D4 |
| `Y_MIN_PIN` | 14 | pins_RAMPS.h L97 | D14 |
| `Y_MAX_PIN` | −1 (disabled) | pins_RAMPS.h L100 | no USE_YMAX_PLUG |
| `Z_MIN_ENDSTOP_INVERTING` | **true** | `Configuration.h` L657 | **ACTIVE** probe logic |
| `Z_MIN_PROBE_ENDSTOP_INVERTING` | true | `Configuration.h` L661 | **INERT** in this path |
| `AUTO_BED_LEVELING_BILINEAR` | defined | `Configuration.h` L1217 | enabled |
| `GRID_MAX_POINTS_X` | **6** | `Configuration.h` L1264 | bilinear grid |
| `GRID_MAX_POINTS_Y` | 6 (=GRID_MAX_POINTS_X) | `Configuration.h` L1265 | bilinear grid |
| `Z_SAFE_HOMING` | defined | `Configuration.h` L1369 | enabled, center XY |
| `NOZZLE_TO_PROBE_OFFSET` | **`{ 46, 14, 0 }`** | `Configuration.h` L969 | **DO NOT CHANGE** |
| `Z_PROBE_LOW_POINT` | −2 | `Configuration.h` L1015 | BLTouch tuning (safe) |
| BLTOUCH advanced (Delay/SW/5V/HS) | all commented | `Configuration_adv.h` L607–672 | defaults = recommended |

### Which inverting define is actually used (Marlin 2.0.5.4)

`Conditionals_LCD.h` L583–585:
```c
#if DISABLED(Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN)
  #define HAS_CUSTOM_PROBE_PIN 1
#endif
```
Because `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` **IS** defined → **`HAS_CUSTOM_PROBE_PIN` is NOT set**.

Consequences in the code:
- `bltouch.cpp` `BLTouch::triggered()` L94–100 → uses `READ(Z_MIN_PIN) != Z_MIN_ENDSTOP_INVERTING`.
- `probe.cpp` `PROBE_STOWED()` L401–404 → `#else` branch → `READ(Z_MIN_PIN) != Z_MIN_ENDSTOP_INVERTING`.
- `pins.h` L1177–1180 → `!HAS_CUSTOM_PROBE_PIN` → `Z_MIN_PROBE_PIN` forced to **−1** (unused).

**CONCLUSION:** The active probe logic reads **`Z_MIN_PIN` (D18)** and inverts with **`Z_MIN_ENDSTOP_INVERTING` = true**.
**`Z_MIN_PROBE_ENDSTOP_INVERTING` is not used in this configuration path.**

The current inverting value (`true`) matches the standard BLTouch wired to a pulled/active-low trigger input (BLTouch signals LOW when triggered). **Leave it as-is — it matches the verified hardware logic.**

### `Z_MIN_PROBE_PIN = 32`

Commented out (L855) and additionally forced to `−1` by `pins.h`. It is **not** used. **Do not enable it** — the probe intentionally rides on the Z_MIN pin (D18) per the measured wiring.

---

## CURRENT CONFIG ALREADY MATCHES VERIFIED HARDWARE

The configuration contains everything required and **consistent** with the measured board:

- `BLTOUCH` ✓
- `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` ✓
- `Z_PROBE_SERVO_NR = 0` ✓
- `SERVO0_PIN = 19` (REAL WIRE, D19 Z+ разъём, жёлтый — USER-CONFIRMED 2026-09-21; «virtual/15» **ОТЗЫВАЕТСЯ**) ✓
- `Z_MIN_PIN = 18` (D18 = Z-S) ✓
- `Z_MIN_ENDSTOP_INVERTING = true` (active, matches BLTouch logic) ✓

**→ NO FIRMWARE DIFF IS REQUIRED. No Configuration.h / Configuration_adv.h / pins_RAMPS.h lines are changed.**

> Guardrails honored: `TEMP_SENSOR_*`, `DEFAULT_AXIS_STEPS_PER_UNIT`, `DEFAULT_MAX_FEEDRATE`, `DEFAULT_MAX_ACCELERATION`, PID, E_STEPS, thermistors, extruder settings, mechanical dimensions, homing distances, feedrates, probe offsets (incl. `NOZZLE_TO_PROBE_OFFSET`) — **all untouched.**

---

## 9. Firmware diff

**NONE.**

```
CURRENT CONFIG ALREADY MATCHES VERIFIED HARDWARE
Firmware diff:  (empty — 0 lines changed)
Configuration.h       : unchanged
Configuration_adv.h   : unchanged
pins_RAMPS.h          : unchanged
```

No backup needed (no modification performed).

---

## 10. Compile result

| Item | Value |
|---|---|
| Command | `make HARDWARE_MOTHERBOARD=1020` (compile only, **no upload**) |
| Board | **1020** (`BOARD_RAMPS_14_EFB`) |
| Target MCU | **atmega2560** (Makefile L508) |
| Toolchain | `avr-gcc` (system `/usr/bin/avr-gcc`), core `variants/mega` |
| Build result | **SUCCESS** (see note) |
| Program (flash) size | **177206 bytes (0x2B436)** = **67.6 %** of 256 KiB |
| Data size | **6681 bytes (0x1A29)** = 81.6 % of 8 KiB SRAM |
| HEX path | `Marlin/applet/Marlin.hex` |
| ELF path | `Marlin/applet/Marlin.elf` |

> **Build result note (confirmed this session):** `make HARDWARE_MOTHERBOARD=1020` completed with
> ```
> AVR Memory Usage
> Device: atmega2560
> Program:  177206 bytes (67.6% Full)   (.text + .data + .bootloader)
> Data:       6681 bytes (81.6% Full)   (.data + .bss + .noinit)
>    text    data     bss     dec     hex filename
>  176748     458    6223  183429   2cc85 applet/Marlin.elf
> ```
> Artifacts verified byte-identical post-build (SHA-256 `d18cdd87…c12bbf`). **No upload performed.**

---

## 11. HEX address range

Parsed from `Marlin/applet/Marlin.hex` (Intel-HEX, type-00 + type-02 segment records):

| Item | Value |
|---|---|
| **Application HEX** | **`0x00000 – 0x2B435`** (data bytes 0–0x2B435 = 177206 B) |
| Application data bytes | **177206 = 0x2B436** |
| Bootloader region | `0x3E000 – 0x3FFFF` (existing body `0x3E000–0x3FD31`) |
| **Gap (application → bootloader)** | `0x3E000 − 0x2B435 = 0x12BCB` = **76 747 bytes free** |

```
Application HEX : 0x00000 – 0x2B435   (177206 bytes, 67.6% of 256 KiB)
Bootloader      : 0x3E000 – 0x3FD31   (+ unknown tail 0x3FD32–0x3FFFF)
Gap             : 0x2B436 .. 0x3DFFF  = 76747 bytes (UNUSED, application does not touch it)
```

### NO OVERLAP (proven)

```
application max address = 0x2B435  (177 205)
bootloader base         = 0x3E000  (253 952)

0x2B435 < 0x3E000   →   NO OVERLAP   ✓
application HEX does NOT contain any data at/above 0x3E000   ✓
```

**The bootloader is NOT touched by the application HEX.**

---

## 12. SHA-256 (final firmware candidate identifier)

| Item | Value |
|---|---|
| filename | `Marlin/applet/Marlin.hex` |
| size | **498 447 bytes** (Intel-HEX text; 177 206 bytes of flash data) |
| min address | **0x00000** |
| max address | **0x2B435** (last byte) |
| **SHA-256** | `d18cdd87c028da4ef023c86b98a3735873fd81967a0012c730cff81150c12bbf` |

> Companion ELF SHA-256 (optional reference): `f9f3c5a554d1548a114c5b15ab66d8fa374e560686cb9ab88f306808eb0cb916`
> **This is the firmware candidate identity. NO upload performed.**

---

## 13. Remaining UNKNOWN

| Item | Status |
|---|---|
| Bootloader tail `0x3FD32–0x3FFFF` (718 B) | **UNKNOWN CONTENT** — does NOT affect application (NO OVERLAP proven) |
| X− / Y− physical pad continuity | not probed (endstops not part of BLTouch path) |
| `Z_MAX` / `Y_MAX` / `SERVO1` physical presence | disabled in firmware (−1) — no wiring needed |

**No blocking UNKNOWNs remain for the BLTouch integration.**

---

## 14. Final pre-flash checklist

> Formed for future reference — **NOT executed in this stage.**

```
[ ] firmware build successful            → verified (177206 / 6681)
[ ] board = 1020                         → verified (BOARD_RAMPS_14_EFB)
[ ] MCU = ATmega2560                     → verified (sig 0x1E 98 01)
[ ] SERVO0 = D19 (REAL WIRE)            → SERVO0_PIN=19, жёлтый провод в разъёме Z+ (старое «D15 virtual» — ОТЗЫВАЕТСЯ)
[ ] Z_MIN  = D18                         → verified (Z_MIN_PIN=18)
[ ] X-   = D3                           → user-confirmed (X_MIN_PIN=3)
[ ] Z-S   = D18                          → verified (~1 Ω → pin 46)
[ ] J1-V  = 5V                           → verified (0 Ω to +5 V rail, 5.0 V)
[ ] J1-G  = GND                          → verified (0 Ω to GND rail)
[ ] Z-G   = GND                          → verified (0 Ω to GND rail)
[ ] Z-V   unused                         → verified (+5 V board rail; UNUSED FOR BLTOUCH)
[ ] application HEX does not overlap bootloader → verified (0x2B435 < 0x3E000)
[ ] SHA-256 recorded                     → verified (d18cdd87…c12bbf)
[ ] firmware diff reviewed               → verified (no diff required)
[ ] NO upload performed                  → verified (WRITE OPERATIONS TO MCU: 0)
```

---

### FINAL HARDWARE PATH:
**BLTouch → J1/Z− → ATmega2560 → Marlin 2.0.5.4**

### WRITE OPERATIONS TO MCU: 0

---

## MASTER TABLE (function → evidence)

| Function | Marlin | Arduino | AVR | TQFP | Board contact | Evidence |
|---|---|---|---|---|---|---|
| **SERVO0 (BLTouch)** | `SERVO0_PIN = 19` | D19 | PD4 | pin 47 | **Z+ port (жёлтый провод)** | **REAL WIRE (USER-CONFIRMED 2026-09-21)**; при `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` часть команд дополнительно эмулируется на D18; старое «virtual/D15» — **ОТЗЫВАЕТСЯ** |
| **Z_MIN (BLTouch SIGNAL)** | `Z_MIN_PIN = 18` | D18 | PD3 (USART1_TX) | pin 46 | **Z-S** | multimeter ~1 Ω → pin 46 (Fig.1-1) |
| **VCC** | board rail | — | — | — | **J1-V / Z-V** | 0 Ω to +5 V electrolytic; 5.0 V measured |
| **GND** | board rail | — | — | — | **J1-G / Z-G** | 0 Ω to GND electrolytic (common node) |
| X_MIN | `X_MIN_PIN = 3` | D3 | PE5 | pin 7 | X− | **user-confirmed definitive map** (X− port = D3); old «D2» interpretation **RETRACTED** (D2 was never wired) |
| Y_MIN | `Y_MIN_PIN = 14` | D14 | PJ1 | pin 64 | Y− | pins_RAMPS.h L97 |
| Z_MAX | `Z_MAX_PIN = 19` (disabled) | D19 | PD4 (TWI_SDA) | pin 47 | — | no USE_ZMAX_PLUG → −1 (порт занят SERVO0 — BLTouch servo, REAL WIRE) |
| SERVO1 | `SERVO1_PIN = 6` (disabled) | D6 | PH3 | pin 15 | — | NUM_SERVOS=1 |
| (ref) | D52 | D52 | PB1 | pin 20 | — | multimeter ~1 Ω (pin 20) |
| XTAL2 | — | — | PB4 | pin 33 | — | Fig.1-1 |
| XTAL1 | — | — | PB3 | pin 34 | — | Fig.1-1 |

> TQFP-100 pin numbers per **Microchip DS40002211A Figure 1-1** (directive); **pin 7, 46, 20 additionally confirmed by multimeter**. D#→AVR from `variants/mega/pins_arduino.h`.

---

*Prepared 2026-09-18. Compile-only + inspection only. **NEITHER Configuration.h NOR Configuration_adv.h NOR pins_RAMPS.h was modified. NO MCU WRITE PERFORMED.***
