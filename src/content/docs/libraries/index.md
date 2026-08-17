---
title: Using libraries
description: "Find, install and audit third-party PyMCU libraries — pymcu search, install, libraries and uninstall, plus how to read the curated index and what its ok / unsupported / failed labels mean."
---

A PyMCU library is **source code the compiler reads at build time**, not a module imported at
runtime. There is no interpreter on the device: a library's `.py` files are parsed by `pymcuc`
and compiled into your firmware alongside your own code, and what survives dead-code
elimination is what ends up on the chip.

Two consequences shape every command on this page. A library is always a **project
dependency**, never a global install — the compiler looks inside the project's environment, so
one installed anywhere else does nothing. And whether it works for you is a question about
**your chip**, not about the library in the abstract, which is why the index measures rather
than asks.

## Find one

```bash
pymcu search              # the whole catalogue
pymcu search sensor       # by name, summary or category
```

Run inside a project, results are filtered to what fits that project's chip and declared
layer, and the footer says how many were hidden; `--all` shows those too. The catalogue is new
and short — `pymcu search` with no query is the honest answer to what is in it today.

## Install one

```bash
cd my_project
pymcu install dht
```

Nothing is downloaded until the target has been checked. The name is resolved against the
index, the index's measurements for your chip and layer are consulted, and a library that
would not build is refused **before** the download rather than after. Then the requirement
goes into `[project] dependencies` and the install is handed to `uv add` or the project's
`pip`, whichever that project already uses.

By default the install is verified: a small program importing the modules the library declares
is compiled for your chip. If that fails, the install is **rolled back** — package removed,
dependency unwritten — so a failed verify leaves the project exactly as it was. `--no-verify`
skips it.

On success you get the import line, the compat adapter if one applies to your layer, and the
flash figure the index measured on your chip:

```text
+ dht 0.1.0 installed.
  import with: from dht import ...
  measured on atmega328p: 1014 bytes flash
```

A distribution that is not in the index can still be installed by its **distribution** name:

```bash
pymcu install --from-pypi pymcu-lib-something
```

That skips resolution, not validation — the package must still carry a valid manifest, and the
target is still checked.

## Audit what a project has

```bash
pymcu libraries           # version, modules, and whether each is usable here
pymcu libraries --all     # including the ones that do not fit this target
pymcu uninstall dht
```

`pymcu libraries` reports two things a plain package list cannot. **Collisions** — two
libraries claiming the same top-level module name, which is a resolution error rather than a
silent shadowing, because the compiler's include path is flat and shared. And **invalid
libraries** — a package whose manifest cannot be read is named and left out of the include
path instead of taking down the build of a project that never imported it.

Full flags for all four commands are in the [CLI reference](/driver/#pymcu-search).

## Reading the index

The index is a measurement, not a directory listing. Every entry is produced by installing the
library and **compiling its example for one chip per architecture** — `atmega328p`, `rp2040`
and `pic16f877a` — including architectures the author never declared, because that is the only
way to tell "does not support ARM" from "nobody updated `supports.arch`". The flash figure
beside an entry comes from that build.

Each chip carries one of four labels, and the difference between the middle two is the whole
point:

| Label | What it means |
|---|---|
| `ok` | The example compiled for that chip. The flash figure is measured |
| `unsupported` | It did not compile on a chip the **author never declared**. This is not a defect — the manifest already said the library does not go there |
| `failed` | It did not compile on a chip the author **did** declare. This is a defect: a promise the code does not keep |
| `unmeasured` | The build never ran, because the machine doing the measuring had no backend for that chip. It says something about that machine and nothing about the library |

`unsupported` exists because the label used to be `failed` either way, and a catalogue that
prints `failed` next to a library used outside its own declared scope is making an accusation
the manifest had already answered. Three rules keep that from becoming an excuse:

- A **declared** architecture that does not build is still `failed`.
- **Declaring nothing claims everything** — silence in a manifest is not a way out of being
  measured.
- A specific **chip list wins** over the architecture, for a library that supports one part
  rather than a family.

At the level of the whole entry, `status` is `active` when the library builds somewhere and
`broken` when it builds nowhere.

`pymcu install` and `pymcu search` both read these labels. Anything other than `ok` on your
chip is a reason not to serve you, so `install` refuses and `search` hides it unless you pass
`--all`; the layer the library is written against and the language level it needs are checked
the same way. The refusal quotes the reason, in the words you need to act on:

```text
'dht' does not fit this project: it measured as 'unsupported' on rp2040
```

### Where it lives, and how fresh it is

The catalogue is a single JSON document at
[`libraries.pymcu.org/index.json`](https://libraries.pymcu.org/index.json), with a mirror at
`raw.githubusercontent.com/PyMCU/pymcu-libraries/main/index.json`. The driver tries the first
and falls back to the second: the `pymcu.org` zone answers 403 to requests from data centres,
and `pymcu install` has to work inside other people's CI. `PYMCU_LIBRARY_INDEX` overrides
both, which is also how you point the driver at a local file while testing.

Downloads are cached under `~/.pymcu/`, keyed by the source they came from; `--refresh` fetches
again, and any command falling back to the cache says so.

It is regenerated **weekly**, not only when something is submitted: every listed library is
re-installed at its newest version and measured against the current compiler. So an entry
describes what builds today, and a library that stops building against a new compiler is
marked without anyone filing an issue. Each entry records the compiler version and the date it
was measured.

## Next

- [Writing a PyMCU library](/libraries/authoring/) — layout, manifest, architecture dispatch
  and publishing
- [CLI reference](/driver/#pymcu-search) — every flag on `search`, `install`, `uninstall` and
  `libraries`
- [Standard Library](/stdlib/) — what ships with PyMCU and needs no install
