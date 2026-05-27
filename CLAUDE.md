# picoruby-py32-io-expander

PicoRuby driver gem for the PY32 IO Expander on M5Stack StackChan AI Desktop Robot.

## Cold-boot / wiring notes (M5Stack StackChan AI)

### GPIO 0 — VM_EN (WS2812 5V rail)

Set GPIO 0 (`set_direction(0, true)` + `digital_write(0, true)`) and wait 200 ms **before** any LED operation. This enables the 5V supply rail that powers the WS2812 LEDs via the AW9523 IO expander. Without it the LED chip receives no power and all writes are silently ignored.

### GPIO 13 — WS2812 data line

Configure GPIO 13 as push-pull output (`set_direction(13, true)` + `set_drive_mode(13, false)`) during init. This is the data line the PY32 uses to clock out the WS2812 protocol frames.

### `refresh_leds` is read-modify-write — mandatory

`refresh_leds` reads `REG_LED_CFG` (0x24) first, then ORs in bit 6 (REFRESH_BIT = 0x40) before writing back. This preserves the LED count (bits 5:0) set by `set_led_count`. Writing a plain `0x40` would zero the count and the chip would emit no WS2812 frame (silence, not an error).

Always call `set_led_count(n)` before `refresh_leds` on cold-boot, then use `refresh_leds` to trigger subsequent frames.

### Initialization sequence (PicoRuby application.rb context)

The full cold-boot sequence for StackChan AI requires the AXP2101 PMIC and AW9523 to be initialised first (system I2C SDA=GPIO 12 / SCL=GPIO 11) before this driver is useful:

1. AXP2101 @ 0x34 — enable all LDO/DLDO rails (especially Reg 0x99 for LCD backlight rail)
2. AW9523 @ 0x58 — `Reg 0x02 P0 output = 0b00000111` to enable the WS2812 5V rail; `Reg 0x03 P1` pulse for LCD reset
3. PY32IOExpander @ 0x6F — GPIO 0 VM_EN HIGH + 200 ms settle, then GPIO 13 push-pull output
4. `set_led_count` + `write_led_ram` + `refresh_leds`

The `puts` statements in the PY32IOExpander init region of `application.rb` are required (memory-layout dependent, removing them causes a LoadProhibited crash earlier in boot).

## Host test notes

- `FakeI2C` is defined in `test/test_helper.rb`. No external mock gem is needed.
- `picoruby-i2c` is a PicoRuby built-in; host tests shim it via `$LOADED_FEATURES << "i2c"`.
- Run with `bundle exec rake test`.

## PicoRuby on-device require

On device: `require 'py32-io-expander'` (hyphenated, strip `picoruby-` prefix). Underscore form works only in CRuby host context.
