---
title: Python on the Arduino Uno
description: Compile Python to native ATmega328P firmware for the Arduino Uno. Pin map, supported peripherals, the real size of a blink, and how this differs from PySerial and MicroPython.
---

The Arduino Uno is PyMCU's reference board. It is the target with the deepest test
coverage and the widest driver set, and every release is validated on real silicon
before it ships.

The chip is an **ATmega328P at 16 MHz**, with 32 KB of flash and 2 KB of SRAM. PyMCU
compiles your Python on your PC and sends the board nothing but machine code. There is
no interpreter, no bytecode and no garbage collector on the chip.

## The short version

```python
from machine import Pin
from time import sleep_ms

led = Pin(13, Pin.OUT)
while True:
    led.value(1)
    sleep_ms(500)
    led.value(0)
    sleep_ms(500)
```

```bash
pymcu new blink --board arduino_uno --stdlib micropython
cd blink
pymcu build
pymcu flash
```

That program builds to **150 bytes of flash and 0 bytes of SRAM**: 46 bytes of your code
plus the 104-byte interrupt vector table the ATmega328P always carries. It uses 0% of the
32 KB available, and nothing lands in `.data` or `.bss`.

Here is what the compiler emits for the loop:

```asm
main:
	SBI	0x04, 5          ; DDRB  bit 5 -> output
L_46:
	SBI	0x05, 5          ; PORTB bit 5 -> high
	CALL	__dly_c1333333
	CBI	0x05, 5          ; PORTB bit 5 -> low
	CALL	__dly_c1333333
	JMP	L_46
```

`Pin(13, Pin.OUT)` became a single `SBI` instruction. The object exists in your source and
nowhere on the chip. That is the whole idea behind
[zero-cost abstractions](/guides/zero-cost-classes/).

## How this differs from the other ways to use Python with an Uno

Three different things get called "Python on the Arduino Uno", and they are not
interchangeable.

| Approach | Where Python runs | Board works standalone? |
|---|---|---|
| **PySerial / pyFirmata** | On your PC | No. The Uno is a peripheral of a running computer |
| **MicroPython / CircuitPython** | On the chip, as an interpreter | Not on an Uno. The runtime does not fit in 32 KB / 2 KB |
| **PyMCU** | Nowhere. It is compiled away | Yes. The board runs native firmware |

If you want to plot sensor data on a laptop, PySerial is the right tool and always was.
If you want the board to run on its own, on battery, with deterministic timing, you have
historically had to write C++. PyMCU is the third option: you write Python, the board
receives AVR instructions.

The 32 KB / 2 KB budget is the reason MicroPython and CircuitPython do not offer an Uno
build. PyMCU sidesteps the constraint rather than fitting into it, because nothing of the
language survives to runtime. You can still write in the
[MicroPython](/compat/micropython/) or [CircuitPython](/compat/circuitpython/) API, which
is what the example above does, and the familiar `machine.Pin` call compiles to `SBI`.

## Pin map

Import the names from `pymcu.boards.arduino_uno` and you never have to remember which
port a digital pin lives on:

```python
from pymcu.boards.arduino_uno import D2, D13, A0, LED_BUILTIN
from pymcu.hal.gpio import Pin

led = Pin(LED_BUILTIN, Pin.OUT)
```

These constants are resolved at compile time, so using them costs nothing.

### Digital pins

| Board pin | Chip pin | Also |
|---|---|---|
| `D0` | `PD0` | UART RX |
| `D1` | `PD1` | UART TX |
| `D2` | `PD2` | INT0 |
| `D3` | `PD3` | INT1, PWM OC2B |
| `D4` | `PD4` | |
| `D5` | `PD5` | PWM OC0B |
| `D6` | `PD6` | PWM OC0A |
| `D7` | `PD7` | |
| `D8` | `PB0` | |
| `D9` | `PB1` | PWM OC1A |
| `D10` | `PB2` | SPI SS, PWM OC1B |
| `D11` | `PB3` | SPI MOSI, PWM OC2A |
| `D12` | `PB4` | SPI MISO |
| `D13` | `PB5` | SPI SCK, built-in LED (`LED_BUILTIN`) |

### Analog pins

All six work as plain GPIO too.

| Board pin | Chip pin | Also |
|---|---|---|
| `A0` | `PC0` | |
| `A1` | `PC1` | |
| `A2` | `PC2` | |
| `A3` | `PC3` | |
| `A4` | `PC4` | I2C SDA |
| `A5` | `PC5` | I2C SCL |

The ADC also reaches the internal temperature sensor (`"TEMP"`), the bandgap reference
(`"VBG"`) and `"ADC8"`. See [ADC](/stdlib/adc/).

:::note
`pymcu.boards.arduino_uno` refuses to compile for a non-AVR chip. The guard is evaluated
before IR generation, so a wrong `--board` fails the build instead of producing firmware
that silently writes to the wrong registers.
:::

## Peripherals

Every peripheral on the ATmega328P is available, through the HAL or through the
MicroPython and CircuitPython APIs, which compile to the same firmware.

| Peripheral | Module | Notes |
|---|---|---|
| GPIO | [`gpio`](/stdlib/gpio/) | high / low / toggle / irq / `pulse_in` |
| UART | [`uart`](/stdlib/uart/) | write / read / println, RX interrupt |
| ADC | [`adc`](/stdlib/adc/) | polling and interrupt driven |
| Timers | [`timer`](/stdlib/timer/) | CTC mode, `millis()` and `micros()` |
| PWM | [`pwm`](/stdlib/pwm/) | multi-channel, `set_duty` / `set_freq` |
| SPI | [`spi`](/stdlib/spi/) | hardware, plus bit-banged `SoftSPI` on any pins |
| I2C | [`i2c`](/stdlib/i2c/) | hardware TWI, plus bit-banged `SoftI2C` |
| EEPROM | [`eeprom`](/stdlib/eeprom/) | the 1 KB of on-chip EEPROM |
| Watchdog | [`watchdog`](/stdlib/watchdog/) | enable / disable / feed |
| Sleep modes | [`power`](/stdlib/power/) | all six ATmega328P modes |
| Interrupts | [`interrupts`](/stdlib/interrupts/) | ISRs written in Python |

Seven device drivers ship in the standard library and all of them run on the Uno:
[DHT11](/stdlib/drivers/dht11/), [DS18B20](/stdlib/drivers/ds18b20/),
[HD44780 LCD](/stdlib/drivers/lcd/), [SSD1306 OLED](/stdlib/drivers/ssd1306/),
[MAX7219](/stdlib/drivers/max7219/), [BMP280](/stdlib/drivers/bmp280/) and
[WS2812 NeoPixel](/stdlib/drivers/neopixel/).

## Language features on this target

AVR is the most complete backend. Everything in the
[language reference](/language-reference/) works here, including the pieces that took
longest to reach the other targets:

- `try` / `except` / `raise` / `finally`, propagating across function calls
- `async` / `await`, with the time base taken from Timer0
- f-strings, `dict`, `set`, tuples and generators
- IEEE-754 floats, `print(float)` included
- Inline `asm()`, and `const` tables placed in flash rather than RAM

The [Limitations](/limitations/) page describes the exact shape of the accepted Python
subset. The short version is that anything requiring runtime dynamism is rejected at
compile time rather than failing on the board.

## Calling Arduino libraries

`@extern` declares a symbol implemented in C or C++, and `[tool.pymcu.ffi]` in
`pyproject.toml` lists the sources to compile and link alongside your firmware. Existing
Arduino C and C++ code can be called from PyMCU without rewriting it. See
[C interop](/guides/c-interop/) and the [CLI driver](/driver/) reference.

## Building and flashing

```bash
pymcu build     # produces dist/firmware.hex
pymcu flash     # uploads over the Arduino bootloader with avrdude
```

The AVR toolchain, assembler, linker and C/C++ front ends run in-process as WebAssembly
modules. There is no `avr-gcc` to install on your machine and the wheel is
architecture-independent, so the same firmware comes out on macOS, Linux and Windows.

`pymcu build` also writes `dist/firmware.gas.asm`. Reading it is the fastest way to
confirm what your Python actually became, as in the blink above.

## Try it without hardware

The [playground](https://playground.pymcu.org) compiles and simulates an Arduino Uno in
the browser against a register-accurate emulator. No install, no board.

## See also

- [Quick Start](/getting-started/quickstart/), the same blink walked through step by step
- [Supported Targets](/targets/), the other chips and backends
- [Port a MicroPython project](/migration/from-micropython/)
- [Zero-cost classes](/guides/zero-cost-classes/), why `Pin` costs nothing
