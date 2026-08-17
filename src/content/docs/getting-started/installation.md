---
title: Installation
description: Install the PyMCU compiler and the toolchain for your target — AVR, ARM (RP2040 / RP2350) or PIC.
---

PyMCU ships as a single PyPI package, `pymcu-compiler`, plus one extra per architecture.
Each extra pulls in its own backend **and** its own assembler and linker, so there is
nothing to install from your system package manager.

:::caution[Alpha release]
The current release is **v0.1.0a8**. `pip`, `uv` and `pipx` skip pre-releases unless you ask
for them, so every command on this page carries a `--pre` flag. Once `0.1.0` stable ships,
`--pre` will no longer be needed.
:::

## Requirements

- **Python 3.11 or newer** — [download it from python.org](https://www.python.org/downloads/)
- One installer: [**pipx**](https://pipx.pypa.io/stable/how-to/install-pipx.html) (recommended),
  [**uv**](https://docs.astral.sh/uv/getting-started/installation/) or
  [**pip**](https://pip.pypa.io/en/stable/installation/)
- A **supported platform**: Linux x86-64 or arm64, macOS on Apple Silicon, or Windows on
  x64 or arm64. Those are the platforms PyMCU publishes wheels for; Intel Macs are not
  among them — see [below](#macos-on-intel).

That is the whole list **to compile**: each extra brings its own assembler and linker, so
there is no `avr-gcc`, no ARM GNU toolchain and no MPLAB to install. On AVR they are not
even native binaries — [the toolchain is WebAssembly](#the-avr-toolchain-is-webassembly).
(The one exception is ARM on a platform with no prebuilt toolchain wheel, noted below.)

**Flashing** is the one step that reaches outside PyMCU. Each target needs its own uploader
on your machine — `avrdude` for AVR, nothing at all for a Pico in bootloader mode, `pk2cmd`
for PIC — and the [per-target notes](#per-target-notes) below say exactly what to do for
yours.

:::note[Starting from a machine with nothing installed?]
On **macOS**, typing `python3` for the first time opens a dialog offering to install the
Xcode Command Line Tools. Accept it, or install Python from python.org instead; either
gets you a working `python3`. `pipx` is not preinstalled.

On **Windows**, install Python from python.org or the Microsoft Store, and make sure
"Add python.exe to PATH" is ticked.

On **Linux**, the distribution's `python3` is usually recent enough, but `pipx` normally
comes as its own package.

After installing `pipx`, run `pipx ensurepath` and open a new terminal so the `pymcu`
command is found.
:::

## Install with pipx (recommended)

```bash
pipx install --pip-args=--pre "pymcu-compiler[avr]"    # AVR (ATmega / ATtiny)
pipx install --pip-args=--pre "pymcu-compiler[arm]"    # RP2040 / RP2350 (Pico / Pico 2)
pipx install --pip-args=--pre "pymcu-compiler[pic]"    # PIC16
pipx install --pip-args=--pre "pymcu-compiler[all]"    # everything
```

`pipx` installs into an isolated environment and puts the `pymcu` command on your PATH.
If the command is not found afterwards, run `pipx ensurepath` and open a new shell.

## Install with uv

```bash
uv tool install --pre "pymcu-compiler[avr]"
uv tool install --pre "pymcu-compiler[arm]"
uv tool install --pre "pymcu-compiler[pic]"
uv tool install --pre "pymcu-compiler[all]"
```

`uv tool install` also places `pymcu` on your PATH globally, isolated in its own virtual
environment — no activation step. Requires **uv 0.11 or newer** (`uv self update`).

## Install with pip

```bash
pip install --pre "pymcu-compiler[avr]"
```

Use this inside a virtual environment you manage yourself, or when adding PyMCU to an
existing project's dependencies.

:::caution[Not into the system Python]
On Debian/Ubuntu 23.04 and newer, and on Homebrew Python, a plain `pip install` outside a
virtual environment fails with `error: externally-managed-environment`. That is the
distribution refusing to let pip write into the system interpreter, not a PyMCU problem —
use `pipx` (above) for a global `pymcu` command, or create a virtual environment first.
:::

## Verify

```bash
pymcu version
```

`pymcu --version` prints the same table.

This prints a small table rather than a single version string:

```text
        PyMCU Ecosystem Version Info
+----------------+-----------------------+----------+
| Package        | Description           | Version  |
+----------------+-----------------------+----------+
| pymcu-compiler | Compiler & CLI Driver | 0.1.0a8  |
| pymcu-stdlib   | Standard Library      | ...      |
| python         | Python Interpreter    | 3.11+    |
+----------------+-----------------------+----------+
```

Anything missing shows as **Not Installed** instead of being left out, so this is also the
quickest way to check that the standard library is on the path.

:::note[Package name]
PyMCU is published as `pymcu-compiler` while a PEP 541 request to reclaim the `pymcu` name
is under review. Once approved, a `pymcu` metapackage will alias `pymcu-compiler` — installs
and project configs will stay compatible.
:::

## Compat packages

Two optional packages give you the MicroPython and CircuitPython APIs. **During the alpha
these are the recommended way to write firmware** — they are stable and community-specified,
while the native `pymcu.hal.*` API may still change between releases.

| Package | What it gives you |
|---|---|
| `pymcu-micropython` | `machine` (Pin, UART, ADC, PWM, SPI, I2C, Timer, WDT), `utime` |
| `pymcu-circuitpython` | `board`, `digitalio`, `analogio`, `busio`, `pwmio`, `time`, `neopixel` |

**Do not install these globally.** They are not tools you run: they are *project*
dependencies that the compiler reads when it builds your firmware, so they belong to the
project's environment, not to your interpreter. Installed anywhere else they do nothing.

You do not normally install them by hand at all. `pymcu new` asks which API you want, writes
the matching package into your `pyproject.toml`, and offers to install it:

```bash
pymcu new blink --stdlib micropython
```

To add one to a project that already exists, put it in `[project] dependencies` in
`pyproject.toml` and install with whatever manages that project (`uv sync`, `poetry install`,
or `pip install -e .` inside its virtual environment). The [Quick Start](/getting-started/quickstart/)
walks through a project from scratch, and the [CLI Driver](/driver/#pymcu-new-name) page
documents every `pymcu new` flag.

## macOS on Intel

**There is no macOS x86-64 wheel, so `pip` finds no candidate and the install stops.** That
is the whole of it today: `pymcu-compiler` and `pymcu-avr` are published for Linux x64 and
arm64, macOS arm64, and Windows x64 and arm64, and macOS Intel is not on the list.

The reason it left the list has since gone away. The blocker used to be the AVR toolchain: a
native `avr-gcc` bundle that existed only for Apple Silicon, and building an Intel one meant
compiling GCC for hardware Apple has stopped shipping. That argument is spent — the AVR
toolchain is now [architecture-independent WebAssembly](#the-avr-toolchain-is-webassembly),
and `wasmtime` does publish a `macosx_10_13_x86_64` wheel. What remains is that PyMCU's own
wheels, dropped while the toolchain was the obstacle, have not come back.

Whether it would then work is a **reasoned expectation, not a measurement**: GitHub retired
the `macos-13` runner, so there is nowhere to verify it. Nothing in the current design points
against it and nobody has run it.

Meanwhile, the practical options for an Intel Mac are a Linux machine, a Linux virtual
machine, or WSL on Windows. All three are supported.

## Per-target notes

### AVR

#### The AVR toolchain is WebAssembly

`avr-as`, `avr-ld`, `avr-objcopy` and the C/C++ front ends (`cc1`, `cc1plus`) ship as
`wasm32-wasip1` modules in **`pymcu-avr-toolchain-wasi`**, and `pymcu build` runs them
in-process through [wasmtime](https://wasmtime.dev/). It is a single **22.9 MB
`py3-none-any` wheel for every platform**, where the native toolchain was five separate
builds of 70.7 MB each. `wasmtime` is the only piece left that carries per-platform
binaries, and PyMCU asks for `wasmtime>=20`.

**The firmware is unchanged.** All 59 AVR examples — 53 plain plus 6 going through the C/C++
path, one of them C++ — were compared against the native toolchain's output **by sha256**,
not merely by whether the build succeeded: 59 of 59 identical on Linux x64 and arm64, macOS
arm64, and Windows x64 and arm64, from the same modules. It is also faster in ordinary use
(0.27 s against 0.51 s for 53 builds), because the modules are compiled once per process
rather than a tool being forked per stage.

Three things follow for you:

- **Rosetta 2 is no longer needed.** The old bundled `avr-gcc` was an x86_64 PlatformIO
  build, so on Apple Silicon it ran under Rosetta and failed with `bad CPU type in
  executable` on a Mac that had never installed it. That step is gone.
- **C and C++ interop needs no extra.** `cc1` and `cc1plus` travel in the same wheel, so
  `@extern` and `[tool.pymcu.ffi]` work out of the box.
- **`PYMCU_AVR_WASI=0`** forces the old native path, for anyone who has a native toolchain
  installed and wants to compare against it.

**The first build on a machine is slower**, and says so while it happens. wasmtime compiles
the modules to native code once and caches the result, keyed by your OS, CPU and wasmtime
version; after that a build reuses it. The measured figures are **0.82 s for a first build and
0.23 s for later ones**, with 6.2 MB of cache on disk. Nothing is downloaded at build time —
the wheel already carries everything, and the wait is compilation.

Everything needed to *compile* is bundled. Flashing is the part WASI cannot do — `avrdude`
talks to a serial port — so to *flash* an Arduino Uno you still need **avrdude** on your host:

```bash
brew install avrdude          # macOS
sudo apt-get install avrdude  # Debian / Ubuntu
```

On Windows, grab a build from the
[avrdude releases page](https://github.com/avrdudes/avrdude/releases).

Then:

```bash
pymcu flash --port /dev/cu.usbmodem*   # macOS
pymcu flash --port /dev/ttyACM0        # Linux
pymcu flash --port COM3                # Windows
```

### ARM (RP2040 / RP2350)

The `[arm]` extra lowers PyMCU's IR to **LLVM IR** and drives an LLVM toolchain
(`opt` → `llc` → `ld.lld` → `llvm-objcopy`). On Linux x64/arm64, Windows x64 and macOS
arm64 the prebuilt `pymcu-arm-toolchain` wheel is pulled in automatically. On any other
platform, install a system LLVM instead:

```bash
brew install llvm lld        # macOS
sudo apt install llvm lld    # Debian / Ubuntu
```

A build produces a flat `dist/firmware.bin`. Flashing a Pico is the usual bootloader dance:
hold **BOOTSEL** while plugging the board in, and it mounts as a USB mass-storage device —
then drop the UF2 onto it, or let the driver do it:

```bash
pymcu flash
```

### PIC

The `[pic]` extra bundles **gputils** (`gpasm`) as a wheel for every supported platform, so
there is nothing else to install. `pymcu flash` drives a PICkit 2 by default (`pk2cmd` is
fetched on first use).

## Build from source

For contributors, or anyone who wants the latest development build.

**Requirements**

- [.NET 10 SDK](https://dotnet.microsoft.com/download)
- Python 3.11 or newer
- [`uv`](https://docs.astral.sh/uv/) and [`just`](https://github.com/casey/just)

**Steps**

```bash
git clone https://github.com/PyMCU/PyMCU.git
cd PyMCU

# Build the C# compiler (.NET 10, AOT)
dotnet publish src/compiler/PyMCU.csproj -c Release -o build/bin --nologo

# Set up the Python environment
uv venv && source .venv/bin/activate
uv sync
just sync-stdlib      # editable stdlib: uv pip install --no-deps -e lib/
pip install -e src/driver
```

`just sync-stdlib` only has to run once — after that, edits under `lib/src/pymcu/` are picked
up live.

:::danger[Never copy the stdlib into site-packages]
Copying `lib/src/pymcu/` into `.venv/.../site-packages/pymcu/` shadows the editable `.pth`
install, and your edits silently stop taking effect — with no error to tell you why. If
stdlib changes seem to be ignored, check `pymcu.__file__` and delete any physical copy you
find in site-packages.
:::

Verify:

```bash
pymcu --version
```

Backends live in their own repositories — [`PyMCU/pymcu-avr`](https://github.com/PyMCU/pymcu-avr),
[`PyMCU/pymcu-arm`](https://github.com/PyMCU/pymcu-arm) and
[`PyMCU/pymcu-pic`](https://github.com/PyMCU/pymcu-pic) — see [Contributing](/contributing/)
for the full layout.

## Next steps

:::tip[Installed? Go blink an LED]
The [**Quick Start**](/getting-started/quickstart/) takes you from an empty directory to a
blinking Arduino Uno, and shows the assembly your Python turned into along the way. It is
about five minutes, and it is the fastest way to find out whether PyMCU fits what you want
to build.
:::

- [Supported Targets](/targets/) — chips, peripherals and per-architecture capabilities
