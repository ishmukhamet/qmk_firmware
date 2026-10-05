# K11 Max ANSI RGB: short-idle sleep workaround

This branch delays MCU deep sleep for **Keychron K11 Max ANSI RGB with encoder**. It is a workaround for delayed input after a short idle period, **not a fix for the underlying wake-up problem**.

## Scope

- Target: `keychron/k11_max/ansi_encoder/rgb:keychron` (VIA/Keychron Launcher enabled).
- USB VID:PID: `3434:0ab3`.
- Upstream base: [`c5d998442710b38650903937249144a0cd728ebe`](https://github.com/Keychron/qmk_firmware/commit/c5d998442710b38650903937249144a0cd728ebe), branch `2025q3`.
- Firmware reports version **1.1.1**. This does not imply binary equivalence to a factory firmware with the same version number.
- Only `ansi_encoder/rgb/config.h` changes executable behavior. Other K11 Max variants and other keyboards are unchanged.

Do not flash this target onto a K11 Pro, an ISO/JIS variant, or a white-backlight variant.

## Observed problem

On Bluetooth, running on battery with the backlight off, typing after a short idle period could stall for several seconds. The behavior occurred on two Macs. Enabling the backlight, or connecting USB power while retaining Bluetooth transport, prevented the symptom.

The shared wireless code permits MCU low-power entry when the backlight is off and blocks it while USB power is connected. The public `2025q3` code uses `DEFAULT_PROCESS_TIME` (3000 ms by default) for the MCU idle timer on this non-analog board. This is separate from the backlight timeout and the Bluetooth module's longer idle/disconnect timeout.

Related upstream reports, not proof of the same underlying defect:

- [#348: Bluetooth resets with RGB off](https://github.com/Keychron/qmk_firmware/issues/348)
- [#355: dropped first keys on Q3 Max wake-up](https://github.com/Keychron/qmk_firmware/issues/355)
- [#510: shared wireless low-power issues](https://github.com/Keychron/qmk_firmware/issues/510) — some findings concern a different MCU and must not be applied to STM32F401 without validation.

## Change and trade-offs

```c
#define DEFAULT_PROCESS_TIME 600000U
```

This changes the MCU idle threshold from **3 seconds to 10 minutes**. Key activity resets the timer. Existing USB, backlight, indicator, matrix-activity and low-power eligibility guards remain in place; ten minutes is an eligibility threshold, not a promise of an exact sleep time.

- LEDs can remain off; the patch does not change or force the saved backlight setting.
- The MCU remains active longer, so battery life may decrease. The effect has not been measured.
- The original wake-up delay may still occur after longer idle periods.
- The common transport path affects Bluetooth and 2.4 GHz, but only Bluetooth has been tested so far.
- Radio firmware, battery protection and the long Bluetooth idle timeout are not changed relative to the control build.
- This branch does not implement a runtime Launcher setting for this threshold.

## Validation so far

Two builds were made from the same base: an unmodified control and this one-file configuration change.

- Both compiled successfully with xPack GNU Arm Embedded GCC 13.2.1-1.1 (macOS ARM64) and QMK CLI 1.1.5.
- Checked the initialized `lpm_reset_time`, its assignment in `lpm_timer_reset()`, and use by `lpm_task()` in the compiled ELF: 3000 ms for control, 600000 ms for the modified build.
- Checked ELF/BIN payload agreement, USB descriptor, DFU suffix/CRC, image size and reset vectors.
- Both images were flashed to the matching hardware; readback matched each image's payload byte for byte.
- The user reported that the delay remained on the control build and that short-idle behavior appeared normal with this build.

Long-idle wake-up, battery life, 2.4 GHz and broader regression testing are **not yet established**. This is a preliminary single-device result, not a general compatibility claim.

## Build

Use the existing QMK build environment and keep the base and submodule revisions pinned. The tested build used the `lib/chibios`, `lib/chibios-contrib`, `lib/lufa`, and `lib/printf` revisions recorded by the Git tree:

```sh
git submodule update --init --recursive lib/chibios lib/chibios-contrib lib/lufa lib/printf
export QMK_HOME="$PWD"
make keychron/k11_max/ansi_encoder/rgb:keychron -j4 SKIP_GIT=yes
```

The toolchain must include `arm-none-eabi-gcc` and `dfu-suffix`, in addition to the QMK CLI/Python dependencies. `SKIP_GIT=yes` skips QMK's Git housekeeping, not compilation checks; it does not initialize or update dependencies for you.

To compare with the control, build the pinned upstream base in a separate clean checkout using the same tools. Embedded build timestamps can change checksums between builds. Never identify a custom binary solely by its displayed 1.1.1 version.

## Flashing and recovery precautions

See the model's [upstream README](readme.md) for the Cable-mode/USB bootloader procedure and build target. Export your layout/macros first. Holding the physical factory Esc position during USB connection invokes Bootmagic regardless of key remapping; Bootmagic may invalidate EEPROM before flashing. A system-level remap to Esc does not substitute for that physical key.

Flashing a full BIN may overwrite the settings area. After reconnecting without held keys, restore your layout as needed and explicitly turn off the backlight before testing. Test on battery, with USB unplugged: attached USB power masks the low-power condition.

A factory settings reset does **not** restore factory firmware. Reinstallation through Launcher depends on the device being accessible in a mode it supports; Launcher recovery from an unbootable application was not tested. STM32 DFU can be a recovery path with a compatible firmware file, but it is not a guarantee against every failure. Do not change option bytes, disable read protection, or perform a mass erase as part of this workaround.

This modifies the main STM32 firmware, not the separate Bluetooth/radio module. Preserve upstream licenses and provide corresponding source and build information when distributing compiled images. No personal layouts, device dumps or USB identifiers beyond the public VID/PID are included here.
