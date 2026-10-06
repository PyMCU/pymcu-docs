---
title: Roadmap
description: What PyMCU already implements across AVR, ARM and PIC, and what is planned next.
---

This page tracks which language and HAL features are implemented, how far each target has
come, and what is planned next. The current release is **v0.1.0b1**
([release notes](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0b1)).

:::note[Beta and alpha]
The frontend and AVR backend are beta. ARM and PIC remain alpha. Prefer the
[MicroPython](/compat/micropython/) or [CircuitPython](/compat/circuitpython/) APIs where
they cover your use case; drop to the native HAL for direct register access and run
`pymcu lint` before porting a larger project. See [State of the beta](/state-of-the-beta/)
for the measured release claim.
:::

---

## Implemented

### Language

| Feature | Notes |
|---|---|
| `if / elif / else` | Compile-time DCE on `__CHIP__`; compat projects also fold `sys.implementation`, `sys.platform` and `os.uname()` guards |
| `while` + `break` / `continue` | |
| `for i in range(n)` | Runtime or constant bounds; counter width and signedness come from the bounds. Short cheap loops unroll; expensive bodies use a counter. The variable holds the last visited value afterward |
| `for x in array`, list or tuple | Fixed arrays and constant sequences. Strings, pairs and constructed ZCA instances may be elements. Long named sequences become flash tables |
| `for chunk in s.split(sep)` | Compile-time strings and separators, optional compile-time `maxsplit`; `split()` is not a list value |
| `for i, x in enumerate(iterable)` | Constant sequences, runtime ranges, fixed buffers and strings; long strings use a flash counter loop |
| `for x, y in zip(a, b)` | Compile-time unroll over paired lists |
| `reversed(iterable)` | Constant reverse unroll; runtime `reversed(range(...))` for steps 1 and -1 |
| `match / case` | Literal, wildcard, OR, guard, sequence, capture and dotted-name patterns; captures use Python scope rules |
| `def` | Defaults, keywords, compile-time `*args` / `**kwargs`, typed overloads, tuple multi-return, buffer annotations and forward-reference annotations |
| Type inference | Parameters, returns, locals, fields and globals widen or become signed from store evidence; loops warn when an unannotated accumulator may outgrow its seed width |
| Top-level scripts (no `def main():`) | The compiler synthesizes `main` from top-level statements |
| Explicit module-level `main()` | Marks where the body runs, including under `if __name__ == "__main__"`; a second call and unsafe early return are refused |
| `class` | ZCA flattening, constructors, properties, data descriptors, class methods and nested classes. Unsupported descriptor shapes are diagnosed |
| Single-level class inheritance | `super()` and imported dotted bases; fields assigned inside a conditional base constructor remain constructor fields |
| Nested class-typed ZCA fields | Method calls and field reads on a class-typed field (a `machine.Pin` wrapping the native HAL `Pin`) dispatch correctly, including a **value-returning** method (`self.pin.read()`), through facade re-exports and single-level inheritance |
| `class Foo(Enum)` | Zero-cost integer constants; no SRAM |
| `collections.namedtuple` | Two-argument compile-time class factory with fields, `__len__` and `__match_args__` |
| `with obj:` / `with a as x, b as y:` | `__enter__` / `__exit__`; zero-cost for `@inline` methods |
| `assert condition, msg` | Compile-time only; statically false → `CompileError` |
| `global` / `nonlocal` | Cross-function variable access; `nonlocal` in `@inline` |
| Exceptions | AVR + ARM flag propagation, messages, tuple and qualified handlers, bounded `except E as e`, OSError integer arguments, and unhandled UART output with automatic initialization |
| `Optional` / scalar `Union` | Up to four members, compile-time specialization where possible and a one-byte runtime tag for returns, parameters, locals and fields |
| Generators (`yield`) | Function or bound-method state machine, direct iteration and `yield from`; no two-way `yield` expression |
| `async def` / `await` (v2) | Compile-time state machine, no heap; requires `import asyncio`. `await asyncio.sleep()` / `sleep_ms()` anywhere in the body — inside `if` / `elif` / `else`, `while <cond>` and `for i in range(...)` at any nesting, with `break` / `continue`; `return expr` surfaces via `._value`. Executors `asyncio.run(coro)` / `asyncio.gather(a, b)` |
| Closed `dict` / `set` literals | Read-only compile-time lookup tables, including class-body dictionaries and rectangular tables of rows |
| `pymcu.collections.FixedDict` | Mutable fixed-capacity integer dict (open addressing over per-instance fixed arrays — no heap, no GC): `d[k]` / `d[k] = v`, `KeyError` / `ValueError`, `k in d`, `len(d)`, `get(k, default)`, `pop(k)`, `clear()`. Capacity is a compile-time constant |
| Integer arithmetic promotion | `+` / `-` / `*` / `<<` promote to the next wider type (`uint8 255 + 45 == 300`); the annotation is a storage width; `uint8(a + b)` is the fixed-width escape hatch; out-of-range literals and folded constants are a `CompileError` |
| True division `/` vs `//` | `/` yields `float` (warns on integer operands); `//` and `%` are integer floor div / mod; a runtime divide-by-zero raises `ZeroDivisionError` |
| f-strings | Streamed or fixed-buffer values, integer and float format specs, self-interpolation snapshot, compile-time folding of constant parts |
| Compile-time string operations | Indexing, slicing, membership, equality, trim, search, replace, case conversion and split iteration |
| Runtime choice between flash strings | Branches store a 16-bit interned id; printing and literal equality dispatch without copying text to SRAM |
| Functions with more than 5 arguments | Overflow arguments passed via a fixed SRAM spill region |
| `in` / `not in` | Constant sequences and strings, runtime equality chains, `__contains__`, and `range` membership |
| `isinstance(x, T)` | Compile-time ZCA and builtin-shape checks, plus runtime tagged-union narrowing |
| `is` / `is not` | Maps to `==` / `!=` |
| `divmod(a, b)` | Unpacked, named or printed tuple result; runtime zero raises |
| `bitcast(T, v)` | Reinterpret raw bytes as `T`; float ↔ uint32; compile-time folding |
| `hex` / `bin` / `oct` | Flash string for constants, fixed runtime buffer otherwise, including negative values |
| `round(x[, n])` | Half-to-even; compile-time integer/float or runtime float with compile-time `n` |
| Reductions | `sum`, `any`, `all`, `min`, `max`, including direct generator-expression arguments over fixed iterables |
| `str(n)` compile-time | `str(42)` → `"42"` string constant |
| `pow` / `**` / selected `math` | Runtime integer and float powers; `sqrt`, `exp`, `log`, `radians`, `isnan`, `isinf`, `isfinite` linked lazily |
| `bytes` literal `b"\x00\xFF"` | Treated as `uint8[N]`; works in `for`, array init, `len()` |
| `bytearray` / `memoryview` | Fixed mutable buffer and writable windows; once-only runtime-sized `bytearray(n)` uses an AVR static arena |
| `array.array` | Integer typecodes map to the bounded AVR list implementation |
| `input(prompt?, maxlen?)` | `line: bytearray = input("prompt")` — reads a newline-terminated line from UART; auto-injects the UART init preamble |
| `open(name, mode)` | Read-only ROMFS handle over an embedded flash blob: `read`, `readinto`, `readline`, `seek`, `tell`, `close`, `with` |
| `int.from_bytes(b, 'little'/'big')` | Compile-time fold or runtime |
| Raw strings `r"\n"` | No escape processing |
| Extended unpacking `first, *rest = tup` | Compile-time tuples only (PEP 3132) |
| Single-clause list comprehensions | Fixed result length. Nested or multi-clause forms are not supported; one multi-clause shape is a tracked wrong-code issue |
| 2-D fixed grids | CircuitPython-style nested list or bytearray comprehensions flatten to one array with row views |
| Slice **read** `arr[1:3]`, `arr[::2]` | Compile-time constant bounds; the result is a fixed-size array sized at compile time |
| Equal-length slice **assignment** `arr[a:b] = src` | List, `bytes` literal, array and slice sources, including overlapping same-array copies (snapshot semantics), and through `__setitem__` objects (`nvm[0:4] = b'…'`) |
| Slice **iteration** `for x in buf[lo:hi]` | Runtime bounds accepted; rewritten to a `range` loop over the backing array. A runtime `step` is a diagnostic |
| `print()` of a buffer | `print(bytearray)`, `print(arr[a:b])` and `print(obj[a:b])` (via `__getitem__` / `__len__`) emit the CPython repr — `bytearray(b'\xcc\x10\xca\xfe')`; the length must be compile-time |
| Float text | MicroPython-style float32 repr with 6 to 9 significant digits; fixed format specs round half-to-even |
| `lambda x: expr` (no capture) | Inlined as an anonymous `@inline` function |
| Dunder operator overloading | Arithmetic, comparison, bitwise, length, containment, call, get/set item and descriptor protocol |
| `@extern("symbol")` | External C/C++ symbol interop with the AVR ABI (AVR only) |
| `__name__` / `if __name__ == "__main__":` | Compile-time guard; body promoted in main, eliminated in libraries |
| Triple-quoted strings | Multiline string literals; the leading newline after the opening quote is stripped; useful for multiline `asm()` |
| `list[T]` heap-bounded list | AVR-only bounded allocator + GC; inferred literals and appends, parameters and returns, integer `array.array` alias |
| Recursion diagnostics | An illegal recursive call reports the full call cycle; user-facing errors are located `file:line` |

### MCU extensions

| Feature | Notes |
|---|---|
| `uint8 / int8 / uint16 / int16 / uint32 / int32` | Annotation for variables; unannotated outlined `def` params and returns are inferred from call sites |
| `int` (built-in) | Maps to `int16`; no import required |
| `float` | IEEE 754 single precision on AVR and ARM. Unformatted text follows the MicroPython float32 policy; format specs use exact float32 digits and half-to-even rounding |
| `ptr[T]` / `ptr(addr)` | Memory-mapped I/O at module scope, in instance fields and as grouped peripheral registers in class namespaces |
| `const[T]` / `const[uint8[N]]` | Compile-time constants — integer, string and **float** (`Timer(freq=2.5)`); flash-resident arrays via `LPM Z` on AVR and `.rodata` on ARM. A runtime-varying argument is a located `CompileError`, not a silent fold |
| `asm("instr")` | Inline assembly; register constraints `%N` on AVR, textual operand constraints on ARM |
| `delay_ms(n)` / `delay_us(n)` | Busy-wait on AVR / PIC; hardware TIMER on ARM |
| `millis()` / `micros()` | Timer0 overflow; atomic 32-bit read under CLI/SEI. `millis()` carries the Arduino-style fractional correction (an overflow is 1024 µs, not 1000 µs); `micros()` is monotonic across an overflow |
| `@inline` | Zero-cost expansion |
| `@interrupt(vector)` | ISR handler generation with automatic `sei` |
| `@property` / `@name.setter` | Compile-time expansion |
| `@naked` | No compiler prolog/epilog; registers hold raw calling-convention values at entry |
| `@classmethod` | Compile-time class-namespace population; `cls` is not a runtime object |
| `@staticmethod` | Silently ignored — all class methods in PyMCU are effectively static |
| `__CHIP__` | Conditional compilation by chip name / architecture |
| `sys.implementation` / `sys.platform` / `os.uname()` | Compile-time compatibility facts when MicroPython or CircuitPython is selected |
| `__FREQ__` | Compile-time clock frequency in Hz |
| `[tool.pymcu.ffi]` build config | C/C++ interop: `sources`, `include_dirs`, `cflags` (AVR) |
| `CompileError` intrinsic | `raise CompileError("msg")` aborts compilation with a `CompileError:` diagnostic; never generates runtime code; used across the native HAL for unsupported arch/chip guards; cannot be caught by `try / except` |

### HAL (ATmega328P)

| Module | Coverage |
|---|---|
| `pymcu.hal.gpio` | `Pin` — `high` / `low` / `toggle` / `value` / `irq` / `pulse_in` |
| `pymcu.hal.uart` | `UART` — `write` / `read` / `read_line` / `write_str` / `println` / `print_byte` / `available` + RX interrupt |
| `pymcu.hal.adc` | `AnalogPin` — poll + interrupt; channels `"PC0"`–`"PC5"`, plus `"TEMP"` (internal sensor), `"VBG"` and `"ADC8"` |
| `pymcu.hal.timer` | `Timer(n, prescaler)` — Timer0/1/2 unified; CTC mode |
| `pymcu.hal.pwm` | `PWM` — `start` / `stop` / `set_duty` / `set_freq`; multi-channel (two channels of one timer coexist — the COM bits are OR-ed). `set_freq` picks the **nearest** reachable prescaler bucket |
| `pymcu.hal.spi` | `SPI` (bit-banged `SoftSPI` lives in `pymcu.hal.softspi`) |
| `pymcu.hal.i2c` | `I2C`; block and register transfers; internal SDA/SCL pull-ups enabled by default (`pullups=False` opts out). Bit-banged `SoftI2C` lives in `pymcu.hal.softi2c` |
| `pymcu.hal.eeprom` | `EEPROM` — `write(addr, val)` / `read(addr)` |
| `pymcu.hal.watchdog` | `Watchdog` — `enable` / `disable` / `feed` |
| `pymcu.hal.power` | `sleep_idle` / `sleep_adc_noise` / `sleep_power_down` / `sleep_power_save` / `sleep_standby` / `sleep_extended_standby` |

### Drivers

| Module | Device |
|---|---|
| `pymcu.drivers.dht11` | DHT11 temperature + humidity |
| `pymcu.drivers.ds18b20` | DS18B20 1-Wire precision temperature (12-bit) |
| `pymcu.drivers.lcd` | HD44780 character LCD (4-bit parallel), class `LCD` |
| `pymcu.drivers.ssd1306` | SSD1306 OLED (I2C, 128×64) |
| `pymcu.drivers.max7219` | MAX7219 8x8 LED matrix (SPI) |
| `pymcu.drivers.bmp280` | BMP280 barometer (I2C) |
| `pymcu.drivers.neopixel` | WS2812 NeoPixel |

These seven are the whole of `pymcu.drivers`. There is **no** LM35 driver module — an LM35 is
an analog sensor, so read it with [`AnalogPin`](/stdlib/adc/). See
[Device drivers](/stdlib/#device-drivers) for the full table.

### Compatibility layers

| Package | Activation | Coverage |
|---|---|---|
| `pymcu-micropython` | `stdlib = ["micropython"]` | `machine` (Pin, UART, ADC — pin or channel number, PWM with `freq()` / `duty_u16()` getters, SPI, I2C, `SoftI2C`, `Timer(id, period, callback)`, WDT), `utime`, `micropython`; `network.WLAN` + `umqtt` on the Pico 2 W (RP2350) only |
| `pymcu-circuitpython` | `stdlib = ["circuitpython"]` | `board`, `digitalio`, `analogio`, `busio` (SPI + I2C), `pwmio`, `time`, `supervisor`, `alarm`, `microcontroller` (`cpu`, `nvm`, `watchdog`, `reset_reason`); `wifi` + `socketpool` + `adafruit_minimqtt` on the Pico 2 W (RP2350) only |

### Boards

| Module | Pins |
|---|---|
| `pymcu.boards.arduino_uno` | `D0`–`D13`, `A0`–`A5`, `LED_BUILTIN` |
| `pymcu.boards.arduino_mega` | `D0`–`D53`, `A0`–`A15`, `LED_BUILTIN` |
| `pymcu.boards.arduino_leonardo` | `D0`–`D13`, `A0`–`A5`, `LED_BUILTIN` |

### Tooling

| Tool | Notes |
|---|---|
| `pymcu lint` | MicroPython / CircuitPython porting assistant: flags constructs PyMCU cannot compile — unbounded `dict` / `set`, reflection, unbounded `append`, `*args` / `**kwargs`, untyped params, … — with a severity and a suggestion per finding |
| `pymcu new` / `build` / `flash` / `clean` | Project scaffolding, compilation, upload and cleanup — see the [CLI driver](/driver/) |
| `pymcu search` / `install` / `uninstall` / `libraries` | Third-party [libraries](/libraries/), resolved against a curated index that measures rather than asks: an install is refused before the download when the library does not build for your chip |
| `pymcu lint --library` | The publication checks for a library of your own — manifest, ASCII, architecture dispatch and API surface. See [Writing a library](/libraries/authoring/) |
| Build-time capacity checks | An image larger than the chip's flash, or static data beyond its SRAM, fails the build with the part's real numbers instead of surfacing at flash time; the SRAM check reserves 64 bytes for the hardware call stack |
| `pymcu-test` (AVR) | Turnkey pytest fixtures over the avr8sharp emulator |

---

## Target status

| Architecture | Chips |
|---|---|
| **AVR** (ATmega) | ATmega48/88/168/328P, ATmega2560, ATmega32U4 |
| **AVR** (ATtiny) | ATtiny25/45/85, ATtiny24/44/84, ATtiny13/13A, ATtiny2313/4313 |
| **ARM** (Cortex-M0+ / M33) | RP2040 (Pico / Pico W), RP2350 (Pico 2 / Pico 2 W) — the CYW43439 radio is driven on the Pico 2 W only |
| **PIC** (mid-range) | PIC16F84A, PIC16F877A — new in alpha 3 |

### AVR

The reference target, and the most thoroughly tested. Direct assembly codegen (no LLVM),
the full HAL and driver set listed above, soft-float, exceptions, and the only backend with
C/C++ interop.

| Feature | Status |
|---|---|
| Full HAL (GPIO, UART, ADC, Timer, PWM, SPI, I2C, EEPROM, watchdog, power) | ✅ |
| `try / except / raise / finally` | ✅ Zero-cost T-flag propagation |
| `float` (soft-float) | ✅ `__fp_*` assembly helpers |
| `list[T]` + GC, generators, `async` / `await` | ✅ |
| `@extern` C/C++ interop | ✅ GCC's `cc1` / `cc1plus`, Arduino libraries usable; no extra to install |
| `@interrupt`, `Pin.irq()`, GPIOR flag promotion | ✅ |
| Toolchain | ✅ Assembler, linker and C/C++ front ends as WebAssembly — [one wheel for every platform](/getting-started/installation/#the-avr-toolchain-is-webassembly) |

### ARM (RP2040 / RP2350)

The ARM backend lowers PyMCU's architecture-agnostic IR to **LLVM IR** rather than emitting
assembly directly, so LLVM handles register allocation, instruction selection, the AAPCS
calling convention and optimization — `thumbv6m-none-eabi` for the RP2040 (Cortex-M0+) and
the Cortex-M33 target for the RP2350. `pymcu build` produces a flat flash image
(`firmware.bin`). Alpha 3 brings this target to feature parity with AVR.

| Feature | Status |
|---|---|
| GPIO (`pymcu.hal.gpio.Pin`) | ✅ Single-cycle IO (SIO); zero-cost; runtime `Pin(n)` |
| UART (`pymcu.hal.uart.UART`) | ✅ PL011; compile-time baud divisors (assume `clk_peri = 125 MHz` on RP2040, `150 MHz` on RP2350) |
| SPI, I2C, PWM, ADC, DMA | ✅ Full HAL on both RP2040 and RP2350 |
| PIO (`@rp2.asm_pio`) | ✅ Both RP2040 and RP2350; PIO assembler in the compiler, the DSL lowers to state-machine setup |
| CYW43439 WiFi | ✅ **Pico 2 W (RP2350) only** — gSPI bring-up, WLAN join, TCP, MQTT publish, with MicroPython and CircuitPython flavors. Open networks only: `connect()` raises `CompileError` on a non-empty key, because WPA is not implemented yet. There is no CYW43 path on the RP2040 / Pico W |
| `try / except / raise / finally` | ✅ Same flag-propagation model as AVR |
| `float` | ✅ RP2040 via the bootrom fast-float library; RP2350 natively on the M33 FPU; `print(float)` on both |
| Flash-resident const data | ✅ Interned strings and `const[uint8[N]]` in `.rodata`; `const[str]` runtime subscript |
| `delay_ms` / `delay_us` | ✅ Hardware TIMER (1 MHz); accurate on silicon |
| Inline `asm()` with operands | ✅ Textual constraints |
| Dual-core / SIO FIFO | ⏳ Planned — only core 0 runs today |
| `@extern` C/C++ interop | ❌ AVR-only |

### PIC

New in alpha 3, covering the mid-range **PIC16F84A** and **PIC16F877A** with direct assembly
codegen. The toolchain (`pymcu-compiler[pic]`) bundles self-contained gputils / `gpasm`
wheels — no system packages needed.

| Feature | Status |
|---|---|
| GPIO, `delay_ms` / `delay_us` | ✅ |
| Software `*` / `//` / `%` (8- and 16-bit) | ✅ No hardware multiplier needed |
| RAM arrays with a runtime index | ✅ FSR / INDF addressing |
| `ZeroDivisionError` on runtime `//` and `%` | ✅ Catchable |
| EUSART UART | ✅ Baud divisors derived from `__FREQ__` (PIC16F877A) |
| Flash strings and `print()` | ✅ |
| General `try / except / raise` | ❌ AVR + ARM only — use return codes on PIC |
| `@extern` C interop | ❌ AVR-only |
| PIC12 / PIC18 / PIC14E arrays | ⏳ Planned |

---

## Planned / next

| Feature | Notes |
|---|---|
| WPA / WPA2 on the CYW43439 | Today `connect()` joins open networks only; a non-empty key is a `CompileError` |
| CYW43439 WiFi on the RP2040 (Pico W) | The gSPI driver is wired for the RP2350 only so far |
| Dual-core / SIO FIFO on RP2040 + RP2350 | Launch core 1 and expose the inter-core FIFO |
| More PIC families | PIC12 and PIC18 codegen; arrays on PIC14E |
| Publish RISC-V 32-bit codegen | CH32V003/V203 builds in-tree but has no PyPI package or install extra; declared-width narrowing remains open in [PyMCU#222](https://github.com/PyMCU/PyMCU/issues/222) |
| `fixed16` (Q8.8 fixed-point) | Float-like sensor math without soft-float overhead |
| Over-the-air (OTA) updates | Bootloader + `pymcu flash` over UART |
| Broader Cortex-M support | STM32, nRF52 — reusing the same LLVM backend |
| MicroPython / CircuitPython API alignment | Broaden the compat modules and close the remaining API gaps |

---

## Not planned

| Feature | Reason |
|---|---|
| **Unbounded** `dict` / `set` | Growing hash tables require a heap. Closed literals (read-only compile-time lookup tables) and `pymcu.collections.FixedDict` (mutable, fixed capacity, no heap) cover the fixed-footprint cases |
| Garbage collection beyond `list[T]` | A full GC is incompatible with deterministic ISR timing |
| Awaiting another coroutine / future, and `await` as an expression | async/await v2 ships (`await asyncio.sleep()` anywhere in the body, `asyncio.run` / `gather`). Sub-future fields would need ZCA construction outside `__init__`, and splitting the state machine mid-expression; call the coroutine and poll it, or use `asyncio.gather` |
| `yield` inside an `@inline` function, or `yield` as an expression | Regular functions and bound methods lower to state machines and support `yield from`; there is no two-way send protocol |
| `f"..."` inline in arbitrary expression positions | Streaming and fixed-buffer assignment both ship, including float format specs; other expression positions have no lowering, so assign to a name first |
| `complex` / `Decimal` | Not available |
| Closures capturing mutable vars | Captured variables require heap cells; `nonlocal` inside `@inline` is supported |
| Runtime `*args` / `**kwargs` | Compile-time sequences and mappings splice at specialized call sites; runtime containers have no fixed call shape |
| Multiple inheritance | Complexity vs. benefit for the ZCA model |
| Metaclasses | No runtime type system |
| Reflection / runtime `getattr` / `hasattr` | No runtime type information. `getattr(module, "name", default)` with a literal name folds at compile time |
| `eval()` / `exec()` | No interpreter on the MCU |

---

For the full list of what the compiler rejects and why, see [Limitations](/limitations/).
Release-by-release detail lives in the [changelog](/changelog/).
