---
title: Limitations
description: What PyMCU cannot compile, why, and the idiomatic bare-metal alternative for each case.
---

**Read this before writing your first project.**

PyMCU compiles a statically-typed, allocation-free subset of Python to bare-metal machine
code. There is no runtime and no interpreter, and **no heap by default**: nothing allocates
behind your back, and every size is fixed at compile time. A heap exists only if you ask for
one — the opt-in `list[T]` bounded list (see [Dynamic memory and containers](#dynamic-memory-and-containers)
below) links a small bump allocator and a GC on AVR. Many standard Python features are
therefore incompatible with this model.

:::note[Standard library philosophy]
Because of the architectural differences between a PC and a bare-metal microcontroller,
PyMCU **does not attempt to replicate the CPython standard library 1:1**.

Instead, PyMCU adopts the philosophy and API design of **MicroPython and CircuitPython**
(specifically the `machine` and `board` modules) as its official user-facing standard
library. This ensures that code written for PyMCU looks familiar to developers coming from
the broader Python-on-hardware ecosystem, even though it executes entirely differently.
See [MicroPython compat](/compat/micropython/) and
[CircuitPython compat](/compat/circuitpython/).
:::

This page lists every known unsupported feature, explains _why_ it cannot be compiled, and
suggests the idiomatic PyMCU alternative where one exists.

---

## Dynamic memory and containers

| Feature                                    | Why it fails                       | Alternative                                                                                                                                                |
| ------------------------------------------ | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `list.append(x)` on a **fixed-size** array | Fixed arrays have no `append`      | `list[uint8]` heap-bounded list, or `uint8[N]` fixed-size array                                                                                            |
| **Growing** `dict` (unbounded)             | Hash table requires heap           | [`pymcu.collections.FixedDict(capacity)`](/stdlib/#module-index) (mutable, fixed footprint), a closed dict literal (below), or `match / case` key dispatch |
| **Mutable** `set` (`.add()`)               | Hash set requires heap             | Closed set literal (below), or a `uint8` bitmask                                                                                                           |
| Dict / set **comprehensions**              | Would build a container at runtime | Build a closed literal, or fill a `FixedDict` in a loop                                                                                                    |

**Supported:** `list[T]` (`x: list[uint8] = list()`) is the one **opt-in** heap in PyMCU — it
compiles to a bounded bump allocator with a shadow-stack GC, linked only into firmware that
actually uses it, and it supports `append()`, `len()`, `x[i]` and `for v in x:`. It is
currently **AVR-only** (suitable for the ATmega328P's 2 KB of SRAM and up); on ARM and PIC it
is not available. `bytearray(N)` and `bytearray(b"...")` compile to SRAM `uint8[N]` arrays and
involve no allocator at all.

The bounded list may cross a function boundary as `list[T]`. A regular function with a list
parameter or return is specialized at its call sites because no fixed-width scalar ABI can
carry the list. An unannotated `[]` learns its element type from the first append, while a
list literal takes the widest element type it needs. Appending an incompatible value is a
located error that suggests the explicit `list[T]` spelling.

`array.array(typecode)` uses the same bounded list implementation. Integer typecodes map to
the matching PyMCU width: `B` / `b`, `H` / `h`, and `I` / `L` / `i` / `l`. Float, double and
64-bit typecodes are refused. A bare `array.array` parameter or return takes its element
width from the list passed by the caller.

**Closed dict/set literals** (`d = {0: 10, "mid": 2}` / `OK = {1, 3, 5}`) bind compile-time
lookup tables with no storage: `d[const]` folds to its value, `d[runtime_key]` lowers to a
compare chain that raises `KeyError` (catchable with `try/except`) on no match, `x in d` /
`x in {...}` test membership, and `len(d)` folds. They are read-only.

**`pymcu.collections.FixedDict(capacity)`** is the mutable counterpart: a fixed-capacity
integer dict (open addressing over per-instance fixed arrays — no heap, no GC) with Python
semantics where they fit a fixed footprint: `d[k]` / `d[k] = v`, `KeyError` on a missing
key, `ValueError` when inserting into a full dict, `k in d`, `len(d)`, `get(k, default)`,
`pop(k)`, `clear()`. The capacity is a compile-time constant.

Fixed-size arrays `arr: uint8[N]` support both constant- and variable-index access, and
equal-length slice assignment (`arr[a:b] = src`, including overlapping same-array copies).

All scalar element widths work on AVR, including `int32`, `uint32` and `float`, through a
field as well as a local name. A negative constant index counts from the end. A constant
index outside the array is a compile-time `IndexError`.

**Runtime-sized `bytearray(n)` on AVR.** A runtime size uses a static arena when the compiler
can prove the allocation runs at most once: at module level outside a loop, or in an inlined
`__init__` reached once from module scope. The arena is reserved at startup and never freed.
`pymcu build` reports its reservation and links no allocator when no program uses it.

Indexing, stores, augmented stores, negative indices and `len()` work on the resulting
buffer. Slicing, `memoryview`, passage to a normal `bytearray` parameter, allocation inside
a non-inlined function, an ISR, a loop or a multiply-called path do not. Those shapes are
refused with the once-only alternatives. ARM, PIC and RISC-V refuse runtime-sized buffers.

**Rule of thumb:** storage has a compile-time bound, except for AVR `bytearray(n)` where the
compiler proves one startup allocation and reserves a static arena for it.

---

## String operations

| Feature                                            | Why it fails                                       | Alternative                                                                                            |
| -------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `f"..."` inline in arbitrary expressions           | No general runtime string objects                  | Assign it to a name first (`s = f"..."` builds a fixed buffer), or stream it: `print(f"...")`          |
| `str.split()` as a value                           | There is no list of runtime strings to return      | Iterate it directly with `for` or `enumerate`; the receiver and separator must be compile-time strings |
| `str.format()` with named fields or `**kwargs`     | Only compile-time positional formatting is lowered | Use `{}`, `{0}`, numeric format specs, `*seq`, or an f-string                                          |
| `str.join()` over a runtime-length sequence        | The result must have a compile-time bound          | Join compile-time strings, or a formatted generator over a fixed sequence                              |
| `len(string_variable)` with several possible texts | There is no single compile-time string             | Use one compile-time text or a fixed runtime buffer                                                    |
| `str + str` concatenation                          | Heap allocation                                    | Separate `uart.write_str()` calls, or one f-string                                                     |
| `str[i]` on a runtime string                       | No runtime string object                           | Use `const[str]` parameters                                                                            |
| `s == "literal"` on an f-string value              | No runtime string comparison                       | Compare the underlying integers instead                                                                |

**Supported:** string literals in flash, raw strings `r"\n"`, `uart.println("literal")`,
`const[str]` runtime subscript, compile-time `str.format`, and runtime f-strings in streamed
and fixed-buffer value forms.

A name bound to one compile-time text keeps that text through module constants, parameters,
`super().__init__` and nested inline calls. `len`, indexing, slicing, substring membership,
equality, `strip`, `lstrip`, `rstrip`, `index`, `find`, `startswith`, `endswith`, `count`,
`replace`, `upper`, `lower`, and `str(x)` of a constant fold while compiling. `split()`
unrolls when used directly by `for` or `enumerate`; it does not return a list value.

`for ch in "ABC"` and `enumerate("ABC")` unroll up to eight characters. Longer strings
use a counter over the flash copy and expose each character as a runtime `uint8` accepted by
`ord()`.

### A string selected at runtime

When different runtime paths bind the same name to different literals, the name stores a
16-bit interned id and the texts remain in flash:

```python
s: str = "idle"
if seed > 10:
    s = "running"
print(s)
if s == "running":
    start()
```

This works for branch and loop rebinding, a global string changed by a function, and a
conditional expression. `print`, `uart.write_str`, `println`, and `==` / `!=` against a
literal dispatch on the id. `len(s)`, indexing, concatenation and passage to `const[str]`
remain errors because there is no single text to operate on. A conditional written directly
inside a sink lowers one literal write per arm and needs no id slot.

### f-strings

`f"..."` with **runtime interpolations** is supported streamed to a sink — the compiler
lowers each piece to a direct write (no heap, no format buffer):

```python
print(f"adc={raw} v={mv:04d}")
uart.write_str(f"t={temp:5d}")
uart.println(f"err 0x{code:02X}")
lcd.print_str(f"{hours:02d}:{mins:02d}")
```

A streamed interpolation also accepts a `float`. Unformatted `print(x)`, `str(x)`,
`repr(x)` and `f"{x}"` follow PyMCU's compact port of MicroPython's float32 formatter: 6 to
9 significant digits selected by a float32 round-trip check, with fixed or scientific
notation chosen at CPython's threshold. Representative output is `0.1`, `3.1415928` and
`1e+20`.

It is also supported **as a value**: `s = f"t={t} C"` builds the string into a
compiler-managed fixed `bytearray` whose size is statically bounded per part. On the value
form, `len(s)` is the formatted length, `s[i]` is a one-character string, `print(s)` /
`uart.write_str(s)` stream it, and re-assigning `s` in a loop reuses the buffer. Assign
the longest f-string first, since the buffer is sized at the first assignment.

Self-interpolation, such as `s = f"{s}..."`, snapshots the old bytes before rewriting the
buffer. It still cannot grow beyond the size established by the first assignment.

Not yet supported in the value form: `s == "lit"` comparison and f-strings inline in other
expression positions. Assign to a name first.

**Format specs** supported in interpolations: `{x:02x}`, `{x:X}`, `{x:08b}`, `{x:o}`,
`{x:5d}`, `{x:04d}`, plus float `{v:f}`, `{v:.Nf}`, `{v:W.Nf}` and zero padding such as
`{v:08.2f}`. Float precision is capped at 15 digits and rounds half-to-even from the exact
float32 expansion. Both streamed and fixed-buffer forms use the same formatter. Compile-time
constant integer interpolations fold into flash text.

### `print()` of a buffer

`print()` renders a `bytearray`, a slice of a fixed-size array (`print(arr[a:b])`) and a slice
of an object with `__getitem__` / `__len__` (`print(obj[a:b])`) as the faithful CPython repr,
escapes and all:

```python
buf: bytearray = bytearray(b"\xcc\x10\xca\xfe")
print(buf)              # bytearray(b'\xcc\x10\xca\xfe')
print(buf[0:2])         # bytearray(b'\xcc\x10')
```

The length has to be a compile-time constant — the repr is unrolled into direct writes, so
`print(buf[0:n])` with a runtime `n` has nothing to unroll.

---

## Exception handling

`try / except / raise / finally` are **supported** on AVR and ARM (RP2040 / RP2350) via a
zero-cost **flag error-propagation** model — _not_ `setjmp` / `longjmp`. A function that
raises marks the error (AVR: the SREG T flag via `SET` / `CLT` / `BRTS`; ARM: an internal
flag + code global pair) and returns normally; every call site inside a `try` tests the flag
and branches to the matching `except`. There is no `jmp_buf` and no stack unwinding, so the
happy path costs a single skipped branch per guarded call.

Because propagation rides on the function return, raise from a helper and catch it where you
call that helper:

```python
def read_sensor(raw: uint16) -> uint8:
    if raw > 1000:
        raise ValueError        # sets the error flag, returns to the caller
    return uint8(raw)

try:
    v: uint8 = read_sensor(adc.read())   # caught here if read_sensor raised
    handle(v)
except ValueError:
    handle_error()
finally:
    cleanup()
```

`ValueError`, `TypeError`, `IndexError`, `KeyError`, `NotImplementedError` and
`ZeroDivisionError` are builtins — no import required, exactly like CPython.
`ZeroDivisionError` is raised automatically on a runtime `//` or `%` by zero.

The full statement is supported: `try` / `except` / **`else`** / **`finally`**, plus a bare
`raise` to re-raise the active exception. `finally` runs on **every** exit path — normal
completion, a caught exception, propagation to an outer scope, and `return` / `break` /
`continue` out of the `try` (including a `break` or `return` inside `finally` that discards
the in-flight exception):

```python
try:
    v: uint8 = read_sensor(adc.read())
except ValueError:
    handle_error()
    raise                # bare re-raise — propagates to the caller
else:
    handle(v)            # runs only if no exception
finally:
    cleanup()            # always runs
```

**How it works, and its limits:**

| Property                          | Notes                                                                                                                                                                                                                                                               |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zero SRAM, zero happy-path cost   | No `jmp_buf`; each guarded call is followed by one branch, skipped when no error was raised                                                                                                                                                                         |
| Propagates across calls           | A `raise` inside a called function is caught at the call site in the caller's `try` — cross-function propagation **is** the model; there is no same-function restriction                                                                                            |
| Propagates to any depth           | An unmatched exception re-propagates to the **enclosing** `try`, then the caller, and so on — there is no single-nesting-level limit                                                                                                                                |
| Caught at call sites              | An exception is detected after a **function call** inside the `try`. Raise from a helper and catch it where you call it, rather than `raise`-ing directly in the `try` body                                                                                         |
| AVR + ARM only                    | On PIC, use return codes or sentinel values instead (`ZeroDivisionError` guards on `//` and `%` are still emitted there)                                                                                                                                            |
| Exception types are integer codes | Handlers match by code. A string literal costs one flash word; an f-string, concatenation or call is replayed as a deferred print when the exception is read                                                                                                        |
| Handler spellings                 | Bare, module-qualified and tuple forms all work, for example `except mod.Error:` and `except (A, B):`                                                                                                                                                               |
| User exception bases              | A `class E(A, B)` records an is-a edge for every base inside the `OSError` subtree, so `except A` and `except B` both catch an `E`; this is exception matching only, not general multiple inheritance                                                               |
| `raise X(...) from Y`             | Accepted as `raise X(...)`; there is no traceback, `__cause__` or `__context__`                                                                                                                                                                                     |
| Bare `raise` stays in the handler | A bare `raise` re-signals the record the enclosing `except` snapshotted, so it must expand inside that handler's scope. A bare raise inside a separately compiled function refuses at compile time; mark the helper `@inline` so the raise expands at the call site |
| `except E as e`                   | Binds a bounded object. `print(e)`, `str(e)`, `e.args[0]`, `e.args`, `len(e.args)` and `isinstance(e, X)` are supported. Integer `OSError` arguments may also expose `e.errno`                                                                                      |
| Unmatched at top level            | Initializes UART0 if necessary, prints `E:<TypeName>` or `E:<TypeName>: <message>`, then halts                                                                                                                                                                      |

:::note[Return codes are still often clearer for firmware]
`try / except` is now zero-cost on the happy path (no `jmp_buf`, one skipped branch per
guarded call), so the old "21 bytes of SRAM per `try`" objection no longer applies. Even so,
an explicit status return is frequently the clearest bare-metal style, reads the same on
every backend (including PIC), and makes the error path obvious at each call:

```python
STATUS_OK:    uint8 = 0
STATUS_RANGE: uint8 = 1

def read_sensor() -> uint8:
    if adc.read() > 1000:
        return STATUS_RANGE
    return STATUS_OK

match read_sensor():
    case STATUS_OK:    ...
    case STATUS_RANGE: ...
```

:::

### `CompileError` — compile-time intrinsic

`raise CompileError("msg")` is intercepted by the compiler and **aborts compilation** with a
`CompileError:` diagnostic. It never generates any runtime code or error-propagation
instruction. It is used throughout the native HAL to reject unsupported configurations at compile
time:

```python
from pymcu.exceptions import CompileError

match __CHIP__.arch:
    case "avr":
        ...
    case _:
        raise CompileError("SPI not supported on this architecture")
```

`CompileError` **cannot be caught** by `try / except` — compilation aborts before any binary
is produced.

### Unhandled exception output

When a `raise` has no active handler, PyMCU prints `"E:<TypeName>\r\n"` or
`"E:<TypeName>: <message>\r\n"` to UART0 and then halts. Useful for debugging from a serial
monitor:

```
E:ValueError
```

The transmitter need not already be active. When no code owns UART0, the unhandled path
initializes it at `[tool.pymcu] stdout_baud` (115200 by default). A program that already
configured UART0 keeps its active stream and rate.

Only exception types actually raised in the program have their name strings emitted in
flash — no overhead for unused exception codes. Chips without UART0 (ATtiny85 and friends)
skip the output and go directly to the halt loop.

`assert condition, msg` is a compile-time check: a statically false assertion is a
`CompileError`; a true or runtime-dependent assertion is stripped.

### Float edge cases

- Runtime float division by zero raises `ZeroDivisionError`, including when it occurs
  directly in top-level code. The unhandled path prints and halts rather than spinning.
- `x ** n`, `pow(x, n)` and `math.pow(x, n)` support runtime float operands. Zero to a
  negative power raises `ZeroDivisionError` for `**` and builtin `pow`, but `ValueError` for
  `math.pow`, matching CPython's two APIs. A negative base with a non-integral exponent is
  refused as `ValueError` because PyMCU has no complex type.
- `round(x[, n])` uses half-to-even. A runtime float goes through `pymcu.round2`; a
  compile-time integer follows Python's positive and negative digit semantics.
- `float("inf")`, `float("infinity")` and `float("nan")` fold at compile time. NaN compares
  false under `==`, `<`, `<=`, `>` and `>=`, and true under `!=`, both as a value and in
  `if` / `while` conditions. Separate backend and branch-optimizer bugs that once made
  `NaN > 1.0` true were fixed during Beta 1 work.
- A constant float-to-integer cast outside the target width is refused rather than inheriting
  a host-runtime conversion artifact.

---

## Functions and closures

| Feature                                                 | Why it fails                                         | Alternative                                                                        |
| ------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Closures capturing mutable vars                         | Closure cell requires heap                           | Pass captured values as explicit parameters                                        |
| `*args` / `**kwargs` from a runtime sequence or mapping | Only compile-time expansion has a fixed call shape   | Pass a literal or compile-time sequence/mapping, which is spliced at the call site |
| `functools.partial`                                     | Runtime partial object                               | Wrapper `@inline` function                                                         |
| Function target selected by runtime control flow        | The address must be known while compiling            | Use `match`, or index a `Callable[N]` table of known functions                     |
| Direct or mutual recursion                              | Static stack slots cannot represent recursive frames | Iterative equivalent; the diagnostic reports the full cycle                        |

**Supported:** `@inline` functions expand at call sites — zero call overhead, zero stack.
Non-`@inline` functions use a conventional call/ret ABI but cannot recurse. A free function
whose annotated parameter is a class instance expands like an inline function so the
instance fields stay in the caller. `lambda x: expr` (no closure capture) is inlined
at the call site. `nonlocal` is supported inside nested `@inline` functions. Unannotated
parameters and return types on outlined functions are inferred from call sites, defaults and
return expressions.

A call whose result is read must return a value on every reachable path. A missing return
would leave an untouched register or temporary, so it is refused at the call. Calling the
same function as a statement is permitted because no value is consumed.

**Function references are supported.** A function assigned to a `Callable`-annotated variable
captures its code address, and calling through that variable emits an indirect call
(`ICALL` on AVR):

```python
from pymcu.types import uint8, Callable

fn: Callable = add_one     # captures the address
r: uint8 = fn(10)          # indirect call
```

`funcref(fn)` is the explicit spelling of the same thing, and it is what you need to build a
`Callable[N]` dispatch table or to hand an address to inline assembly. A bare
`cb = my_handler` is supported too. The target must be a named function known at compile
time: no closures, bound methods or branch-selected targets.

---

## Classes and inheritance

| Feature                                         | Why it fails                                                 | Alternative                                  |
| ----------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------- |
| Multiple inheritance / MRO                      | C3 linearization is a runtime concept                        | Single-level inheritance only                |
| Runtime polymorphism (vtable dispatch)          | Requires vtable + heap class objects                         | Compile-time `match / case` dispatch         |
| Runtime `type()`                                | No runtime class object                                      | Use compile-time dispatch                    |
| `__repr__`, `__str__`                           | No runtime string formatting                                 | `uart.println()` with explicit fields        |
| `__new__` / `__init_subclass__`                 | There is no runtime allocation or class-creation event       | Do the work in `__init__`                    |
| `__del__`                                       | Static storage has no collection event                       | Explicit `deinit()` / `close()`, or `with`   |
| Descriptor `__set_name__`                       | No user code runs at class creation                          | Pass the owner/name explicitly               |
| Class-level descriptor read (`Box.value`)       | Descriptor lowering requires an instance                     | Read through an instance                     |
| Assignment to a non-data descriptor             | PyMCU has no per-instance `__dict__` for the shadowing value | Add `__set__`, or use another instance field |
| `dataclass`                                     | Metaclass + runtime heap                                     | Manual `@inline` class                       |
| `namedtuple` factory options and tuple indexing | The factory is a compile-time ZCA, not a tuple subclass      | Two-argument `namedtuple` plus field access  |

**Supported:** zero-cost abstraction (ZCA) `@inline` classes (zero SRAM), `@property` / `@name.setter`,
single-level class inheritance with `super()`, `with obj:` context managers
(`__enter__` / `__exit__`), `@staticmethod`, class-typed fields nested inside another class
(a `machine.Pin` wrapping the native HAL `Pin`, including through facade re-exports, and
including a **value-returning** method on the nested field — `self.pin.read()`), and operator
dunder methods (`__add__`, `__sub__`, `__mul__`, `__len__`, `__contains__`, `__getitem__`,
`__setitem__`, and all comparison / bitwise dunders).

Data descriptors with `__get__` and `__set__` are supported, including an attribute named
`value`. Nested classes are constructible and their constants resolve through
`Outer.Inner.NAME` and imported module aliases. `isinstance()` folds for known class
instances and the builtin `tuple`, `list`, `int` and `slice` shapes.

`collections.namedtuple("Name", ("a", "b"))` creates a compile-time ZCA class with those
fields, `__len__` and `__match_args__`. Defaults, rename and module options are not supported.

Without a comparison dunder, `==`, `!=`, `is` and `is not` use compile-time object identity.
Two constructions are distinct and `b = a` preserves identity. Ordering has no fallback and
is refused. `min` and `max` over instances are refused too.

Field layout includes assignments in `__init__`, property setters and helpers reached from
construction, including a base constructor reached through `super()`. An unannotated field
takes the widest compatible value from those writes. A later categorical change, such as a
number to a string, is a located error. A field read with no reachable write is refused; a
read that happens before a conditional write can still observe PyMCU's zero-initialized
static slot where an interpreter would raise `AttributeError`.

---

## Type system limitations

| Feature                                               | Why it fails                            | Alternative                                    |
| ----------------------------------------------------- | --------------------------------------- | ---------------------------------------------- |
| `complex` numbers                                     | Not implemented                         | Not available                                  |
| `Decimal`                                             | Requires heap                           | Not available                                  |
| `None` assigned where no Optional member exists       | `None` has no scalar payload width      | Add `None` to the annotation or use a sentinel |
| Union with more than four members                     | The tag encodes at most four states     | Split the API                                  |
| Union containing a buffer, sequence or class instance | These members have no scalar tagged ABI | Specialize an inline function at the call site |
| `TypeVar` / `Generic`                                 | Runtime generics                        | Separate `@inline` functions per type          |

**Unannotated widths.** A local, parameter, return, field or module global starts from the
first evidence the compiler can type. Later stores are checked, and the compiler repeats
type assignment with a wider or signed slot when needed. An accumulator does not widen from
the number of loop iterations, so `c = GPIOR0.value; c += 1` repeated 300 times still wraps
at the seed width. User-code loops warn about this `unannotated-accumulator`; annotate the
counter with its intended range.

**None and tagged unions.** `None` is compile-time when control flow decides it, so an
optional constructor argument can eliminate a whole branch and storage for one instance.
Where a plain function return, parameter, local or field chooses at runtime, `Optional[X]`,
`X | None` and scalar `Union[A, B, ...]` use the widest payload plus a one-byte member tag.
`is None`, truth tests, `isinstance` and `match` narrow the tag. `print` and f-strings emit
the active member. An unnarrowed operation dispatches per member and raises `TypeError` for
a live `None` member, matching CPython.

An inline or constructor Union parameter is specialized to the argument type at each call
site. A regular subroutine carries the member tag as part of its ABI. Annotation aliases,
including ones declared under a discarded `TYPE_CHECKING` guard, resolve to the same union.

**Declared parameter width.** A literal that does not fit a parameter is refused and names
the narrowed value. A computed value of the same magnitude is currently narrowed silently.
For example, `take(300)` against `n: uint8` is an error, but `take(len(bytearray(300)))`
passes 44. Library authors must choose widths for computed inputs, not only test literals.

:::note[`float` is supported on AVR and ARM]
IEEE 754 single-precision `float` works on **AVR** (pure-assembly `__fp_*` helper library,
~200-400 cycles per operation), on **RP2040** (the bootrom fast-float library, reached
through `__aeabi_f*` shims) and on **RP2350** (natively on the Cortex-M33 FPU).
`print(float)` works on AVR and ARM. Subnormals are treated as zero; NaN and Inf propagate
correctly. A cast from `float` truncates toward zero on the **value**, not on its raw bit
pattern, so `uint32(x * 100.0 + 0.5)` rounds as written. Float format specs work in both
streamed and fixed-buffer f-strings.
:::

:::note[`const[T]` takes float constants, and only constants]
A `const[T]` parameter accepts compile-time **float** constants as well as integers and
strings, so `Timer(freq=2.5)` binds. What it does not accept is a value that varies at
runtime: passing one is a located `CompileError` naming the parameter, rather than a silent
fold of whatever the variable happened to hold. `Pin(n)` with a runtime `n` is the case you
are most likely to hit — a pin identity has to be known at compile time for the GPIO access
to stay zero-cost.
:::

---

## Pointer arithmetic

`ptr[T]` in PyMCU is an **address alias for memory-mapped I/O**, closest to a C volatile
pointer:

```c
// C: what ptr[T] models
volatile uint8_t* const PINB = (volatile uint8_t*)0x36;
```

**What does work.** A `ptr[T]` may be passed as a function parameter and returned from a
function, and the address handed to `ptr(...)` may be computed at runtime — the native HAL
relies on all three:

```python
# a ptr parameter (pymcu.hal.avr.gpio)
def pin_pulse_in(pin_reg: ptr[uint8], bit: uint8, state: uint8, timeout_us: uint16) -> uint16:
    ...

# a ptr return value (pymcu.hal.avr.pwm)
def pwm_select_ocr(pin: str) -> ptr[uint8]:
    ...

# a runtime address (pymcu.hal.rp2040.gpio) — `pin` is a runtime variable
pad: ptr[uint32] = ptr(PADS_BANK0_BASE + 4 + 4 * pin)
pad.value = 1 << 6
```

Reads and writes go through `.value`, and augmented assignment on `.value` works.

**What does not work** is treating a `ptr` as an iterator you can walk:

| Operation                      | Example                                                    | Why it fails                                                                            |
| ------------------------------ | ---------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Pointer advance                | `p = p + 1`                                                | There is no pointer arithmetic in the IR — recompute the address instead                |
| Pointer difference             | `p - q`                                                    | Not in the IR                                                                           |
| Array address                  | `ptr(buf)` where `buf` is a fixed array                    | The assembler assigns the array label; the compiler has no scalar address value to pass |
| Register base + runtime offset | a base address already held in a register, plus a variable | Only `ptr(<expression>)` is lowered; there is no base-register addressing form          |

`p[i]` is bit indexing, not element indexing, but it does support both constant and runtime
bit numbers when `p` holds a runtime address. Reads, writes and augmented writes load the
register at that address, test or change its bit, and store it back.

A `ptr[T]` may also be an instance field. The element width travels through fields,
parameters and returns, so `.value` keeps the declared access width. A `ptr[T]` declaration
inside a class body defines a grouped peripheral namespace, such as
`Timer1.TCCR1A.value` and `Timer1.TIFR1[Timer1.TOV1]`. The class has no runtime existence and
calling it is refused. Grouped peripherals in `pymcu.chips.<chip>` are the stable register
surface; loose register names remain HAL implementation detail. See
[Writing a library](/libraries/authoring/).

**Idiomatic alternative — fixed arrays with a variable index:**

```python
buf: uint8[16] = [0] * 16
i: uint8 = 0
while i < 16:
    buf[i] = compute(i)   # compiles to: LDD / STD with Y+offset
    i = i + 1
```

`uint8[N]` arrays with a runtime index already compile to efficient `ld` / `st` with
Y+offset addressing on AVR — no pointer arithmetic needed.

For performance-critical pointer walks in assembly on AVR, use the Z register (`r30:r31`)
with `ld r24, Z+` / `st Z+, r24` to auto-increment through a buffer:

```python
asm("""
ldi  r30, lo8(my_buf)
ldi  r31, hi8(my_buf)
ldi  r18, 16          ; length
_loop:
    ld   r24, Z+      ; load byte and advance pointer
    dec  r18
    brne _loop
""")
```

---

## Iterators and comprehensions

| Feature                                                      | Why it fails                                | Alternative                                                                            |
| ------------------------------------------------------------ | ------------------------------------------- | -------------------------------------------------------------------------------------- |
| List comprehension over a **runtime** iterable               | Length not known at compile time            | `for` loop with a fixed-size array                                                     |
| `if`-filtered comprehension with a **runtime** condition     | The result length would vary at runtime     | Keep the filter compile-time constant, or a `for` loop with an explicit index          |
| Tuple literal passed as a runtime value or stored in a field | A tuple is a compile-time sequence here     | Separate variables, or a fixed-size array                                              |
| Dict comprehension                                           | Would build a container at runtime          | Closed dict literal, or fill a `FixedDict` in a loop                                   |
| Set comprehension                                            | Would build a container at runtime          | Closed set literal, or a `uint8` bitmask                                               |
| Generator expression used as a lazy value                    | No runtime iterator object                  | Pass it directly to `all`, `any`, `sum`, `min` or `max`, or write a generator function |
| `yield` inside an `@inline` function                         | There is no independent state-machine frame | Use a regular function or method                                                       |
| `yield` used as an **expression** (`x = yield v`)            | No two-way generator protocol               | One-way `yield` only                                                                   |
| `map()` / `filter()` with runtime iterables                  | Lazy iterator requires heap                 | Explicit `for` loop                                                                    |

**Supported:** `for i in range(N)` with runtime or constant bounds, fixed arrays, list and
tuple literals or names, `enumerate`, `zip`, `reversed`, string split iteration, and
single-clause list comprehensions with compile-time length.

The `range` counter is sized from its bounds: `range(300)` is 16-bit and
`range(200, -1, -1)` is signed. A short constant range or sequence unrolls when the body is
cheap; an expensive body uses a counter loop. A loop variable passed to a `const` parameter
stays compile-time and forces unrolling. Afterward it holds the last visited value. A
zero-iteration range leaves it at `start`, unlike Python where it may remain unbound.

Constant strings and pairs work as loop elements, including
`for pin, name in [(board.D2, "D2"), ...]`. A named constant list or tuple of any length
becomes flash data when a counter loop needs it; the element width comes from the widest
element.

**Nested comprehensions are not supported.** A comprehension nested inside another is
refused. Two `for` clauses in one comprehension are a known silent-wrong-code case tracked by
[PyMCU#394](https://github.com/PyMCU/PyMCU/issues/394), and a filter clause is refused. Use
one clause that fills a fixed array, or explicit nested loops.

A generator expression is supported only as the direct argument of `all`, `any`, `sum`,
`min` or `max`, over a known-length tuple, list, constant range, compile-time string or fixed
array. `all` and `any` short-circuit, `sum` honors its start value, and a generator `if`
clause filters. Anywhere else, there is no iterator object to return.

### Compile-time sequences in classes

A driver may accept a list of pins, instances or numbers and keep it in a field. The list is
a compile-time sequence of the original elements, not a copy. Constant subscripts, `for`,
`len`, and a method call through a runtime index work. A runtime subscript that extracts the
instance itself is refused, and runtime method dispatch past eight elements is refused to
avoid emitting a large compare-and-expand chain.

A constant numeric table such as `DIGITS = [0x3F, 0x06, ...]` is emitted to flash only when
a runtime index needs it. A writable table must declare SRAM storage explicitly. A dict of
same-length rows becomes a rectangular flash table and may use one-character string keys as
character codes.

A class instance as a sequence element is refused, by name. A compile-time
sequence (the shape above) and a growable `xs = []; xs.append(...)` list are two different
representations: the sequence flattens every element at the call site, while a growable list
is a real heap buffer that `append`, `len()` and a run-time index all read through an
address. A class instance has no such address here -- `self.x` is flattened to `<name>_x` at
compile time, never a value with storage of its own -- so `xs.append(Counter(i))` (a fresh
instance), `xs.append(c)` (an existing named one) or `xs.extend([Counter(i)])` had nothing
for the element to copy: it silently stored the instance's own, never-written handle name,
and every element after the first read back as 0, indistinguishable from a real field value
(`xs[1]` answered `xs[0]`'s field). The same holds for `xs[i] = Counter(...)` into scalar
element storage, for `bins[0][0] = Counter(...)` into a nested list's element storage, for
the annotated initializer `xs: list[uint8] = [c]`, for `a = xs[i]` reading an element of an
instance array as a value (the element names the instance, not a byte -- `xs[i].field` and
`xs[i].method()` stay legal), and for an instance element inside a tuple return, including
one produced by a method call such as `factory.make()`. Refused now, naming the class: build
a fixed-size array of
instances instead (`xs: Pair[N]` then `xs[i] = Pair(...)`, RFC 0001 Model B -- a class with
at least two fields), which gives each instance real, run-time-indexed storage and a shared
method body.

Two-dimensional grids written as `[[0] * W for _ in range(H)]` or
`[bytearray(W) for _ in range(H)]` flatten to one fixed `W * H` array. `g[y][x]`, row length,
row iteration and a temporary row view work. Passing, returning, storing, comparing or
slicing a row view is refused. `[[0] * W] * H` is refused because Python makes all rows
aliases of one object; use the comprehension form.

A method is not a field. Assigning `p.value = 1` to a class whose `value` is a method is
refused and suggests `p.value(1)`. CircuitPython's `DigitalInOut.value` is a property, so
assignment to that API remains valid.

### Slices

| Form                                             | Status                                                                                                                                      |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `b = arr[1:3]` / `arr[::2]` (slice **read**)     | Compile-time constant bounds only — the result is a fixed-size array sized at compile time                                                  |
| `arr[a:b] = src` (slice **assignment**)          | Supported, equal length, from a list / `bytes` literal / array / slice, including overlapping copies of the same array (snapshot semantics) |
| `obj[a:b] = src` through `__setitem__`           | Supported — lowers to one `__setitem__` call per byte                                                                                       |
| `for x in buf[lo:hi]` (slice **iteration**)      | Supported with **runtime** bounds; rewritten to a `range` loop over the backing array                                                       |
| `for x in buf[lo:hi:step]` with a runtime `step` | Rejected with a diagnostic — the step has to be a compile-time constant                                                                     |

A slice _read_ with runtime bounds (`b = buf[0:n]`) has no lowering: the result would need a
runtime-sized array. Iterate it instead, or index the backing array directly.

The `__setitem__` form is what makes the canonical CircuitPython persistence pattern compile:

```python
import microcontroller

microcontroller.nvm[0:4] = b"\xcc\x10\xca\xfe"   # one byte-write per element
```

### Generators (`yield`)

A function or bound method containing `yield` lowers to the same zero-cost state-machine
class the compiler builds for `async def`. No heap or `asyncio` is needed. `poll()` returns
`2` (yielded), `1` (still working) or `0` (done), with the produced value in `._value`, and
`for x in gen(...)` desugars to a poll loop with Python-exact `break` / `continue`
semantics:

```python
def countdown(n: uint8):
    while n > 0:
        yield n
        n = n - 1

for v in countdown(5):
    print(v)
    if v == 2:
        break
```

The limits are the ones in the table above: no `yield` inside an `@inline` function and no
two-way `yield` expression. `yield from` delegates to another supported generator.

---

## Async and concurrency

| Feature                                        | Why it fails                                                                   | Alternative                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| Awaiting another coroutine or future           | Sub-future fields need ZCA construction outside `__init__` (not supported yet) | Call the coroutine and poll it, or restructure with `asyncio.gather` |
| `await` as an **expression** (`x = await f()`) | The state machine only splits at statement boundaries                          | `await` the sleep, then read the result from `._value`               |
| `threading` / `multiprocessing`                | An OS is required                                                              | `@interrupt` ISRs                                                    |

**Supported:** `async def` / `await` (compiled to a zero-cost state machine; requires
`import asyncio`). `await asyncio.sleep()` / `sleep_ms()` works anywhere in the body —
inside `if` / `elif` / `else`, `while <cond>` and `for i in range(...)` at any nesting,
with `break` and `continue`; `return expr` surfaces the result via `._value`. Executors:
`asyncio.run(coro)` and `asyncio.gather(a, b)`. Locals become state-machine fields only when
they survive a suspension.

Also supported: the `@interrupt` decorator for hardware ISRs, `Pin.irq(trigger, handler)`
for external pin interrupts, and atomic flag patterns via `GPIOR0` on AVR.

One routine may occupy only one interrupt vector. Registering the same function at a second
vector is refused. Define a second ISR that calls a shared body when two vectors need the
same behavior.

:::caution[Timer0 and `millis()` / `ticks_ms()` on AVR]
`millis_init()` is auto-injected for `ticks_ms()`, `monotonic()` and ATmega async code. It
runs **Timer0** with prescaler 64. PWM on PD5/PD6 (Arduino D5/D6) at the default frequency
can share the timer. A different frequency would change the time base and is refused at
compile time, suggesting D3/D11 or D9/D10. Direct CTC or other Timer0 reconfiguration is
still the program author's responsibility.

A Timer0 overflow is 1024 µs, not 1000 µs. `millis()` — and everything layered on it:
`ticks_ms()`, `time.monotonic()`, `supervisor.ticks_ms()` — carries the Arduino-style
fractional correction (1 ms per overflow plus 3/125 accumulated in eighths), so it counts
real milliseconds instead of running 2.4 % slow. `micros()` reads the raw overflow count
plus `TCNT0` and is monotonic across an overflow.

`delay_ms()` and `delay_us()` are unaffected — on AVR they are a software busy-loop with no
hardware timer dependency.
:::

---

## Imports and modules

| Feature                                | Why it fails                                         | Alternative                                                |
| -------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------- |
| Third-party PyPI packages              | Only the `pymcu` stdlib is compiled                  | Implement it in the `pymcu` stdlib, or use `@extern` (AVR) |
| `importlib` / dynamic imports          | Runtime module loading                               | Not available                                              |
| Circular imports                       | Not supported                                        | Restructure the module dependencies                        |
| A function defined twice in one module | PyMCU compiles the first while Python binds the last | Rename one, or use typed inline overloads                  |

**Supported:** `import foo`, aliases, `from foo import Bar`, `from foo import *`,
`from package import submodule`, relative imports in both forms, package re-exports,
multi-module projects, the PyMCU standard library and both compatibility layers. Star
imports use `__all__` when present and otherwise bind public top-level names.

Import aliases are file-scoped and may be rebound to an instance. `from package import
submodule` binds the submodule when the package has no other member by that name. Package
`__init__.py` re-exports resolve to the one original definition. `from __future__ import X`
is a no-op because the compiler already reads annotations directly from source.

`import types` is deliberately refused. `pymcu/types.py` is the compiler's special type
module, not Python's `types` module. `import os`, `import uos` and `from os import uname`
resolve to the PyMCU OS facade. `uname`, `os.name` and `os.sep` are compile-time target
facts; `stat` and `listdir` operate on embedded ROMFS files.

With a configured compatibility layer, guards on `sys.implementation.name` and version,
`sys.platform`, and `os.uname()` fold to the facts reported by the selected MicroPython or
CircuitPython board. This lets upstream platform-selection code eliminate dead branches.

Module-level objects in project modules are constructed in import order before the entry
module's statements. A bare `def main()` runs after module-level startup. An explicit
`main()` or guarded `if __name__ == "__main__": main()` marks its exact execution point, so
statements after the call run afterward. A second call is refused.

---

## Built-ins summary

| Built-in                                                        | Status             | Notes                                                                                                                                                                                                                               |
| --------------------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `print(str)` / `print(int)`                                     | ✅ Supported       | Routes to UART; `sep=` and `end=` take a compile-time string literal, `file=` is not supported                                                                                                                                      |
| `print(float)` / `str` / `repr` / unformatted f-string          | ✅ Supported       | MicroPython-style float32 output, 6 to 9 significant digits; AVR and ARM                                                                                                                                                            |
| `print(bytearray)` / `print(arr[a:b])`                          | ✅ Supported       | CPython repr — `bytearray(b'\xcc\x10')`; the length must be compile-time                                                                                                                                                            |
| `range(n)`                                                      | ✅ Supported       | Runtime or constant loop bounds, membership, `reversed` and `enumerate`; not a standalone value                                                                                                                                     |
| `len(arr)` / `len(b"...")`                                      | ✅ Supported       | Compile-time constant fold                                                                                                                                                                                                          |
| `abs(x)`                                                        | ✅ Supported       | Intrinsic                                                                                                                                                                                                                           |
| `min(a, b)` / `max(a, b)`                                       | ✅ Supported       | Also fixed arrays and `key=f`; the key is evaluated once per operand                                                                                                                                                                |
| `sum(iterable)`                                                 | ✅ Supported       | Compile-time fold or unrolled additions                                                                                                                                                                                             |
| `enumerate(iterable)`                                           | ✅ Supported       | Constant sequences, range, fixed arrays, buffers and strings                                                                                                                                                                        |
| `zip(a, b)`                                                     | ✅ Supported       | Compile-time unroll over constant lists                                                                                                                                                                                             |
| `reversed(iterable)`                                            | ✅ Supported       | Compile-time reverse unroll                                                                                                                                                                                                         |
| `any` / `all` / `sum` / `min` / `max` on a generator expression | ✅ Supported       | Direct argument only, over a known-length iterable; no lazy generator object                                                                                                                                                        |
| `divmod(a, b)`                                                  | ✅ Supported       | Unpacked, bound to one name or printed; runtime zero raises; a pair whose quotient or remainder can exceed `int32` (`int32` by a divisor that can be -1, `uint32` remainders) refuses at compile time: no wider integer type exists |
| `pow(x, n)` / `x ** n` / `math.pow`                             | ✅ Supported       | Constant integer, runtime integer and runtime software-float forms                                                                                                                                                                  |
| `math.sqrt` / `exp` / `log` / `radians`                         | ✅ Supported       | Runtime software float, linked only when called                                                                                                                                                                                     |
| `hex(n)` / `bin(n)` / `oct(n)`                                  | ✅ Supported       | Flash string for constants, fixed runtime buffer otherwise                                                                                                                                                                          |
| `round(x[, n])`                                                 | ✅ Supported       | Half-to-even; `n` is compile-time                                                                                                                                                                                                   |
| `str(n)`                                                        | ✅ Supported       | Compile-time only                                                                                                                                                                                                                   |
| `ord('A')` / `chr(n)`                                           | ✅ Supported       | `ord` compile-time; runtime `chr` survives a character-returning function                                                                                                                                                           |
| `int.from_bytes(b, e)`                                          | ✅ Supported       | Compile-time fold or runtime                                                                                                                                                                                                        |
| `memoryview(buf)`                                               | ✅ Supported       | Fixed-buffer alias or sliced writable window; no runtime buffer protocol                                                                                                                                                            |
| `input(prompt?, maxlen?)`                                       | ✅ Supported       | `line: bytearray = input("prompt")` — reads a newline-terminated line from UART; the prompt is an optional compile-time string, the max length an optional integer (default 64); the UART preamble is auto-injected                 |
| `getattr(module, "name", default)`                              | ✅ Supported       | Compile-time module lookup with a literal attribute name                                                                                                                                                                            |
| `open()` / file I/O                                             | ✅ Read-only ROMFS | Compile-time path and mode; `read`, `readinto`, `readline`, `seek`, `tell`, `close`, `with`                                                                                                                                         |
| `sorted()`                                                      | ❌ Not supported   | No dynamic allocation                                                                                                                                                                                                               |
| `map()` / `filter()`                                            | ❌ Not supported   | Use explicit `for` loops                                                                                                                                                                                                            |
| `exec()` / `eval()`                                             | ❌ Not supported   | An interpreter would be required                                                                                                                                                                                                    |
| Runtime `getattr()` / `hasattr()`                               | ❌ Not supported   | No runtime type information                                                                                                                                                                                                         |

---

## Platform notes (AVR — ATmega328P / Arduino Uno)

- **Static call layout:** direct and mutual recursion are refused. Ordinary non-inline calls
  use the AVR ABI, but function storage is assigned statically rather than as recursive
  per-call frames.
- **Soft float:** `float` variables and arithmetic go through a pure-assembly soft-float
  library. No FPU required; expect ~200-400 cycles per operation.
- **No heap by default:** every variable must have a size known at compile time. The one
  opt-in exception is `list[T]`, which links a bounded bump allocator and a GC — and only
  into firmware that uses it. A once-only runtime-sized `bytearray(n)` uses a separate
  static arena with no `free()`.
- **Capacity is checked at build time, not at flash time:** an image larger than the chip's
  flash fails the build with the exact overage (`firmware is 32864 bytes but atmega328p has
32768 bytes of flash (96 bytes over)`), and static data that does not fit in SRAM fails in
  the backend with the same shape (`static data needs 2700 bytes but atmega328p has 2048
bytes of SRAM`). The SRAM check reserves 64 bytes for the hardware call stack, which grows
  down into the same space.
- **String literals live in flash:** read-only, sent to UART through the flash string pool.
  Compile-time text can be compared and indexed; a name selected from several flash strings
  can be printed and compared by interned id. The text itself cannot be modified.
- **Float text has a visible flash cost:** unformatted float output links the MicroPython-style
  float32 formatter lazily. Tiny parts with 4 KB of flash or less use a compact one-decimal
  fallback because the full formatter alone would consume most of the device.
- **Profile-guided optimization is experimental:** `pymcu build --profile` and
  `pymcu profile --pgo` require `pgo = true` under `[tool.pymcu.experimental]` or
  `PYMCU_EXPERIMENTAL_PGO=1`. The profile may keep hot inline regions from being outlined;
  the default build is unchanged.
- **C/C++ interop:** supported via `@extern` and `[tool.pymcu.ffi]` in `pyproject.toml`.
  C sources go through GCC's `cc1`, C++ sources (`.cpp` / `.cc` / `.cxx`) through `cc1plus`
  with `-fno-exceptions -fno-rtti`, which makes Arduino libraries usable. No extra to
  install: the front ends ride in the same WebAssembly toolchain wheel as the assembler and
  linker. One known limit, and it predates that wheel: a **namespace-scope C++ object with a
  constructor does not link**, because PyMCU links `-nostartfiles` with its own linker script
  and neither `__do_global_ctors` nor `__bss_start` / `__bss_end` is provided. It fails the
  same way on a native toolchain — it is a linker-script gap, not a compiler one.
- **Toolchain:** the assembler, linker and C/C++ front ends are `wasm32-wasip1` modules run
  through wasmtime, so there is one wheel for all platforms and firmware identical to the
  native toolchain's, verified by sha256 across 59 examples on five hosts. See
  [Installation](/getting-started/installation/#the-avr-toolchain-is-webassembly).

---

## Platform notes (ARM — RP2040 / RP2350)

The ARM backend does **not** emit assembly directly. It lowers PyMCU's
architecture-agnostic IR to **LLVM IR** — target `thumbv6m-none-eabi` for the RP2040
(Cortex-M0+) and the Cortex-M33 triple for the RP2350 — so LLVM handles register
allocation, instruction selection, the AAPCS calling convention and all optimization
passes. `pymcu build` emits a flat flash image (`firmware.bin`), with the stage-2 boot
loader at offset 0 on RP2040 and a picobin image block on RP2350.

Both chips ship the **full peripheral HAL** — GPIO, UART, SPI, I2C, PWM, ADC and DMA — plus
the `@rp2.asm_pio` PIO DSL. Exceptions (`try` / `except` / `raise` / `finally`) and `float`
work here exactly as they do on AVR. Interned strings and `const[uint8[N]]` tables are
flash-resident (`.rodata`), and `const[str]` runtime subscript works.

The CYW43439 WiFi stack (`pymcu.hal.wifi`: gSPI bring-up, WLAN join, TCP, MQTT publish) is
**Pico 2 W (RP2350) only** — importing it on any other chip, the Pico W (RP2040) included, is
a `CompileError`.

Remaining ARM-specific caveats:

- **Single core:** only core 0 runs. Dual-core launch and the SIO FIFO are not exposed yet.
- **Delays:** `delay_ms` / `delay_us` poll the hardware **TIMER** (the free-running 1 MHz
  microsecond counter), so timing is accurate on real silicon regardless of CPU clock and
  pipeline — it is not a calibrated busy-loop. In the emulator, wall-clock measured by
  `RunMilliseconds` reads the wait slightly short, because that harness budgets execution by
  retired instruction count while the timer advances by elapsed cycles; the firmware delay
  itself is exact.
- **WiFi is RP2350-only, and open networks only:** there is no CYW43 driver for the RP2040,
  and WPA / WPA2 is not implemented — `connect(ssid, key)` raises a `CompileError` if `key`
  is non-empty rather than silently dropping it.
- **UART clock assumption:** the baud divisors assume the board default for the chip —
  `clk_peri = 125 MHz` on the RP2040 and `150 MHz` on the RP2350 (`clk_peri == clk_sys` in
  both cases). A configurable clocks HAL is future work.
- **No C/C++ interop:** `@extern` and `[tool.pymcu.ffi]` are AVR-only.
- **Toolchain:** install with `pipx install --pip-args=--pre "pymcu-compiler[arm]"`. The
  backend ships in `pymcu-arm` and requires **LLVM** (`opt`, `llc`, `llvm-mc`, `ld.lld`,
  `llvm-objcopy`) on the host, provided by the
  [`pymcu-arm-toolchain`](https://github.com/PyMCU/pymcu-arm-toolchain) wheel. If no wheel
  is available for your platform, the toolchain falls back to a system LLVM (for example
  `brew install llvm lld`).

---

## Platform notes (PIC)

The PIC backend is **new in alpha 3** and currently covers the mid-range
**PIC16F84A** and **PIC16F877A**. It emits assembly directly (no LLVM) and supports:

- Software `*`, `//` and `%` routines for 8- and 16-bit operands (no hardware multiplier).
- Fixed-size RAM arrays with runtime indices (FSR / INDF addressing).
- A catchable `ZeroDivisionError` guard on runtime `//` and `%`.
- EUSART UART (baud divisors derived from `__FREQ__`), flash-resident strings and `print()`.

Caveats:

- **No general `try` / `except`:** full exception propagation is AVR + ARM only. On PIC,
  use return codes or sentinel values; only the `ZeroDivisionError` guard is emitted.
- **No `@extern` C interop.**
- **Toolchain:** install with `pipx install --pip-args=--pre "pymcu-compiler[pic]"`. It
  bundles self-contained gputils (`gpasm`) wheels — no system packages required.

---

## What stops each Adafruit CircuitPython library

This compatibility campaign compiled 37 unmodified upstream libraries for an Arduino Uno,
then constructed their objects and exercised methods from a separate entry program. The
measurements come from the 2026-09 Beta 1 campaign and the fixes it produced. A build is not
the same as a silicon run; Beta 1's real-board coverage is limited to the two libraries
listed in [State of the beta](/state-of-the-beta/#what-ran-on-real-hardware-for-this-release).

Twenty of the original 37-library set built unmodified at the recorded checkpoint, and
additional entries moved forward as later Beta 1 fixes landed. The table keeps the last
measured blocker rather than turning a refusal into a vague compatibility percentage.

| Library                             | Result or current blocker                                                                                  |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `adafruit_ahtx0`                    | Builds unmodified, 7,108 bytes                                                                             |
| `adafruit_ads1x15`                  | `next(key for key, value in ...)`: generator expression used as a lazy value                               |
| `adafruit_aw9523`                   | Builds unmodified, 2,294 bytes                                                                             |
| `adafruit_bme280`                   | Undefined `_bus_implementation.read_register` call                                                         |
| `adafruit_bmp280`                   | Builds unmodified, 25,006 bytes                                                                            |
| `adafruit_bus_device`               | Builds unmodified, 800 bytes; its simpletest builds at 1,444 bytes                                         |
| `adafruit_character_lcd`            | Runtime bit index in the wrapped `Pin.high()` path                                                         |
| `adafruit_debouncer`                | Passed object matches neither member of `Union[ROValueIO, Callable[[], bool]]`                             |
| `adafruit_dht`                      | Builds unmodified, 12,252 bytes, after tagged multi-member returns and fields                              |
| `adafruit_dps310`                   | Builds unmodified, 13,162 bytes                                                                            |
| `adafruit_ds18x20`                  | Missing `onewireio` module                                                                                 |
| `adafruit_ds3231`                   | `time.struct_time` exists in the core stub but is not exported by the CircuitPython overlay                |
| `adafruit_74hc595`                  | Builds unmodified, 402 bytes                                                                               |
| `adafruit_hcsr04`                   | Builds unmodified, 3,430 bytes; also ran on a real Uno for Beta 1                                          |
| `adafruit_ht16k33` matrix           | Builds unmodified; matrix simpletest 4,234 bytes                                                           |
| `adafruit_ht16k33` segments         | Builds unmodified, 5,710 bytes                                                                             |
| `adafruit_ina219`                   | Builds unmodified, 7,294 bytes                                                                             |
| `adafruit_irremote`                 | Tagged union payload has no valid member slot in this generator path                                       |
| `adafruit_lis3dh`                   | Builds unmodified, 2,522 bytes                                                                             |
| `adafruit_mcp230xx`                 | Runtime bit index in the wrapped `Pin.high()` path                                                         |
| `adafruit_mcp3xxx`                  | Builds unmodified, 3,094 bytes                                                                             |
| `adafruit_mcp9808`                  | Builds unmodified, 4,646 bytes                                                                             |
| `adafruit_mlx90614`                 | Builds unmodified, 3,846 bytes                                                                             |
| `neopixel` with `adafruit_pixelbuf` | Builds unmodified; GRB byte order verified in the AVR emulator                                             |
| `adafruit_pca9685`                  | Builds unmodified, 1,852 bytes                                                                             |
| `adafruit_pcf8523`                  | Same CircuitPython `time.struct_time` export gap as `adafruit_ds3231`                                      |
| `adafruit_pcf8574`                  | Builds unmodified, 1,442 bytes                                                                             |
| `adafruit_seesaw`                   | Builds unmodified, 3,706 bytes; emulator I2C stream matches CPython byte for byte                          |
| `adafruit_sht31d`                   | Moved past indexed `struct.unpack`; next construct was still being measured                                |
| `adafruit_sht4x`                    | Moved past field-buffer slicing; stops on iterating a buffer in static `_crc8`                             |
| `adafruit_si7021`                   | Moved past a quoted dotted class annotation; next construct was still being measured                       |
| `adafruit_ssd1306`                  | 128x32 simpletest builds unmodified at 4,348 bytes and matches CPython I2C traffic; also ran on a real Uno |
| `adafruit_tcs34725`                 | Builds unmodified, 28,414 bytes                                                                            |
| `adafruit_tmp117`                   | Imported descriptor context manager cannot resolve `__enter__`                                             |
| `adafruit_tsl2591`                  | Builds unmodified, 5,814 bytes                                                                             |
| `adafruit_veml7700`                 | Builds unmodified, 10,614 bytes                                                                            |
| `adafruit_motor` servo              | Builds unmodified, 2,332 bytes                                                                             |

Several compiler features on this page came directly from reducing those failures:
compile-time `**kwargs`, bounded exception objects, structural Protocol members in inline
unions, buffer annotations, tuple-returning properties, descriptors, package re-exports,
ROMFS font access, `memoryview` windows, runtime float `pow`, fixed tables of rows,
generator reductions, and tagged scalar unions.

The remaining entries are a mix of deliberate fixed-footprint limits and compiler gaps. The
important guarantee is diagnostic quality: a refusal must name the construct at the source
line, not report an internal compiler error or a missing bracket. A clean build is still not
a hardware claim unless it appears in the real-silicon section linked above.

---

## Getting help

If you hit a compile error on a Python construct not covered here, please
[open an issue](https://github.com/PyMCU/PyMCU/issues). Include the source snippet and the
compiler error message — the compiler reports `file:line` for user-facing errors, so paste
the whole diagnostic.

For what is coming next, see the [roadmap](/roadmap/).
