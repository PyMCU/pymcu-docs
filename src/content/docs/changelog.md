---
title: Changelog
description: "Release notes for every PyMCU alpha: what landed in v0.1.0a5, v0.1.0a4 and the earlier pre-releases, listed by language feature, backend and driver."
---

## v0.1.0a5 — Alpha 5 (2026-08-13)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a5)

A portability-and-honesty release. The reader flow — `pipx install` → `pymcu new` →
`pymcu build` → `pymcu flash` — was rehearsed end to end on Ubuntu (ARM64), Windows 11
(ARM64) and macOS before the tag, with real-hardware flashes on two of them.

### Fixed
- **macOS wheels install again on macOS 14 / 15**: the arm64 wheels inherited the build
  runner's `macosx_26_0_arm64` tag, so most Macs found no candidate at all
- **A `ptr[T]` stored in a zero-cost-class field keeps its element width**, so MMIO through
  such a field emits the right access width
- **Error messages no longer lose their own instructions**: Rich markup ate the
  `[tool.pymcu.flash]` section name out of the text telling you to add it, and install hints
  named a package that does not exist
- **`pymcu new` survives hostile environments**: no console and no `git` are handled, and the
  generated `pyproject.toml` declares `requires-python = ">=3.11"`
- **The flash figure is honest**: `pymcu build` reports code bytes consistently with the
  disassembly (blink: **46 bytes**, 150 written to the chip). The old figure subtracted a
  different preamble than the one it claimed
- `pymcu version` exists as a subcommand, not only as the `--version` flag

### Downloads
- avrdude downloads carry pinned SHA-256 digests for all eight official v8.1 assets, and the
  selection distinguishes Linux ARM64 / ARMv6 / 32-bit and native Windows ARM64
- Downloads locate a CA bundle (`SSL_CERT_FILE`, system paths, `certifi`) instead of failing
  with `CERTIFICATE_VERIFY_FAILED`; verification is never disabled
- Archive extraction rejects path escapes by path semantics, not string prefixes

### Toolchain cache
- The cache under `~/.pymcu/tools` is keyed by payload rather than by interpreter, so every
  Python on a machine shares one copy
- Seeding keeps the two newest versions, and the new
  [`pymcu toolchain clean`](/driver/#pymcu-toolchain-clean) (`--dry-run`, `--all`) reclaims
  the rest on demand

### Scaffolding
- `pymcu new --stdlib micropython` (and `circuitpython`) generates a **top-level script** —
  the shape real `main.py` / `code.py` files have — instead of a `def main():` wrapper
- Dependency pins in generated projects tolerate pre-releases, so a project scaffolded from a
  pipx-installed CLI installs cleanly with plain `pip`

## v0.1.0a4 — Alpha 4 (2026-08-02)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a4)

A quality-and-finish release over alpha 3.

### Fixed
- **`pymcu flash` works end to end on the Raspberry Pi Pico / Pico 2**: builds pack a
  `firmware.uf2` and the driver dispatches the right artifact per target
- **async/await on AVR really waits** (Timer0 microsecond timebase); on PIC / RISC-V /
  ATtiny it is a clear compile error instead of an infinite hang
- Bare `REG = x` register writes that silently compiled to nothing are now a located
  error, and the three stdlib features it had broken (`Timer.set_compare`, `tone`,
  servo) write their registers again
- ARM critical sections: `asm()` clobbers memory; `enable/disable_interrupts()` emit
  real `cpsid`/`cpsie`
- `pymcu new` scaffolds the config `flash` reads and the real per-board clock;
  MAX7219 honours the configured chip-select

### Smaller firmware
- Region and constant-delay outlining: the 60 AVR examples shrink **23%** overall
  (lcd −51%, FFI examples up to −74%)

### Language and tooling
- Tuple return annotations (`-> (uint8, uint16)`) with arity and width validation
- New diagnostics: dict/set comprehensions, reflection, compat modules without their
  flavor, bare register assignment
- `pymcu boards --json`, `pymcu stubs`, `pymcu lint --json` for IDE integrations
- `py.typed` ships in the stdlib; `ptr[T]` documents its compiled semantics in-source

## v0.1.0a3 — Alpha 3 (2026-07-31)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a3)

### Language
- **Generators**: `yield` compiles to a coroutine state machine; `for x in gen(...)`
  with Python-exact `break` / `continue` — no heap, no asyncio required
- **async/await v2**: `await` inside `if` / `while` / `for range` with
  `break` / `continue`; coroutine return values; `asyncio.run` / `asyncio.gather`
- **dict/set literals** as closed compile-time lookup tables (`d[k]` with catchable
  `KeyError`, `x in d`, `len(d)`), plus `pymcu.collections.FixedDict` — a mutable
  fixed-capacity dict with zero heap
- **f-strings as values**: `s = f"{x:02x}"` materializes into a fixed buffer
- **Type inference** for unannotated `def` parameters and returns
- Equal-length slice assignment `a[i:j] = b[k:l]` (overlap-safe)
- Recursion diagnostics name the full call cycle; located `file:line` compile errors

### ARM parity (RP2040 / RP2350)
- **Exceptions** on ARM via a portable T-flag model (previously AVR-only)
- **float (f32)**: RP2040 through the bootrom fast-float library; RP2350 natively
  on the M33 FPU; `print(float)` on both
- **CYW43439 WiFi** on the **Pico 2 W (RP2350) only**: join, TCP, MQTT — with MicroPython and
  CircuitPython compat flavors. Open networks only; WPA is not implemented, and a non-empty
  key is rejected at compile time
- `@rp2.asm_pio` PIO DSL, inline `asm()` with operands, IRQ-safe SIO divider,
  flash-resident const tables, runtime `Pin(n)`, module-level statements

### New backend: PIC
- First release of `pymcu-pic`: PIC16F84A and PIC16F877A — software mul/div/mod,
  RAM arrays, catchable `ZeroDivisionError`, EUSART UART, flash strings, `print()`
- `pymcu-pic-toolchain`: self-contained gputils (gpasm) wheels for all platforms

### Tooling
- `pymcu lint` — MicroPython/CircuitPython porting assistant
- `pymcu-test` (AVR) — turnkey pytest fixtures over the avr8sharp emulator

## v0.1.0a2 — Alpha 2 (2026-06-19)

Integer promotion, true division, real `None`, runtime f-string interpolations,
full `try` / `except` / `else` / `finally`, RFC 0001 zero-cost classes with runtime
state, and 40+ features more — see the
[GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a2).

---

## Pre-release history

The entries below are the internal milestones that ran before PyMCU was published to PyPI
under the `0.1.0aN` scheme. They are kept for provenance; where a detail was later revised
(the `None` literal became a real null rather than folding to `-1`, and the documentation
site moved from MkDocs to Astro + Starlight), the current behaviour is the one described in
the [Language Reference](/language-reference/) and [Limitations](/limitations/).

### v0.2

#### Language
- `for i in range(n)` loop with runtime or compile-time bound
- `for x in array` iteration over fixed-size arrays
- `for i, x in enumerate(iterable)` with compile-time index counter
- `match / case` OR patterns (`case 1 | 2:`)
- Single-quoted string literals
- `import X as Y` alias
- `//` floor division operator
- Fixed-size arrays `arr: uint8[N]`, constant-index and variable-index access
- Tuple literals and tuple unpacking `a, b = func()`
- Multi-return functions `def f() -> (uint8, uint8): return (q, r)`
- `@property` / `@name.setter` decorators
- Single-level zero-cost abstraction (ZCA) class inheritance
- `None` literal (folds to `Constant{-1}`)

#### Compiler
- Variable→Constant propagation in optimizer (prevents peephole corruption of inline results)
- Fixed inline parameter scope shadowing in `resolve_binding`
- Inline multi-return result variables use 1-dot names (register-allocatable)

#### Standard Library
- `Pin.pulse_in(state, timeout_us)` for pulse measurement
- `UART.print_byte(value)` for decimal uint8 output
- `DHT11` driver (`pymcu.drivers.dht11`)
- `arduino_uno` board pin definitions (`pymcu.boards.arduino_uno`)

#### Documentation
- `docs/LANGUAGE_REFERENCE.md` — complete language and stdlib reference
- `docs-site/` — MkDocs + Material documentation site
- Updated `LANGUAGE_ROADMAP.md` with T1/T2/T3 backfill plan

### v0.1 — Initial Release

- AVR (ATmega328P) backend
- PIC14/14E/18 backend
- Core language: `if/elif/else`, `while`, `match/case`, `def`, `class`, `return`
- GPIO, UART, ADC, Timer, PWM, SPI, I2C HAL modules
- `@inline`, `@interrupt` decorators
- `ptr[T]` and `const[T]` type system
- `delay_ms` / `delay_us` busy-wait delays
- 31 example projects
- 154 integration tests (AVR8Sharp simulator)
