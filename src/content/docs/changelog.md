---
title: Changelog
description: "Release notes for every PyMCU alpha: what landed in v0.1.0a10, v0.1.0a9, v0.1.0a8, v0.1.0a7 and the earlier pre-releases, listed by language feature, backend and driver."
---

## v0.1.0a10 — Alpha 10 (2026-08-18)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a10)

**The hardware-validation release.** Everything here came out of a sustained bug hunt on a
real Arduino Uno with a logic analyzer, plus a sweep of the official MicroPython quick
reference and the CircuitPython Essentials examples — 63 projects, of which 53 compile and
the rest fail on purpose with a clear diagnostic. Suites at release: 517 unit, 508 driver and
1549 AVR integration tests.

### Fixed — the silent-miscompile class
These are the bugs that produced wrong numbers rather than a build error, so only silicon
could find them.

- Copy propagation no longer forwards through same-width float ↔ int casts: `uint32(float_var)`
  handed back the raw float bits (16464 for 3.25 on a real Uno)
- An unannotated module global widens to its call-result type instead of wrapping at a `uint8`
  store — `f0 = pwm.freq()` printed 232 for 1000
- A user global no longer shadows a same-named function parameter, neither of an `@inline`
  (`data = 5` broke `uart.write('hello')`) nor of a plain `def` (`start_low_ms = 250` drove
  the DHT driver's start pulse for 250 ms instead of 18)
- `raise CompileError` inside an `@inline` body aborts the build even when the call site sits
  under runtime control flow — a swallowed raise had let `readline()` compile to an unbound
  temporary written to `UDR0`
- [`print(bytearray)`](/limitations/#print-of-a-buffer) printed garbage; it now emits the
  CPython `bytearray(b'…')` repr
- [`millis()` / `ticks_ms` / `monotonic`](/stdlib/time/) count real milliseconds (a 1 s blink
  measured 1024 ms before), and `micros()` no longer jumps backward across a Timer0 overflow
- The [second PWM channel of a timer](/stdlib/pwm/#two-channels-on-one-timer) no longer
  disconnects the first — the shared `TCCRxA` COM bits are OR-ed in
- `print(3.25)` prints two rounded decimals with the trailing zero trimmed; it printed `3.2`
  before, and `0.05` printed `0.0`

### Language surface
- [Slice assignment](/limitations/#slices) dispatches through `__setitem__` with a `bytes` or
  list literal source: `microcontroller.nvm[0:4] = b'\xcc\x10\xca\xfe'`
- `for b in buf[0:n]` accepts runtime bounds, rewritten to a `range` loop
- `str.join` in an assignment: compile-time strings fold to a constant, and
  `''.join([chr(b) for b in buf])` builds a runtime string
- [`const` parameters](/limitations/#type-system-limitations) accept compile-time float
  constants (`Timer(freq=2.5)`)
- Exception catch-all forms, user exception classes, bare `except`; `__bool__` / `__len__`
  truthiness; `__call__`; n-ary `min` / `max`; `dict.get` on literal dicts
- A nested constructor argument types as its class in overload resolution, so `ADC(Pin(14))`
  picks the `Pin` overload; resolution now matches parameter types rather than declaration
  order

### Guardrails — was silent, now a located error
- A `const[...]` parameter rejects a runtime-varying argument; a loop variable passed to
  `Pin()` used to drive one fixed pin in silence
- An image larger than the chip's flash, and static data beyond its SRAM, are build errors
  quoting the part's real numbers
- Runtime tuples, filtered comprehensions with a runtime condition, `list` parameters,
  instance interpolation and iterator-protocol loops each get a specific diagnostic instead
  of misbehaving quietly

### Compatibility layers
- `machine.ADC(0)`-`ADC(5)` channel form beside `ADC(Pin(14))`; `machine.PWM.freq()` and
  `duty_u16()` getters; `machine.SoftI2C` on any two pins
- Honest errors where the silicon cannot deliver: `machine.unique_id()`, the no-arg
  `UART.readline()`, and `machine.I2C(scl=…)` on fixed TWI pins
- CircuitPython `microcontroller.nvm` slices end to end, plus `watchdog` and `cpu.reset_reason`

Requires `pymcu-avr >= 0.1.0a9` for the paired codegen fixes — float conversions and
comparisons, wide constants, and linker `MEMORY` regions.

## v0.1.0a9 — Alpha 9 (2026-08-17)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a9)

**macOS Intel comes back.** It had been dropped from the publish matrix because no native AVR
toolchain existed for it, and building one meant compiling GCC for hardware Apple has stopped
shipping. [The toolchain is architecture-independent WebAssembly
now](/getting-started/installation/#the-avr-toolchain-is-webassembly), so that justification
is spent — and `pymcu-arm` and `pymcu-pic` were still shipping the wheel anyway, because
their removal never reached a release. Restoring it costs a publish step rather than an Intel
machine: .NET cross-compiles `osx-x64` from the Apple Silicon runner. What it does not buy is
certainty — GitHub retired the `macos-13` runner, so nothing here is verified on that platform.

- The library authoring guide no longer opens by warning that the index is offline and telling
  authors to install with `--from-pypi`. The [index](/libraries/) had been online since
  the day before

## v0.1.0a8 — Alpha 8 (2026-08-17)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a8)

**The library index stops blaming libraries for being used out of scope.** The published
catalogue reported `pymcu-lib-dht` as **failed** on the RP2040, quoting a `TypeError` — for a
library whose manifest declares `arch = ["avr"]` and which the driver correctly skips on ARM.
That reads as "this library is broken" about something it never promised.

- A failure on an architecture the author never declared is now
  [`unsupported`](/libraries/#reading-the-index), not `failed`. Measuring with the
  compatibility filter off stays as it was — code that beats a cautious manifest is still
  published as `ok`; what was wrong was the label
- A **declared** architecture that does not build is still `failed`: a broken promise is
  exactly what the index has to be able to state
- **Declaring nothing claims everything** — silence in a manifest is not a way out of being
  measured — and a specific **chip list wins** over the architecture

## v0.1.0a7 — Alpha 7 (2026-08-17)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a7)

### Fixed
- **HTTPS works on a stock macOS Python**: every request the driver makes — the library index,
  sdists, tool downloads — failed with `CERTIFICATE_VERIFY_FAILED` on a python.org install
  until the user happened to run `Install Certificates.command`, and the error read like the
  server being down. `certifi` is now a real dependency rather than something the code hoped
  was installed
- **A missing backend no longer counts against a library**: the index reports `unmeasured`
  rather than `failed` when the build never ran, so the compatibility matrix stops reflecting
  whichever extras the measuring machine happened to have
- A local TLS failure is no longer published as "no sdist available" beside projects that do
  publish one
- `pymcu install` no longer prints `? bytes RAM`: the index does not measure RAM yet, and a
  question mark reads as a figure that got lost rather than one never taken

### Releases are tested before they are published
- `tools/smoke_release.py` installs the artifacts in a clean environment and puts them to work
  before anything reaches PyPI. Each check maps to a release that shipped broken while the
  build stayed green — including a package left unbumped, whose wheel `skip-existing` then
  discarded in silence

## v0.1.0a6 — Alpha 6 (2026-08-17)

Full notes: [GitHub release](https://github.com/PyMCU/PyMCU/releases/tag/v0.1.0a6)

### Third-party libraries
The compiler no longer sees a library's whole package, only its declared source directory
(`[library] sources`, `mcu/` by default). Before, anything a wheel carried was exposed: a
library's `examples/` answered `import examples` from any firmware in the world, and two
libraries shipping one shadowed each other with no diagnostic.

- A library package is now an ordinary Python package with `__init__.py`. Examples, tests and
  docs live at the distribution root and travel in the sdist, not the wheel
- `pymcu lint --library` rejects any `.py` outside the source tree
- `pymcu install --verify` compiles the modules the library declares instead of its example, so
  it needs nothing but the wheel
- `pymcu index build` measures examples from the sdist — one download per library
- A library with a broken manifest no longer aborts the build of projects that never used it:
  it is named and left out of the include path

Published libraries need the new layout; `pymcu-lib-neopixel` ships 0.2.0 for this reason. See
[Writing a library](/libraries/authoring/).

### Fixed
- **WS2812 on PB0** sent every bit as a zero (312 ns for a one, where ~800 is needed), so a
  strip on that pin stayed dark while the build reported success. The other eleven pins were
  fixed earlier; a comment threw the patch off this one
- **Port auto-detection**: the rule was written twice and `flash()` ran the copy the tests did
  not cover
- **`pymcu-sdk` is published again**: its surface had grown without a version bump, so the
  package in the repo and the `0.1.0a4` on PyPI were different under one number

### AVR toolchain moves to WebAssembly
The AVR backend is versioned separately; this landed in **`pymcu-avr` 0.1.0a6**, which the
`[avr]` extra now resolves to.

- `avr-as`, `avr-ld`, `avr-objcopy`, `cc1` and `cc1plus` ship as `wasm32-wasip1` modules in
  `pymcu-avr-toolchain-wasi` — **the same five modules on every platform**, 22.9 MB in one
  `py3-none-any` wheel, against 70.7 MB per platform for five native builds
- **Firmware is byte-identical**: 59 of 59 examples verified by sha256 against the native
  toolchain — 53 plain and 6 through the FFI path, C++ included — on Linux x64 and arm64,
  macOS arm64, and Windows x64 and arm64
- No Rosetta 2 on Apple Silicon, no MSYS2 on Windows, and no extra for C interop. An existing
  native install keeps working, and `PYMCU_AVR_WASI=0` forces it
- Fixes a latent defect along the way: `firmware.asm` is no longer rewritten in place, so a
  second preprocessing pass — which would have doubled the interrupt vector table's spacing and
  pointed every vector at a wrong address, silently — is impossible rather than merely unlikely

**`pymcu-avr` 0.1.0a7** then made the first build fast again: the C front ends are 48 of the 50
MB of modules and were being compiled whether or not a project had a line of C. They are now
built on the first C compilation and never before, and the one-time warm-up says what it is
doing instead of looking like a hang — first build **5.56 s → 0.82 s**, later builds
**2.30 s → 0.23 s**, cache on disk **114 MB → 6.2 MB**, same firmware. See
[Installation](/getting-started/installation/#the-avr-toolchain-is-webassembly).

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
