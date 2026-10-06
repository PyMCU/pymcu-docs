---
title: Language oracle
description: How PyMCU compares compiled AVR programs against CPython, records deliberate divergences and pins known compiler bugs.
---

The language oracle lives in the
[`pymcu-avr` repository](https://github.com/PyMCU/pymcu-avr/tree/main/tests/oracle) because
it needs the AVR backend and Avr8Sharp emulator as well as the compiler. This page describes
the probe contract, how known divergences stay visible, and what the major sweeps found.

Each probe is a small top-level Python program. The harness runs it directly with CPython,
then compiles it with PyMCU, executes the firmware in the Avr8Sharp `ArduinoUno` emulator,
captures UART0 until `END`, and compares the two outputs line by line. Pytest creates one
test per probe and runs the corpus under both compiler frontends.

## Probe headers

Each file under `tests/oracle/probes/` declares what it is intended to prove:

- `# expect: match`: CPython and emulated firmware must produce byte-identical output.
- `# expect: refuse <substring>`: compilation must fail and the diagnostic must contain
  the given text.
- `# expect: divergence <doc citation>`: PyMCU deliberately differs from CPython and the
  cited documentation explains the difference. The harness transforms CPython's output by
  that one registered rule, then compares again. An unknown citation is a hard error.
- `# doc: <file>:<line>`: identifies the reference or limitation being exercised. A filed,
  not-yet-documented gap may cite its issue URL instead.
- `# tracked: #<N>`: marks a filed compiler bug. Pytest uses `xfail(strict=True)`, so the
  suite stays green while the known failure remains, but a fix becomes a hard XPASS until
  the tracking marker and expected result are updated.
- `# frontend: default` or `# frontend: py-parser`: restricts a probe to one parser when a
  filed frontend disagreement makes one expectation impossible for both. Such cases use a
  pair of probes, one per frontend.
- `# python: >=X.Y`: skips syntax that the selected Python interpreter cannot parse yet.

A probe is corrected when the probe itself is wrong, such as a stale import or a refusal
substring in the wrong case. A genuine mismatch is not edited away. It is filed, left in
place, and marked `# tracked`.

## Running the oracle

From a `pymcu-avr` checkout with a sibling PyMCU checkout and both virtual environments
installed:

```sh
dotnet build tests/oracle/runner/PyMCU.OracleRunner.csproj -c Release -o build/oracle
.venv/bin/python -m pytest tests/oracle -q
PYMCU_PY_PARSER=1 .venv/bin/python -m pytest tests/oracle -q
```

`PYMCU_BIN` defaults to `.venv/bin/pymcu`. `PYMCU_BACKEND_BINARY` and
`PYMCU_ORACLE_RUNNER` can override the backend executable and runner DLL. Every probe that
is not frontend-specific is expected to stay green under both frontends.

## Beta 1 release result

The final Beta 1 measurement used 440 probes. Of 426 runnable cases, the C# frontend passed
415 and tracked 11 known failures; the Python frontend passed 412 and tracked 14. Both
skipped 14 cases restricted to the other frontend or interpreter version. See
[State of the beta](/state-of-the-beta/#what-the-oracle-knows-is-wrong) for the release's
issue table.

## Grammar and context sweep, 2026-09-26

The last full sweep documented in the compiler repository had 368 probes: 280 matches, 84
refusals and 4 documented divergences. The default frontend reported 338 passes, 14 skips
and 16 tracked failures; the Python frontend reported 335 passes, 14 skips and 19 tracked
failures.

Fifty new probes came from two closed enumerations instead of hand-picking features:

- Every concrete AST node class CPython can produce. Coverage moved from 83 of 99 reachable
  classes to 109 of 119; the remaining ten are deprecated stubs or roots for parse modes a
  probe file cannot use.
- Construct by context: module, function, method, `@inline` and `@inline` method, crossed
  with value, condition, argument, return, field, index, f-string expression and ternary
  positions.

The context axis found the bugs. Most were not a broken construct, but a construct lowered
through two internal paths where only one path was correct. Every effective probe therefore
needs a runtime seed that blocks constant folding and a value that changes when the
construct breaks.

## Language-surface sweep, 2026-09-15

The previous run had 178 probes after adding 58 cases from a Python language-reference
sweep. Both frontends were green with tracked bugs represented as strict expected failures.
The main causes found were:

| Cause | Issue | Representative probes |
|---|---|---|
| Unannotated integer arithmetic kept the width of its first store and wrapped | [#364](https://github.com/PyMCU/PyMCU/issues/364) | `007`, `051`, `073` |
| A field reached through `with obj as name` read zero | [#390](https://github.com/PyMCU/PyMCU/issues/390) | `029`, `030` |
| A class without explicit `__init__` could not be constructed | [#391](https://github.com/PyMCU/PyMCU/issues/391) | `037`, `079`, `080` |
| `self.field = bytearray(...)` was refused while an identical local worked | [#392](https://github.com/PyMCU/PyMCU/issues/392) | `049` |
| Folded `hex`, `bin` and `str` printed a flash address | [#393](https://github.com/PyMCU/PyMCU/issues/393) | `056` |
| Multi-clause and filtered comprehensions were wrong or unconditionally refused | [#394](https://github.com/PyMCU/PyMCU/issues/394) | `063` to `065` |
| ZCA arithmetic and comparison dunders returned wrong results in selected contexts | [#395](https://github.com/PyMCU/PyMCU/issues/395) | `075`, `146` |
| `len(instance)` returned zero although implicit truthiness found `__len__` | [#396](https://github.com/PyMCU/PyMCU/issues/396) | `076` |
| Two-index `__setitem__` and `__getitem__` returned zero | [#397](https://github.com/PyMCU/PyMCU/issues/397) | `078` |
| `list[T].append()` did not store its element | [#398](https://github.com/PyMCU/PyMCU/issues/398) | `084`, `153` |
| A one-character runtime-string subscript produced its numeric code | [#399](https://github.com/PyMCU/PyMCU/issues/399) | `087` |
| An enum read produced a misleading undefined-class error | [#400](https://github.com/PyMCU/PyMCU/issues/400) | `027` |
| A sequence pattern over a real array read phantom flattened variables | [#401](https://github.com/PyMCU/PyMCU/issues/401) | `018` |
| `chr()` lost its character identity across a function return | [#436](https://github.com/PyMCU/PyMCU/issues/436) | `127` |
| String concatenation and equality fell through to interned numeric ids | [#438](https://github.com/PyMCU/PyMCU/issues/438) | `134`, `135` |
| A tuple pattern was refused by one frontend and silently wrong in the other | [#439](https://github.com/PyMCU/PyMCU/issues/439) | `168`, `169` |
| Class patterns differed between frontends | [#440](https://github.com/PyMCU/PyMCU/issues/440) | `170`, `171` |
| A type annotation reached through a module alias wrapped instead of promoting | [#449](https://github.com/PyMCU/PyMCU/issues/449) | `172` |

Several of these were fixed and untracked before Beta 1. The table is historical: the
current `# tracked` headers, not this snapshot, are authoritative.

## Probe defects corrected during the sweep

Nine probes were repaired because they tested something other than their header claimed:

- The sequence-pattern probe was rewritten from a runtime tuple, which hit the separate
  runtime-tuple limit, to a real array. That correction exposed #401.
- Two probes imported `inline` from `pymcu` instead of `pymcu.types`.
- A buffer-slice repr probe used a CPython list on one side and a PyMCU byte buffer on the
  other; both now use `bytearray`.
- A float-format probe iterated a float list, which was not part of the claimed iterable
  surface; it now prints the values directly.
- The Optional probe now uses the documented guarded `typing` import.
- Dict, set and generator-expression refusal probes now match the compiler's actual
  diagnostics and parse stage.

## What the oracle does not cover

- ARM, PIC and RISC-V backends. The runner executes only AVR firmware in Avr8Sharp.
- HAL and driver behavior. Peripheral APIs need hardware or peripheral-aware emulation, not
  only captured UART.
- Mutable globals across multiple source modules. A probe currently creates one
  `src/main.py`.
- Allocator exhaustion where no independent probe can yet reach the required heap state.

These gaps are why the beta also uses API parity, user-program corpus, integration and
real-silicon suites.

## Regression sweep after the repository move

When the corpus moved to `pymcu-avr`, six probes had gone red while it was temporarily not
running on `main`:

- String-list and pair-list loops printed interned ids. Folding had run before literal
  strings were recognized.
- A runtime `join` buffer was confused with a same-named module variable inside
  `uart_write_str`, repeating its output.
- A string-concatenation probe became a strict XPASS because its bug had been fixed.
- String membership and instance `__contains__` probes still expected refusal after both
  features had shipped.

The move added a permanent `just test-oracle` gate that builds the runner and executes both
frontends.

## Comparison dunders and construction hooks

The 2026-09-25 sweep added probes `281` through `291` for
[#491](https://github.com/PyMCU/PyMCU/issues/491). They pin all six comparison dunders in
conditions and value positions, module-level and class-field receivers, identity fallback
for classes without comparison dunders, and correct refusal of unsupported ordering,
`min` / `max`, `__new__` and `__init_subclass__`. A `__del__` probe records the deliberate
divergence that static storage has no destruction event.
