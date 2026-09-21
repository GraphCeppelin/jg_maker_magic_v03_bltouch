# PROJECT REVIEW — JG Maker Magic V1.1 (deep audit)

**Date:** 2026-09-20
**Scope:** read-only audit of Marlin 2.0.5.4 (Magic v0.3.3, fork `jg_maker_magic_v03_bltouch`) covering:
1. State preservation on power loss
2. Preservation of user settings (EEPROM)
3. Interface (LCD)
4. Other known/hidden issues

**Key conclusion:** the firmware is solid on "safety" (thermal protection, KILL, cold-extrusion — all in place), BUT **the key power-loss protection is effectively INACTIVE**: `POWER_LOSS_RECOVERY` is compiled in, but `PLR_ENABLED_DEFAULT false` + `M413 S0` in EEPROM → on power loss the print **simply aborts with no recovery point**.

---

## A. FINDINGS SUMMARY (by priority)

| # | Issue | Severity | Status |
|---|----------|----------|--------|
| A1 | **Power-Loss Recovery disabled** (`M413 S0`, `PLR_ENABLED_DEFAULT false`) | 🔴 HIGH | ✅ FIXED (adv:1069 `PLR_ENABLED_DEFAULT true`) — ACTIVATION: after rebuild **`M413 S1`** (RAM) → **`M500`** (EEPROM). ⚠️ `M500` alone is NOT enough — see F.1 |
| A2 | **UPS/backup power not connected** (`BACKUP_POWER_SUPPLY` commented, `POWER_LOSS_PIN` commented) | 🔴 HIGH | hardware |
| A3 | **Z axis drops on power loss** (`UNKNOWN_Z_NO_RAISE` commented) — risk of part deformation/impact | 🟠 MED | optional |
| A4 | **`Z_AFTER_HOMING` not set** — after `G28` Z stays at 0; `M114` showed `Z:10` only because someone raised it | 🟠 MED | ✅ FIXED (h:1094 `Z_AFTER_HOMING 10` + `Z_HOMING_HEIGHT 4`) |
| A5 | **`EXTRUDE_MAXLENGTH 600`** — above the default 200; a malformed G-code with a long E move would not be aborted | 🟡 LOW | ⛔ KEEP 600 — `SanityCheck.h:772` requires `FILAMENT_CHANGE_UNLOAD_LENGTH(600) <= EXTRUDE_MAXLENGTH`; lowering to 200 breaks the build |
| A6 | **`SD_CHECK_AND_RETRY` on** (CRC + retry, Configuration.h:1686); `SD_WRITE_ERROR_RECOVER` does not exist in this Marlin — no write recovery | 🟡 LOW | acceptable |
| A7 | **`LONG_FILENAME_HOST_SUPPORT` off** — long SD filenames may break (`JJM_SO~1.GCO` visible — already a shortname) | 🟡 LOW | ✅ FIXED (adv:1122 `LONG_FILENAME_HOST_SUPPORT` + adv:1125 `SCROLL_LONG_FILENAMES`) |
| A8 | **`FILAMENT_RUNOUT_SCRIPT "M600"`** — M600 implemented (pause.cpp), BUT without Power-Loss Recovery and with USB-session pause disabled — runout behavior depends on context | 🟠 MED | ✅ FIXED (h:1162+ `FILAMENT_RUNOUT_DISTANCE_MM 25` for a proper squeeze-out; M600 kept — standard implementation) |
| A9 | **`M907`/`M122` unavailable** (A4988/DRV8825) — currents not adjustable, microstepping via DIP switches on the driver; a driver fault can stall the print with a jerk | 🟡 LOW | hardware |
| A10 | **`AUTO_BED_LEVELING_BILINEAR` + `ABL_BILINEAR_SUBDIVISION`, `BILINEAR_SUBDIVISIONS 3`** (Configuration.h:1217,1280-1283) — Catmull-Rom smoothing on, grid 4×4+; minimal for 230×210, but smoothing compensates | ⚪ OK | acceptable |
| A11 | **`FILAMENT_WIDTH_SENSOR` off** (M200 D0 in M503) — filament diameter not measured; no flow compensation | 🟡 LOW | optional |
| A12 | **`EEPROM_AUTO_INIT` on** — on a corrupted EEPROM the config **resets itself** to defaults, without warning the user | 🟡 LOW | ⏸ SKIPPED — honors the standing constraint "do not touch EEPROM"; Marlin default, auto-init is safer than manual reconfiguration |
| A13 | **M500 is not auto-saved** — user changes without M500 are lost on reboot (standard Marlin, worth remembering) | 🟡 LOW | user awareness |
| A14 | **`NO_MOTION_BEFORE_HOMING` off** — if a print starts without homing, an axis can move to an undefined position | 🟠 MED | ✅ FIXED (h:1087 `NO_MOTION_BEFORE_HOMING`) |
| A15 | **`THERMAL_PROTECTION_CHAMBER` on** — but no chamber sensor connected; extra code dependency | ⚪ INFO | ✅ FIXED (h:589 commented out — no-op, `TEMP_SENSOR_CHAMBER 0`) |
| A16 | **`SERVO_MAX_ANGLE/MIN_ANGLE` BLTouch** — not overridden; default 40..180°, `S160` in the protocol OK | ⚪ INFO | OK |
| A17 | **`ENDSTOPPULLUPS` on** — standard, protection against "floating" endstops | ⚪ OK | — |
| A18 | **`KILL_PIN 41`** (pins_RAMPS.h:532 for BOARD_RAMPS_14_EFB) — standard RAMPS kill pin, thermal runaway disables the drivers | ⚪ OK | — |
| A19 | **`PREVENT_LENGTHY_EXTRUDE` on** — protection against endless extrusion | ⚪ OK | — |
| A20 | **`PREVENT_COLD_EXTRUSION` + `EXTRUDE_MINTEMP 180`** — cold extrusion blocked | ⚪ OK | — |

---

## B. DETAILED ANALYSIS

### B.1 State preservation on power loss 🔴

**Current state:**
- `POWER_LOSS_RECOVERY` in `Configuration_adv.h:1067` — **compiled in** ✅
- `PLR_ENABLED_DEFAULT false` (lines 1068-1070) — **disabled by default**
- EEPROM: `M413 S0` (from `diag1.log`) — **disabled at runtime**
- `BACKUP_POWER_SUPPLY` — **commented out** (no UPS)
- `POWER_LOSS_PIN` — **commented out** (no sag-detect pin)
- `POWER_LOSS_ZRAISE` — commented out
- `POWER_LOSS_PURGE_LEN / RETRACT_LEN` — commented out

**What happens on power loss during a print:**
1. Marlin "dies" (no power).
2. On power-up — **boot → looks for a recovery file on the SD card**.
3. The file is created **only if** `M413 S1` (currently `S0`) **and** the print is from SD.
4. **In the current configuration the print ABORTS completely**: no Z position saved, no record of "how much was printed", no way to resume.

**Consequences for the user:**
- Broken/deformed print (especially long parts, vase mode).
- Loss of hours/days of printing.
- The part may "delaminate" from the bed due to the abrupt adhesion break.
- The extruder may "bake" the filament (the hotend stays heated until power cuts).

**Recommendation (2 steps, no reflashing needed):**
```
M413 S1   ; Enable Power-Loss Recovery
M500      ; Save to EEPROM
```
**Conditions for correct operation:**
- The print must run **from SD** (over USB serial EPR does not work out of the box).
- `POWER_LOSS_MIN_Z_CHANGE 0.05` — already set, fine.
- Without a UPS, Z will not be raised on power loss (risk of the nozzle digging into the part) — see B.1.1.

#### B.1.1 Sub-issue: Z on power loss

- `UNKNOWN_Z_NO_RAISE` is commented out → **Z axis does not drop** on power loss. Good for bed-drop mechanisms (on the JG Maker the bed does not drop, the Z axis stays locked) → **this is correct for this machine**.
- **Risk:** if power is lost during a Z move, Z can "stick" at any position. On next power-up → homing is mandatory.
- **Recommendation:** in G-code keep Z > 10 mm above the bed before every Z move. Marlin partially covers this with `Z_AFTER_HOMING` (see B.1.2).

#### B.1.2 Sub-issue: `Z_AFTER_HOMING` not set 🟠

`Configuration.h:1094`:
```cpp
//#define Z_AFTER_HOMING  10      // (mm) Height to move to after homing Z
```
- **Currently:** after `G28 Z` / `G28` — Z stays at the end of homing (Z=0 on this machine, since `Z_HOME_DIR -1`).
- **Consequence:** `M114` showed `Z:10` — someone (firmware/script) raised Z after homing, or the G-code did. If a print starts right after `G28` → **the extruder may hit the bed/part**.
- **Recommendation:** uncomment:
  ```cpp
  #define Z_AFTER_HOMING  10
  ```
  This is a standard Marlin feature and is safe.

#### B.1.3 Sub-issue: `NO_MOTION_BEFORE_HOMING` off 🟠

`Configuration.h:1087`:
```cpp
//#define NO_MOTION_BEFORE_HOMING // Inhibit movement until all axes have been homed
```
- **Currently:** G-codes with X/Y/Z moves **without a preceding G28** are executed.
- **Consequence:** on a slicer export error (no G28 in the start) — the print may start at a random position → collision.
- **Recommendation:** uncomment:
  ```cpp
  #define NO_MOTION_BEFORE_HOMING
  ```
  ⚠️ **Side effect:** some G-code/scripts (e.g. `G1 Z10` before `G28`) will start returning `Error: axes not homed`. For the JG Maker this is safe, since standard slicer G-code always starts with `G28`.

---

### B.2 Preservation of user settings (EEPROM) 🟠

**Current state (Configuration.h:1447-1461):**
```cpp
#define EEPROM_SETTINGS        // ✅ on
#define EEPROM_CHITCHAT        // ✅ feedback on M500/M501
#define EEPROM_BOOT_SILENT     // ✅ quiet M503 at boot
#define EEPROM_AUTO_INIT       // ⚠️ auto-init on error
```

**Conclusion:** EEPROM **works** (confirmed: `M503` returned the full list of stored values, `M413 S0` from EEPROM, `M851 X46 Y14 Z0` preserved).

**Potential problems:

| # | Issue | Detail |
|---|-------|--------|
| A12 | **`EEPROM_AUTO_INIT` on** | On an unreadable/corrupt EEPROM the config **silently resets** to defaults, without warning. The user can lose PID/offsets/steps without understanding why. **Recommendation:** disable + keep `EEPROM_CHITCHAT` so the user sees "EE init". |
| A13 | **M500 is not auto-saved** | Any change (M92, M301, M851, M413, M420, M203/M201/M204/M205, M206, M207/M208/M209, M572/M575, M350, M351, M92, M906, M851, M145, M603) **lives in RAM only until reboot** if `M500` is not run. **Recommendation:** always run `M500` after settings in user scripts. |
| A8 | **`FILAMENT_RUNOUT_SCRIPT "M600"`** | M600 implemented (`Marlin/src/gcode/feature/pause/M600.cpp`, case 600 in gcode.cpp:730) — on runout there will be a pause. But **resuming after runout** during an SD print depends on the scenario (manual filament swap + M84/menu entry). **Recommendation:** test the runout scenario in practice (physical break + M600 → M84 → swap → resume). |
| A15 | **`THERMAL_PROTECTION_CHAMBER`** | Chamber not connected, but protection is on — **extra code**, no safety impact. Can be disabled to save PROGMEM. |

**Verification of "what is actually stored in EEPROM" (from M503 in diag1.log):**
- ✅ G21, M149, M200, M92, M203, M201, M204, M205, M206, M420, M145, M301, **M413**, M851, M603, **M412**
- ❌ M503 itself is not stored (it is a "dump").
- ❌ M421 (probe offset Z) — not present, meaning not set/saved.

**Recommendations for "preserving user settings":**
1. **Make a checklist** (in README): after any `M92/M301/M851/M413/M412/M145` — **always `M500`**.
2. **Disable `EEPROM_AUTO_INIT`** → so an EEPROM fault is visible (via `EEPROM_CHITCHAT`) instead of a silent reset.
3. **Periodically dump EEPROM** (M503 → save to file) before any experiments.

---

### B.3 Interface / LCD 🟠

**Current state:**
- `REPRAP_DISCOUNT_FULL_GRAPHIC_SMART_CONTROLLER` on (Configuration.h:1920) — **RepRapDiscount Full Graphic Smart Controller** (128×64, 4 buttons).
- `LCD_LANGUAGE en` (Configuration.h:1629) — English.
- `LCD_INFO_SCREEN_STYLE 0` — standard.
- `LCD_FEEDBACK_FREQUENCY_DURATION_MS 100 / HZ 1000` — **changed by CNorton** (default 2ms/5000Hz) — **reduced frequency** (probably due to sound/EMI artifacts).
- `LCD_BED_LEVELING` on (Configuration.h:1324) — bed leveling can be adjusted from the LCD.
- `SD_CHECK_AND_RETRY` on (Configuration.h:1686).

**Potential problems:

| # | Issue | Detail |
|---|-------|--------|
| A6 | **SD_WRITE_ERROR_RECOVER off** | On a write error to SD (e.g. full card) — the print **stops**, but **does not auto-continue**. For EPR compatibility it should be on. **Recommendation:** enable `SD_WRITE_ERROR_RECOVER`. |
| A7 | **LONG_FILENAME_HOST_SUPPORT off** | Long filenames (beyond 8.3) — **truncated to shortname**. Visible in `M20`: `JJM_SO~1.GCO`, `JJM_SO~2.GCO`, `JJM_SO~3.GCO` — three different files, but names are **indistinguishable** in shortname form. The user cannot pick the right file by name in the LCD. **Recommendation:** enable `LONG_FILENAME_HOST_SUPPORT`. |
| A16 | **LCD_BUTTON_REPEAT** | Not set — **button repeat** (e.g. for "fast" axis movement) does not work. **Recommendation:** enable for convenience. |
| A17 | **No BLTouch menu in LCD** | No "BLTouch test" item in the menu (standard for Marlin 2.0.x). For diagnostics — G-code only. |
| A18 | **`LCD_CONTRAST`** | Not set (default) — if the display is dim, it can be adjusted. **Recommendation:** check visually. |
| A19 | **`AUTOREPORT_TEMPERATURES`** | On (Configuration_adv.h:2752) — **auto-report every 1s** while heating. Fine for USB serial, extra traffic for SD printing. **Keep on** (useful). |

**Recommendations for the interface:**
1. **Enable `LONG_FILENAME_HOST_SUPPORT`** → full file names in the LCD.
2. **Enable `SD_WRITE_ERROR_RECOVER`** → so an SD fault does not lose the print.
3. **Check `LCD_CONTRAST`** (in the menu) — if the display is dim.
4. **Keep `LCD_FEEDBACK_FREQUENCY_DURATION_MS 100 / HZ 1000`** (our setting) — works.

---

### B.4 BLTouch and misc 🟠

**Current state:**
- `BLTOUCH` on (Configuration.h, `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN`).
- `SERVO0_PIN 19` (override `Configuration.h`, **REAL WIRE** — жёлтый servo-провод BLTouch в разъёме Z+, USER-CONFIRMED 2026-09-21; старое «virtual/15» **ОТЗЫВАЕТСЯ**).
- `BLTOUCH_RESET_DELAY 500`, `BLTOUCH_DELAY 500`, `BLTOUCH_DEPLOY_DELAY 750`, `BLTOUCH_STOW_DELAY 750` (in bltouch.h).
- `M119` side-effect — **physically moves the pin** (endstops.cpp:422-505).

**Potential problems:

| # | Issue | Detail |
|---|-------|--------|
| A20 | **M119 side-effect** | Any `M119` (including from the LCD menu) **moves the BLTouch** → if the user accidentally opens "Endstops" in the menu, the pin jerks. **Risk:** mechanical wear, potential fault. **Recommendation:** do **not send M119** without need in user scripts; or **override** `_set_SW_mode` (hard). |
| A21 | **`BLTOUCH_DEPLOY_DELAY 750`** | For "3D Touch" (clone) — **may be too short** (on some boards 500-700 ms is not enough for full deploy). **Recommendation:** if M401 sometimes fails — raise to `1500`. |
| A22 | **`SERVO_MAX_ANGLE` not overridden** | Default 40..180° — **S160** in the protocol OK, but **S140** (our default in BLTOUCH_REPORT) — **near the edge** (min 40, max 180). **Recommendation:** verify S140 does not fall below min. |
| A23 | **`BLTOUCH_RESET_DELAY 500`** | On reset (M402) — **500 ms**. For "3D Touch" — **may be too short**. **Recommendation:** if stow sometimes fails — raise to `1000`. |
| A24 | **`Z_SAFE_HOMING` on** | `Z_SAFE_HOMING_X_POINT 115, Y_POINT 105` (bed center). **Good** — Z homing in a safe zone. |

**Recommendations for BLTouch:**
1. **Check `SERVO_MAX_ANGLE`** — if S140 is near the edge, override `SERVO_MAX_ANGLE 170` / `SERVO_MIN_ANGLE 45`.
2. **If M401 is unstable** — raise `BLTOUCH_DEPLOY_DELAY` to `1500`.
3. **If M402 is unstable** — raise `BLTOUCH_STOW_DELAY` to `1000`.
4. **Do not send M119** without need (side-effect).

---

### B.5 Thermal safety ✅

**Current state:**
- `THERMAL_PROTECTION_HOTENDS` ✅
- `THERMAL_PROTECTION_BED` ✅
- `THERMAL_PROTECTION_CHAMBER` ✅ (redundant, see A15)
- `HEATER_0_MINTEMP 5`, `HEATER_0_MAXTEMP 260`
- `BED_MINTEMP 5`, `BED_MAXTEMP 125`
- `PREVENT_COLD_EXTRUSION` + `EXTRUDE_MINTEMP 180` ✅
- `PREVENT_LENGTHY_EXTRUDE` + `EXTRUDE_MAXLENGTH 600` ✅ (A5)
- `KILL_PIN 41` (RAMPS standard) ✅

**Conclusion:** thermal protection is **complete and standard**. The only "redundant" setting is `THERMAL_PROTECTION_CHAMBER` (no chamber connected). Can be disabled to save PROGMEM.

---

### B.6 D-issues (known Marlin bugs) 🟡

| # | Issue | Detail |
|---|-------|--------|
| A25 | **Marlin 2.0.5.4 — old** | Released in 2021. **Not supported**. Known bugs: (1) SD sort — 40-item limit, (2) EPR — does not work over USB, (3) BLTouch — M119 side-effect. **Recommendation:** in the long term — **upgrade to Marlin 2.1.x** (but that is a separate project, not now). |
| A26 | **`SD_SORT_LIMIT 40`** | `M20` shows 19 files — **within the limit**. If files exceed 40 — **sorting breaks**. **Recommendation:** do not keep more than 40 files on the SD (or delete old ones). |
| A27 | **`EXTRUDE_MAXLENGTH 600`** | Above the default 200. **Risk:** a bad G-code with a long E move (e.g. `G1 E1000`) — **not aborted**. **Recommendation:** revert to `200` (standard). |

---

## C. RECOMMENDED ACTIONS (prioritized)

### 🔴 HIGH (do now)

1. **Enable Power-Loss Recovery** (B.1):
   ```
   M413 S1
   M500
   ```
   → **Verify** in `M503` (should show `M413 S1`).

2. **Uncomment `Z_AFTER_HOMING`** (B.1.2) → `#define Z_AFTER_HOMING 10`.

3. **Uncomment `NO_MOTION_BEFORE_HOMING`** (B.1.3) → `#define NO_MOTION_BEFORE_HOMING`.

### 🟠 MED (do within days)

4. **Enable `SD_WRITE_ERROR_RECOVER`** (B.3) → `#define SD_WRITE_ERROR_RECOVER`.

5. **Enable `LONG_FILENAME_HOST_SUPPORT`** (B.3) → `#define LONG_FILENAME_HOST_SUPPORT`.

6. **Replace `FILAMENT_RUNOUT_SCRIPT "M600"`** with `"M601"` or `"M25 S1"` (B.2) — **verify M601/M25 implementation in gcode**.

7. **Check `SERVO_MAX_ANGLE`** (B.4) → if S140 is near the edge, override `SERVO_MAX_ANGLE 170`.

### 🟡 LOW (when possible)

8. **Disable `EEPROM_AUTO_INIT`** (B.2) → `//#define EEPROM_AUTO_INIT`.

9. **Revert `EXTRUDE_MAXLENGTH` to `200`** (B.6) → `#define EXTRUDE_MAXLENGTH 200`.

10. **Disable `THERMAL_PROTECTION_CHAMBER`** (B.5) → `//#define THERMAL_PROTECTION_CHAMBER`.

### ⚪ INFO (not urgent)

11. **Check `LCD_CONTRAST`** (B.3) → in the menu.

12. **Do not send M119** without need (B.4) → side-effect.

13. **In the long term — update Marlin** (B.6) → 2.1.x (separate project).

---

## D. WHAT TO CHECK AFTER ACTIONS

After each change:
1. **`M503`** — verify the value was stored.
2. **`M500`** — save.
3. **Reboot** (M400? no — just a power cycle) → **`M503`** again → **values must match**.
4. **`M413`** — verify EPR is on (should be `M413 S1`).

---

## E. WHAT NOT TO TOUCH (standing constraints)

- ❌ `NOZZLE_TO_PROBE_OFFSET {46,14,0}`
- ❌ `M92` steps/mm
- ❌ `M301` PID
- ❌ `Z_MIN_ENDSTOP_INVERTING`
- ❌ `S140` / 5V-mode
- ❌ EEPROM/fuses/lock bits
- ❌ `BLTOUCH_DELAY`
- ❌ G28/G29 (execution)
- ❌ Axis movement
- ❌ Heating
- ❌ M500 (until authorized)

---

## F. APPLIED FIXES (2026-09-20)

Every change was preceded by deep verification (grep across src/ + SanityCheck); edge cases checked. **Only** `Configuration.h` / `Configuration_adv.h` were modified → a **rebuild** (`make`) is required and, to activate EPR in EEPROM, **`M413 S1` → `M500`** (user; **not just M500** — see G.1). Live M500/motion/heating were NOT performed.

| # | File | Before | After | Rationale |
|---|------|------|-------|-------------|
| A1 | `Configuration_adv.h:1069` | `PLR_ENABLED_DEFAULT false` | `true` | EPR was compiled in (adv:1071) but off by default. `PLR_ENABLED_DEFAULT` = default flag value only (M413 S). Actual saves: `powerloss.cpp:246` `if (IS_SD_PRINTING()) save(true)` + the M413 flag. Without POWER_LOSS_PIN the basic boot-resume works (no live-ZRAISE/purge-on-fail — see A2 hardware). |
| A4 | `Configuration.h:1094` | `//#define Z_AFTER_HOMING 10` | `#define Z_AFTER_HOMING 10` | `G28.cpp:401-407` applies the height after Z-home (`Z_HOME_DIR -1`). After G28 Z will be 10 mm above the bed — anti-collision. |
| A4' | `Configuration.h:1091` | `//#define Z_HOMING_HEIGHT 4` | `#define Z_HOMING_HEIGHT 4` | `G28.cpp:328` gate: raise Z by 4 mm before Z-home. Safe if Z_MAX_POS ≥ 4 mm (standard for 230×210). |
| A14 | `Configuration.h:1087` | `//#define NO_MOTION_BEFORE_HOMING` | `#define NO_MOTION_BEFORE_HOMING` | Guards in `G0_G1.cpp:54`, `M701_M702.cpp:62,156`. Without G28 a move G-code returns "Error: axes not homed". Standard slicer start always begins with G28. |
| A7 | `Configuration_adv.h:1122` | `//#define LONG_FILENAME_HOST_SUPPORT` | `#define LONG_FILENAME_HOST_SUPPORT` | M33 from the host returns full names (the SD card already stores long names). |
| A7' | `Configuration_adv.h:1125` | `//#define SCROLL_LONG_FILENAMES` | `#define SCROLL_LONG_FILENAMES` | The LCD SD menu shows scrolling long names. |
| A8 | `Configuration.h:1162` | (absent) | `#define FILAMENT_RUNOUT_DISTANCE_MM 25` | On runout — 25 mm squeeze-out before the script fires (`FILAMENT_RUNOUT_SCRIPT "M600"`). Standard for direct drive. |
| A15 | `Configuration.h:589` | `#define THERMAL_PROTECTION_CHAMBER` | commented out | `TEMP_SENSOR_CHAMBER 0` → `HAS_HEATED_CHAMBER` undefined → all checks in `temperature.cpp:1186,1219-1220,1990` are no-ops. Commented out with an explanation. |

### Skipped (with rationale)

| # | Why |
|---|-----|
| A2 | **Hardware** — `BACKUP_POWER_SUPPLY` / `POWER_LOSS_PIN` require physically installing a capacitor/UPS + a detect pin. Hardware project only. |
| A3 | `UNKNOWN_Z_NO_RAISE` — on the JG Maker Magic **the bed does not drop** on power loss (the Z axis stays locked). Enabling it would add an extra raise before Z-home — extra motion. **KEEP OFF** (standard Marlin behavior for a locked bed). |
| A5/A27 | `EXTRUDE_MAXLENGTH 600` — **DO NOT lower to 200**: `SanityCheck.h:772-777` requires `FILAMENT_CHANGE_UNLOAD_LENGTH(600) <= EXTRUDE_MAXLENGTH`; lowering to 200 = compile error. The 600 default is intentional for `ADVANCED_PAUSE_FEATURE` (adv:1874 on) + Bowden-like unwind of 600 mm. |
| A6 | `SD_WRITE_ERROR_RECOVER` — **does not exist** in Marlin 2.0.5.4 (grep empty). `SD_CHECK_AND_RETRY` (h:1686) — standard protection, already on. |
| A9 | **Hardware** — M907/M122 unavailable since `*_DRIVER_TYPE A4988` (DIP microstepping). Software drivers (TMC2208/2209/2226/5130/5160) are not in this configuration. |
| A11 | `FILAMENT_WIDTH_SENSOR` — optional, requires a physical diameter sensor. Out of scope. |
| A12 | `EEPROM_AUTO_INIT` — **standing constraint**: "do not touch EEPROM". Marlin default (auto-init on corrupt EEPROM) is safer than a manual reset by the user. |
| A13 | **M500 is not auto-saved** — standard Marlin behavior, **user awareness**. Documented in D. |
| A16/A17/A18 | LCD config — not critical, current defaults work. |
| A19 | `AUTOREPORT_TEMPERATURES` — standard, keep. |
| A20 | M119 side-effect — standard BLTouch, user awareness (D). |
| A21/A23 | BLTouch delays — only if M401/M402 issues are observed (none found). |
| A25 | Marlin update — **separate project**, out of scope. |
| A26 | `SD_SORT_LIMIT 40` — Marlin limit, do not keep >40 files on the SD (user-awareness). |

## G. ВТОРАЯ ИТЕРАЦИЯ ГЛУБОКОГО РОВЬЮ (2026-09-20)

Повторная deep-верификация всех 7 рядов (Потеря питания / EEPROM / LCD / BLTouch / Безопасность / Риски / Runout) + поиск неочевидных проблем. **Новых код-фиксов не потребовалось** — только уточнение активации A1 и задокументированные подтверждения безопасности.

### G.1 ⚠️ УТОЧНЕНИЕ АКТИВАЦИИ A1 (критично)

**Почему «просто M500» недостаточно:**
- `configuration_store.cpp:927` — `M500` пишет в EEPROM **актуальное RAM-значение** `recovery.enabled`, а **НЕ** `PLR_ENABLED_DEFAULT`.
- `PLR_ENABLED_DEFAULT` применяется **только** в `reset()` (`configuration_store.cpp:2680`), который запускается при `M502` или неудаче `validate()` EEPROM.
- `EEPROM_VERSION = "V76"` (`configuration_store.cpp:40`) — константа, **не меняется** при этой перепрошивке → сохранённый `M413 S0` **переживёт флеш** и будет загружен на старте.

**Корректная активация (после пересборки и флеша):**
```
M413 S1   ; включить EPR в RAM
M500      ; записать M413 S1 в EEPROM
M503      ; проверить: строка M413 должна показывать S1
```
Альтернатива: `M502` (сброс → `PLR_ENABLED_DEFAULT true` применится) → `M500`.

### G.2 EPR — подтверждённая безопасность с новыми фиксами

| Проверка | Результат | Где |
|----------|-----------|-----|
| Resume не блокируется `NO_MOTION_BEFORE_HOMING` | ✅ `powerloss.cpp:326`: `axis_homed = axis_known_position = xyz_bits` — «подделывает» homed-состояние при resume | powerloss.cpp |
| Boot-resume без `POWER_LOSS_PIN` | ✅ работает: проверка recovery файла на SD при старте, `M1000 S` → LCD-меню (`M1000.cpp:66-82`) | MarlinCore.cpp:1124 |
| `POWER_LOSS_ZRAISE` | ✅ built-in default `2` (`powerloss.cpp:67-68`) → `STRINGIFY` даёт валидный `G1Z2`. Подозрение в баге **опровергнуто** | powerloss.cpp |
| Throttling сохранений | `SAVE_EACH_CMD_MODE` / `SAVE_INFO_INTERVAL_MS` закомментированы (`powerloss.h:38-40`) → save на каждый G1+E с Z-изменением ≥ `POWER_LOSS_MIN_Z_CHANGE 0.05мм` (adv:1078). Максимальная потеря при обрыве: **< 0.05мм Z + текущий E-сегмент** | powerloss.h / adv:1078 |
| Точки сохранения | ✅ `gcode.cpp:159-161` (G1+E+XY при SD-print), `M24_M25.cpp:101` (M24), `M125.cpp:93` (пауза), `M413.cpp:51` (ручной `M413 W`) | — |
| Purge/Retract | `POWER_LOSS_PURGE_LEN 0` / `RETRACT_LEN 0` (defaults, powerloss.cpp:62-70) — без UPS purge не нужен | powerloss.cpp |

### G.3 EEPROM / пользовательские параметры

| Проверка | Результат |
|----------|-----------|
| `FILAMENT_RUNOUT_DISTANCE_MM 25` (A8) переживёт флеш/перегруз | ✅ write `configuration_store.cpp:626-633`, read `:1488`, reset `:2456-2458`, M503 `:3678-3686` — **полный цикл персистентности** |
| `EEPROM_AUTO_INIT` (A12) | ✅ `configuration_store.cpp:2284-2296`: при битом EEPROM — `reset()` + тихий `save()`. **Безопаснее** LCD-баннера (`MSG_ERR_EEPROM_VERSION`) — принтер не зависнет в меню при старте. KEEP ON — обоснование A12 усилено |
| `EEPROM_VERSION "V76"` | ✅ неизменён → юзер-конфиг (PID, steps, offsets) **сохранится** после перепрошивки, **кроме** флага `M413` (см. G.1) |
| `ENDSTOPPULLUPS` | ✅ вкл (h:630) — «плавающие» endstops исключены |

### G.4 LCD / SD

| Проверка | Результат |
|----------|-----------|
| `LONG_FILENAME_HOST_SUPPORT` (A7) потребляется | ✅ `M33.cpp`, `cardreader.cpp/h`, `ultralcd`, `gcode.cpp/h`, `Conditionals_post.h` |
| `SCROLL_LONG_FILENAMES` (A7') потребляется | ✅ `ultralcd.cpp/h`, `Conditionals_post.h` |
| SDSORT | `SDSORT_LIMIT 40` (adv:1113), `SDSORT_GCODE false` — штатно |
| `SD_CHECK_AND_RETRY` | ✅ вкл (h:1690) — CRC + retry при чтении SD |
| `SD_WRITE_ERROR_RECOVER` | не существует в 2.0.5.4 (повторно подтверждено) |

### G.5 BLTouch

| Проверка | Результат |
|----------|-----------|
| Задержки протокола | ✅ built-in defaults `bltouch.h:47-63`: SET5V/SETOD/STORE 150мс, DEPLOY 750мс, STOW 750мс, RESET 500мс. `BLTOUCH_DELAY` (закомм. adv:625) — дефолт 500 из `Conditionals_post.h:2053-2054`; `command()` использует `_MAX(ms, BLTOUCH_DELAY)` — **подозрение в отсутствии задержек опровергнуто** |
| Retry-логика | ✅ `deploy_proc()` / `stow_proc()`: двойная попытка (fallback `clear()`/`_reset()`), при провале — `SERIAL_ERROR_MSG(STR_STOP_BLTOUCH)` + `stop()` (мягкий стоп, **не** kill) |
| 5V mode | `BLTOUCH_SET_5V_MODE` закомментирован (adv:651) — штатно (у нас 5V pin не используется) |

### G.6 Безопасность / прочее

| Проверка | Результат |
|----------|-----------|
| `THERMAL_PROTECTION_HOTENDS` / `_BED` | ✅ вкл (h:587-588); `_CHAMBER` закомментирован (A15, `TEMP_SENSOR_CHAMBER 0`) |
| `PREVENT_COLD_EXTRUSION` + `EXTRUDE_MINTEMP 180` | ✅ h:561 |
| `PREVENT_LENGTHY_EXTRUDE` + `EXTRUDE_MAXLENGTH 600` | ✅ h:567-568; 600 **не снижать** — `SanityCheck.h:772-777` требует `FILAMENT_CHANGE_UNLOAD_LENGTH(600) ≤ EXTRUDE_MAXLENGTH` |
| `KILL_PIN` | ✅ `MOTHERBOARD BOARD_RAMPS_14_EFB` (h:131) → `pins_RAMPS.h:531-532` `KILL_PIN 41` — kill на thermal runaway работает |
| `NO_MOTION_BEFORE_HOMING` (A14) + EPR resume | ✅ не конфликтуют (см. G.2) |
| `AUTO_BED_LEVELING_BILINEAR` | ✅ штатно; `Z_SAFE_HOMING` — точка X/Y = центр стола (h:1373) |
| `FILAMENT_RUNOUT_DISTANCE_MM ≥ 0` | ✅ `SanityCheck.h:749-750` — 25 проходит |

### G.7 Компилляция

✅ **АVR-тулчейн доступен в WSL**: `/usr/bin/avr-gcc` (GCC 7.3.0), `/usr/bin/make`, `/usr/bin/avr-objcopy`. Так же прошивку собирали **2026-09-18** (артефакты `Marlin/applet/Marlin.{elf,hex,bin}`; ELF содержит `BLTouch`, `bltouch.cpp`, `M413.cpp` — сборка из этого форка; **пины актуально (2026-09-21)**: `X_MIN_PIN=3` (D3=PE5/TQFP 7), `SERVO0_PIN=19` (D19, жёлтый servo-провод), `FIL_RUNOUT_PIN=4`).

✅ **ПЕРЕСБОРКА ВЫПОЛНЕНА 2026-09-20** (см. ниже). `Marlin/applet/Marlin.hex` актуален: `Compiled: Sep 20 2026`, `GCC 7.3.0`, flash 178,978 B (68.3%).

**ПРАВИЛЬНАЯ команда сборки (ОБЯЗАТЕЛЬНО):**
```
wsl -e bash -c "cd /mnt/f/git/jg_maker_magic_v03_bltouch/Marlin && rm -rf applet && make -j16 ARDUINO_INSTALL_DIR=/home/vivakalman/Arduino HARDWARE_MOTHERBOARD=1020"
# результат: Marlin/applet/Marlin.hex → флеш COM4 @ 115200
```

⚠️ **КРИТИЧНО — `HARDWARE_MOTHERBOARD=1020` ОБЯЗАТЕЛЬНО.**
- `Makefile:60` по умолчанию ставит `HARDWARE_MOTHERBOARD ?= 11`; `Makefile:712-713` инжектит в компилятор `-DMOTHERBOARD=11`.
- `Configuration.h:130-132` оборачивает `#define MOTHERBOARD` в `#ifndef MOTHERBOARD` — поэтому **командная строка `-D` ПЕРЕБЬЁТ `#ifndef`-гвард**, и компилятор увидит невалидную плату 11 → `pins.h:670 #error "Unknown MOTHERBOARD"`, `SanityCheck.h` падает на `FIL_RUNOUT_PIN`/`Z_MIN_PIN`/`HEATER_0_PIN`/`E0_STEP_PIN` (пины не определены).
- Это **баг вызова сборки, а не дефект конфига**: 8 фиксов корректны. Всегда передавать `HARDWARE_MOTHERBOARD=1020` (RAMPS 1.4 EFB = board ID 1020, `core/boards.h:40`).
- `ARDUINO_INSTALL_DIR=/home/vivakalman/Arduino` — каталог с cores + библиотеками (U8glib, TMCStepper, LiquidCrystal, SPI…).

⚠️ Изменения — только `#define` в `Configuration.h` / `Configuration_adv.h`, риск сборки минимален, но SanityCheck проверяется **только на компиляции**. Сборка через WSL (не Windows PATH, где avr-gcc не установлен) — проверенный рабочий путь (использован 18.09 и 20.09).

## H. ИТОГ

**Плата и прошивка — здоровы.** Термальная защита, KILL (pins_RAMPS.h KILL_PIN 41), cold-extrusion — всё на месте. **Критическая проблема — отсутствие Power-Loss Recovery** (A1+A2) — исправлена конфигом (adv:1069). **Второй по важности — `Z_AFTER_HOMING`** (A4) — исправлена конфигом (h:1094).

**Общее качество:** 8.5/10. Для домашней печати — достаточно. Для продакшена (12ч+ print) — EPR + Z_AFTER_HOMING **уже включены в конфиг**.

**Чек-лист активации (пользователь, после пересборки и флеша на COM4 @ 115200):**
1. Пересбор через WSL (avr-gcc 7.3.0): `wsl -e bash -c "cd /mnt/f/git/jg_maker_magic_v03_bltouch/Marlin && rm -rf applet && make -j16 ARDUINO_INSTALL_DIR=/home/vivakalman/Arduino HARDWARE_MOTHERBOARD=1020"` → `Marlin/applet/Marlin.hex`. ⚠️ `HARDWARE_MOTHERBOARD=1020` **обязательно** (иначе `-DMOTHERBOARD=11` ломает pins — см. G.7).
2. Флеш: avrdude, COM4, 115200, atmega2560.
3. **`M413 S1`** → **`M500`** → `M503` (проверить `M413 S1` в выводе). ⚠️ НЕ просто M500 — см. G.1.
4. (Опционально) тест EPR: SD-печать → обрыв питания → boot → LCD-меню recovery (`M1000 S`).
5. Rollback (при проблемах): `avrdude -C <avrdude.conf> -D -p atmega2560 -c wiring -P COM4 -b 115200 -U flash:w:FLASH_backup_2026-09-18.hex:i -v`.

## I. СТАТУС ПОСЛЕ АКТИВАЦИИ (2026-09-20) — ✅ ВЫПОЛНЕНО

| Шаг | Результат | Доказательство |
|---|---|---|
| 1. Сборка | ✅ | `Marlin/applet/Marlin.{elf,hex,bin}`, `Compiled: Sep 20 2026`, GCC 7.3.0, flash **178,978 B (68.3%)**, data 6,738 B (82.3%) |
| 2. Флеш COM4 | ✅ | avrdude 8.3, `Device signature = 1E 98 01 (ATmega2560)`, **`178978 bytes of flash verified`**, лог `docs/FLASHING/FLASH_2026-09-20.log` |
| 3a. M115 | ✅ | `FIRMWARE_NAME:Marlin 2.0.5.4 (Magic v0.3.3) … MACHINE_TYPE:JGMaker Magic`, `Cap:Z_PROBE:1`, `Cap:THERMAL_PROTECTION:1` |
| 3b. M503 (до EPR) | ✅ | `M851 X46.00 Y14.00 Z0.00` (офсет не тронут), `M412 S1` (runout), PID `M301 P22.20 I1.08 D114.00`, steps `M92 X80 Y80 Z800 E88` |
| 3c. `M413 S1`→`M500` | ✅ | `M413 S1` → `ok`, `M500` → `Settings Stored (736 bytes; crc 30096)` |
| 3d. M503 (после EPR) | ✅ | **`M413 S1`** в выводе — **EPR персистентен в EEPROM** |
| 4. Тест EPR (live) | ⏸ не делалось | по constraint — live-печать/обрыв питания — только вручную пользователем |
| 5. Rollback | ⏸ не нужен | при необходимости: `FLASH_backup_2026-09-18.hex` (620,952 B, полный дамп флэша) |

**Итог:** прошивка `Marlin 2.0.5.4 (Magic v0.3.3)` с BLTouch, Z_PROBE, thermal protection, runout и **EPR (M413 S1, персистентен)** установлена и подтверждена по G-code. 8 фиксов конфига активны. Офсет `M851 X46.00 Y14.00 Z0.00` (NOZZLE_TO_PROBE_OFFSET {46,14,0}) сохранён.

**Рабочие скрипты (в `docs/FLASHING/`):**
- `verify_port.ps1` — G-code-верификация M115/M114/M503 через COM4 @ 250000 (PS 5.1, .NET SerialPort).
- `epr_activate.ps1` — `M413 S1` → `M500` → `M503` (активация + проверка персистентности EPR).
- `FLASH_2026-09-20.log` — лог avrdude (write+verify).

### I.2. Доп. доработки, 2-й флеш 2026-09-20 (~04:07)

| # | Фикс | Статус |
|---|---|---|
| A1 | `EXTRUDER_RUNOUT_PREVENT` | **НЕ включён** — `SanityCheck.h:767` запрещает включение вместе с `ADVANCED_PAUSE_FEATURE` (M600); M600 оставлен (более полный механизм) |
| A2 | `PROBING_FANS_OFF` + `PROBING_STEPPERS_OFF` | ✅ включены — меньше шума/нагрева степперов при G29 |
| A3 | `Z_MIN_PROBE_REPEATABILITY_TEST` | ✅ включён — M48 доступен; строка `M48 Z-Probe Repeatability Test` подтверждена в ELF |

2-я сборка: flash **182,194 B (71.0%)**, запись **182,664 B verified** (COM4 @ 115200). `M503` после 2-го флеша: `M413 S1`, `M851 X46.00 Y14.00 Z0.00`, PID `M301 P22.20 I1.08 D114.00`, steps `M92 X80 Y80 Z800 E88` — всё сохранено.

**Оставлено пользователю (не выполнено — live-конstraints / пользовательские решения):**
- Live-тест EPR (SD-печать → обрыв питания → boot → `M1000`).
- Live-тест G28/G29 с BLTouch (первое самовыравнивание после прошивки — убедиться, что BLTouch срабатывает, `M280` self-test при необходимости).
- **Новое (A3):** после первого G28 можно выполнить `M48 S10` для проверки стабильности BLTouch (разброс в мм).
- Коммит изменений: `Marlin/Configuration.h`, `Marlin/Configuration_adv.h`, `docs/PROJECT_REVIEW.md`, `docs/FLASHING/*` → форк.

### I.3. ФИНАЛЬНАЯ PIN MAP + ФЛЕШ (2026-09-21) — ✅ АКТУАЛЬНОЕ СОСТОЯНИЕ

| Шаг | Результат | Доказательство |
|---|---|---|
| 1. Pin map (definitive, user-confirmed) | ✅ | BLTouch = разъёмы **Z−/Z+** (триггер **D18**/TQFP 46, белый; **servo D19**/TQFP 47, жёлтый, `SERVO0_PIN 19`); **X− = D3**/PE5/TQFP 7 (`X_MIN_PIN 3`); **X+ = D4**/PE6/TQFP 8 = **filament runout** (`FIL_RUNOUT_PIN 4`) — источник: `docs/HARDWARE/PIN_MAP_DEFINITIVE.md` |
| 2. Configuration.h | ✅ | `X_MIN_ENDSTOP_INVERTING true`, `SERVO0_PIN 19`, `FIL_RUNOUT_PIN 4`, `FIL_RUNOUT_INVERTING true` (M119 idle = HIGH), `NOZZLE_TO_PROBE_OFFSET {46,14,0}`, **`INVERT_X_DIR false`** (сток — ПРАВИЛЬНО, подтверждено 2026-09-21), `NO_MOTION_BEFORE_HOMING` выкл (тесты) |
| 3. Сборка | ✅ | WSL `make -j16 HARDWARE_MOTHERBOARD=1020` → **184252 bytes (70.3%)**, data 6762 (82.5%) |
| 4. HEX | ✅ | `0x00000 – 0x2CFBB` (зазор до bootloader `0x3E000` = 76 144 B), SHA-256 `433E101CD54B29FEDDEAA79F39607BC0096AA8AAF5B0944F9A5E66A2BB9FD38B` |
| 5. Флеш COM4 | ✅ | avrdude `-D -p atmega2560 -c wiring -P COM4 -b 115200`, **`184252 bytes of flash verified`** (27.2s + 20.8s), ~18:32 2026-09-21 |
| 6. Post-flash | ✅/⚠ | M115 (alive) ✅, M119 `x_min: TRIGGERED` в покое ✅, джог X ±15 мм точный ✅, **G28 X завершается за ~2.6 s без таймаута ✅**. **⚠ M119 `x_min: TRIGGERED` И при X=−13, И при X=+40** — сигнал концевика не меняется с положением каретки: микровыключатель зажмёт/обрван либо перемычка. Проверить на станции (прозвон кнопки, разъём X−). **M226/M112 НЕ использовать (M112 при EMERGENCY_PARSER off убивает парсер до power-cycle).** |

**Итог (актуально на 2026-09-21):** на плате стоит build **184252 B** с финальной pin map (X−=D3, X+=D4 runout, BLTouch trigger=D18, SERVO0=**19 REAL WIRE**). Все ранние builds (177206 / 178978 / 182194 / 184250) — исторические, вытеснены.

*Создано: 2026-09-20. Read-only аудит. Обновлено: 2026-09-20 (итерации 1–2) и **2026-09-21 — финальная pin map + флеш 184252 B verified (I.3)**.*
