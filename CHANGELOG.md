### Change log

****

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased — 0.3.5 fork]
### Added
- `docs/BLTOUCH/` — full sensor spec, wiring, pin map, Marlin trace, and M401 instability diagnosis.
- `docs/FLASHING/` — USB flashing, handshake, byte-by-byte verification, 5-stage backup analysis, avrdude instructions.
- `docs/BLTOUCH/images/` — sensor photo, 3D Touch parameter/dimension sheet, and 11 step-by-step Marlin configuration screenshots.
- `.gitignore` for local build artifacts, flash backups, and diagnostic scripts.
### Changed
- `Marlin/Configuration.h`:
  - `X_MIN_PIN` → `2` (Magic V1.1 wires X-S to D2/PE4, not D3 as in standard RAMPS).
  - `SERVO0_PIN` → `3` (J1-S wired to D3/PE5, not Y+).
  - `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN` enabled.
  - `Z_PROBE_SERVO_NR = 0`.

## [0.3.2] - 2020-07-15
### Changed
- Changed acceleration values to be back to JGMaker manufacture defaults

## [0.3.1] - 2020-07-15
### Added
- Marlin firmware version 2.0.5

### Removed
- Marlin firmware version 1.1.9
