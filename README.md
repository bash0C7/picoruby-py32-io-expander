# picoruby-py32-io-expander

PY32 IO Expander driver for PicoRuby (M5Stack StackChan AI base).

Drives the PY32 microcontroller used on the M5Stack StackChan AI Desktop Robot as an I2C-connected GPIO and WS2812 LED controller. Provides GPIO direction/pull/drive-mode control and RGB565 LED RAM writing with read-modify-write refresh.

## Installation

Add this line to your PicoRuby build configuration (`build_config/xtensa-esp-picoruby.rb`):

```ruby
conf.gem github: 'bash0C7/picoruby-py32-io-expander'
```

## I2C Address

`0x6F`

## Dependencies

- `picoruby-i2c` (runtime built-in on R2P2-ESP32; shimmed for host tests)

## Public API

`refresh_leds` reads `REG_LED_CFG` (0x24) then ORs in bit 6 (`REFRESH_BIT`, 0x40) before writing back, so the LED count (bits 5:0) set by `set_led_count` is preserved. A plain write of 0x40 zeroes the count and the chip emits no WS2812 frame.

```ruby
py32 = PY32IOExpander.new(i2c)  # i2c: a PicoRuby I2C instance

# LED control
py32.set_led_count(12)                        # configure how many LEDs the chip drives
py32.write_led_ram([[255, 0, 0], [0, 255, 0]]) # write RGB pixel array as RGB565 little-endian
py32.refresh_leds                              # trigger WS2812 frame output (read-modify-write)

# GPIO control
py32.set_direction(pin, output)   # true = output, false = input
py32.set_pull_mode(pin, up)       # true = pull-up, false = pull-down
py32.set_drive_mode(pin, open_drain) # true = open-drain, false = push-pull
py32.digital_write(pin, level)    # set GPIO output level
```

### Usage example (R2P2-ESP32 / StackChan AI)

```ruby
require 'i2c'
require 'py32-io-expander'

i2c = I2C.new(unit: :ESP32_I2C0, sda_pin: 12, scl_pin: 11, frequency: 400_000)
py32 = PY32IOExpander.new(i2c)

# Enable VM power rail via GPIO 0
py32.set_direction(0, true)
py32.digital_write(0, true)
sleep_ms 200

# Configure WS2812 data line (GPIO 13) as push-pull output
py32.set_direction(13, true)
py32.set_drive_mode(13, false)

# Light 12 LEDs red
py32.set_led_count(12)
py32.write_led_ram(Array.new(12) { [255, 0, 0] })
py32.refresh_leds
```

## Host tests

```bash
bundle install
bundle exec rake test
```

## mrbgem.rake

The gem declares `add_dependency 'picoruby-i2c'` and `add_test_dependency 'picoruby-picotest'`. Build with [R2P2-ESP32](https://github.com/picoruby/R2P2-ESP32).

## License

MIT — see [LICENSE](LICENSE).
