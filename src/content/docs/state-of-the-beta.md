---
title: State of the beta
description: The measured scope, hardware validation, test suites and known issues behind PyMCU 0.1.0b1.
---

Beta 1 (`0.1.0b1`) covers the compiler frontend and standard library, the AVR backend,
and the CircuitPython and MicroPython compatibility layers. ARM (RP2040 / RP2350), PIC
and RISC-V remain alpha by design. See [Supported targets](/targets/) for the maturity label
and hardware record of each backend.

This page records the measurements behind that claim. The final pre-release measurement was
taken on 2026-09-30 against compiler and stdlib `11e8bbe5`, AVR `740fe5c`, CircuitPython
`4fa38d6` and MicroPython `9f602f0`, with both compiler frontends wherever a suite supports
both. It is a snapshot, not a claim that the compiler has no bugs. The release work itself
kept finding bugs as each new instrument was applied.

## The five suites

| Suite | What it measures | Beta 1 result | Reproduction notes |
|---|---|---|---|
| User-program corpus | 51 user-style AVR programs, each with an expected outcome and size gate | 51 of 51 pass | [CircuitPython corpus](https://github.com/PyMCU/pymcu-circuitpython/blob/main/docs/corpus.md) |
| CircuitPython API parity | Every upstream `digitalio`, `analogio`, `busio`, `pwmio` and related symbol | 240 symbols: 180 provided, 60 allowlisted with a reason, 0 unexpected failures. The full layer had 522 passing tests | [CircuitPython parity](https://github.com/PyMCU/pymcu-circuitpython/blob/main/docs/parity.md) |
| MicroPython API parity | The real `machine`, `utime`, `uasyncio` and related firmware surface | 374 symbols measured, 0 failures. The full layer had 888 passing tests | [MicroPython parity](https://github.com/PyMCU/pymcu-micropython/blob/main/docs/parity.md) |
| HAL parity | Register-level HAL API across AVR, PIC12/14/18, RISC-V, RP2040 and RP2350 | 253 deviations checked: 5 strict passes and 248 reasoned allowlist entries | [HAL parity source](https://github.com/PyMCU/PyMCU/blob/v0.1.0b1/docs/library/hal-parity.md) |
| Differential oracle | 440 probes compiled and run on the AVR emulator, then diffed against CPython with both frontends | C# frontend: 415 of 426 runnable probes pass, 11 tracked failures and 14 skips. Python frontend: 412 pass, 14 tracked and 14 skipped | [Language oracle](/language-oracle/) |

The matching repository gates were also green: 3,352 compiler unit tests, zero
`verify_ir.py` regressions, and 3,966 passing AVR integration tests. Standard-library and
driver suites passed in both the full development environment and a narrower AVR-only
environment; the number collected differed because installed backends unlock additional
parameterized cases, but neither run had a failure.

The AVR integration suite now starts a cold boot with `R0` through `R31` (except `R1`) and
all SRAM filled with `0xFF`. This permanently prevents the emulator from granting a program
implicit zero-initialization that real AVR silicon does not guarantee. The frontend and AVR
startup code now identify and zero static-duration state explicitly.

## What the oracle knows is wrong

At the release measurement, 14 tracked probes represented 11 filed compiler issues. A
tracked probe is `xfail(strict)`: the suite remains green, but it fails if the observed bug
changes without the fixture and disclosure changing too. Three probes run only with the
Python parser, which is why the frontend counts differ.

| Kind | Issues at the Beta 1 measurement | Effect on a program |
|---|---|---|
| Silently wrong value | [#364](https://github.com/PyMCU/PyMCU/issues/364), [#394](https://github.com/PyMCU/PyMCU/issues/394), [#395](https://github.com/PyMCU/PyMCU/issues/395), [#401](https://github.com/PyMCU/PyMCU/issues/401), [#426](https://github.com/PyMCU/PyMCU/issues/426), [#449](https://github.com/PyMCU/PyMCU/issues/449), [#521](https://github.com/PyMCU/PyMCU/issues/521), [#522](https://github.com/PyMCU/PyMCU/issues/522), [#525](https://github.com/PyMCU/PyMCU/issues/525), [#439](https://github.com/PyMCU/PyMCU/issues/439) | Unannotated loop accumulators, multi-clause comprehensions, selected ZCA dunders, array sequence patterns, post-definition methods, module-qualified annotations, Python-parser PEP 695 parameters, mixed signed byte comparisons and one Python-parser tuple pattern can return a wrong result without a diagnostic. |
| Correct refusal, misleading reason | [#400](https://github.com/PyMCU/PyMCU/issues/400) | Reading an `Enum` member outside a plain assignment RHS says the enum class is undefined instead of naming the real limitation. |

The live source of truth is the `# tracked: #N` header in each probe under
`tests/oracle/probes/` in the `pymcu-avr` repository. This table is the release snapshot.

One important bug was fixed during release preparation. A class attribute literally named
`value`, whose class implements `__get__` and `__set__`, bypassed the descriptor protocol
and fell into PyMCU's MMIO or collapsed-scalar shortcut. This is exactly how
`adafruit_register` and `digitalio` name descriptors. The original probe now passes. Three
nearby shapes are deliberately refused instead: a descriptor with `__set_name__`, a class
level descriptor read such as `Box.value`, and an assignment to a non-data descriptor.

## What a second instrument found

The oracle exercises constructs selected by compiler authors. A second instrument compiles
unmodified upstream Adafruit CircuitPython and MicroPython drivers, then compares their bus
transactions with the same code under the real interpreter. On 2026-09-25 it opened 23
issues in one day, including combinations no focused probe had selected.

The following issues were still open and reachable from a Beta 1 program at the published
measurement. Wrong-answer cases come first.

| Issue | What it does | Scope |
|---|---|---|
| [#490](https://github.com/PyMCU/PyMCU/issues/490) | ATtiny register maps are shifted, so an ATtiny85 ADC access can write the wrong register | ATtiny only |
| [#494](https://github.com/PyMCU/PyMCU/issues/494) | `int()` is an `int16` cast, so `int(46051.7)` wraps to `-19485` | Every target |
| [#506](https://github.com/PyMCU/PyMCU/issues/506) | A string returned from a function or accepted by a bare parameter can have its 16-bit interned id truncated to one byte | Every target |
| [#510](https://github.com/PyMCU/PyMCU/issues/510) | A single-call-site method is inlined and a filled ellipse draws the wrong pixels | Every target |
| [#495](https://github.com/PyMCU/PyMCU/issues/495) | A compile-time chip predicate bound in another module can stop folding, leaving both branches in firmware | Every target |
| [#500](https://github.com/PyMCU/PyMCU/issues/500) | Duplicate per-chip HAL modules can drift, so the unwired copy describes behavior the facade does not have | Every target |
| [#449](https://github.com/PyMCU/PyMCU/issues/449) | `t.uint8` after `import pymcu.types as t` can wrap where the bare annotation promotes | Every target |
| [#446](https://github.com/PyMCU/PyMCU/issues/446) | A constructor copying a field from another instance can read zero instead of the stored value | Every target |
| [#439](https://github.com/PyMCU/PyMCU/issues/439) | A tuple pattern can compile under the Python frontend and take the wrong branch | Python frontend only |

Two toolchain issues also affected whether measurements could be trusted:
[PyMCU#496](https://github.com/PyMCU/PyMCU/issues/496) documented that the frontend/backend
IR wire format had no version field, and [PyMCU#504](https://github.com/PyMCU/PyMCU/issues/504)
tracked the diagnostic text format already parsed by editor plugins. Diagnostic-quality
issues [#505](https://github.com/PyMCU/PyMCU/issues/505) and
[#509](https://github.com/PyMCU/PyMCU/issues/509) did not change generated firmware.

## Two known refusals not filed at the measurement

These were reproducible under both frontends but were not silent wrong values:

- `x.value += n` can be refused in a multi-field class, through a property setter, or
  through a receiver such as `w.sensor.value`. Renaming the field avoids the MMIO
  `.value` shortcut.
- A user function named `claim` or `asm` can be captured by PyMCU's intrinsic of the same
  name once a chip module is imported. Rename the function.

## What ran on real hardware for this release

On 2026-10-04 the release candidate at `11e8bbe5` ran two unmodified Adafruit
CircuitPython libraries on a real Arduino Uno (ATmega328P):

- `adafruit_ssd1306`, together with `adafruit_framebuf` and `adafruit_bus_device`, drove an
  SSD1306 128x32 OLED over I2C. Two Game of Life programs ran correctly, one drawing with
  `pixel()` and one using `fill_rect` and `time.sleep`. A third program rendered text with
  `display.text()` and the embedded `font5x8.bin` font.
- `adafruit_hcsr04` ran its upstream `hcsr04_simpletest.py` with only the pins changed to D5
  for trigger and D2 for echo. It reported correct distances.

These are the only two Adafruit libraries claimed as silicon-tested for Beta 1. Every other
Adafruit figure in this page and the release changelog is a compile or AVR-emulator result,
including the byte-for-byte comparisons against interpreter-generated I2C traffic.

## Five silent wrong values found after the candidate was cut

All five programs below build cleanly under Beta 1 and produce a wrong value. None had an
oracle probe, fixture or filed issue in the candidate. All are planned for Beta 2.

**Status on `main` (2026-10-05).** The first, second and fifth are fixed on `main`. The
third and fourth are fixed or turned into a diagnostic on `fix/silent-list-tuple`, which was
still in review. Until a release carries the relevant fix, use the stated workaround.

### Reassigning a literal string to a runtime string

```python
from pymcu.chips.atmega328p import GPIOR0
from pymcu.types import uint8

text = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
print(text)
n: uint8 = GPIOR0.value
text = f"{n}"
print(text)                  # Beta 1 prints the old literal again
```

CPython prints the literal and then `0`; Beta 1 prints the literal twice. Bind the built
string to a different name (`text2 = f"{n}"`). Fixed on `main` for Beta 2.

### An f-string containing only a string literal

```python
text = f"{'literal string'}"
print(text)                  # Beta 1 prints 257
```

Use `text = "literal string"`. Fixed on `main` for Beta 2.

### Appending class instances to a list

```python
from pymcu.types import uint8

class Counter:
    def __init__(self, n: uint8) -> None:
        self._n = n

    @property
    def n(self) -> uint8:
        return self._n

xs = []
for i in range(2):
    xs.append(Counter(i))
print(xs[0].n, xs[1].n)     # Beta 1 prints "0 0", not "0 1"
```

Use a literal (`xs = [Counter(0), Counter(1)]`) or separate instance names. Manually
unrolling the appends does not help. The Beta 2 branch refuses these shapes instead of
silently losing the fields.

### Returning `(bytearray, scalar)`

```python
from pymcu.types import uint8

def search_rom(seed: uint8):
    new_rom = bytearray(8)
    new_rom[0] = 40
    return new_rom, seed

rom = bytearray(8)
diff: uint8 = 7
rom, diff = search_rom(9)
print(rom[0], len(rom), diff)  # Beta 1 prints "0 8 9"
```

Return the buffer alone and pass the scalar separately. The scalar lands correctly; only
the buffer result is lost. The fix was in review for Beta 2.

### Two indexed tuple-return calls in one expression

```python
from pymcu.chips.atmega328p import GPIOR0
from pymcu.types import inline, uint8

@inline
def pair(v: uint8) -> (uint8, uint8):
    return v, v + 1

s = GPIOR0.value
r: uint8 = pair(s + 3)[0] + pair(s + 8)[0]
print(r)                     # Beta 1 prints 16, not 11
```

Bind or unpack each call before combining the results. Fixed on `main` for Beta 2.

## What beta does and does not claim

- Beta means the documented language surface is implemented and test-covered on AVR and
  has real-silicon validation. For third-party libraries, Beta 1's silicon coverage is the
  two Adafruit drivers named above.
- It does not mean every compatibility-layer symbol is implemented. The parity suites make
  each missing or divergent symbol explicit and tracked.
- Every allowlisted deviation and every tracked oracle failure names an issue. A known,
  tracked wrong result is still a bug, but it is not hidden from the release claim.
- `keypad.KeyMatrix` and the CircuitPython storage and OS modules are outside Beta 1 by
  decision, tracked as
  [pymcu-circuitpython#13](https://github.com/PyMCU/pymcu-circuitpython/issues/13) and
  [pymcu-circuitpython#16](https://github.com/PyMCU/pymcu-circuitpython/issues/16).
- Integer-width inference remains experimental where it is feature-gated. It is not part of
  the Beta 1 surface claim.

For language-level alternatives and measured library blockers, see
[Limitations](/limitations/). Release-level changes are summarized in the
[changelog](/changelog/).
