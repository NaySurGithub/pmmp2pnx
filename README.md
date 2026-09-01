<p align="center">
  <b>PMMP2PNX</b>
  <br>
  <b>PocketMine-MP to PowerNukkitX plugin converter, written in C11</b>
</p>

> **This project was written entirely by AI.** Every line of C in this repository was produced by
> Fable 5, Opus 5 and Sol 5.6. No human wrote the implementation.
>
> Total cost to date: **$1,758** in tokens.

PMMP2PNX takes a PocketMine-MP plugin - a source directory or a distributed `.phar` - and produces a
complete Gradle project that compiles into a PowerNukkitX plugin JAR.

It is not a transpiler that stops at syntax. The generated project ships a small Java runtime that
reproduces PHP semantics (arrays, closures, loose comparison, string functions), plus adapters for the
libraries PocketMine plugins commonly bundle, so the output is meant to actually load and run.

## Building

Requirements: CMake 3.20+ and a C11 compiler. No external dependencies - the PHAR reader and its
DEFLATE decompressor are implemented in-tree.

```powershell
cmake -S . -B build
cmake --build build --config Release
```

The executable lands in `build/bin`.

## Usage

```text
pmmp2pnx
pmmp2pnx help
pmmp2pnx version
pmmp2pnx convert <plugin> [output]
pmmp2pnx convert <plugin> [output] --force
pmmp2pnx convert <plugin> [output] --pnx C:\path\to\PowerNukkitX
```

`<plugin>` is either a plugin source directory or a `.phar` file. Archives are extracted to a staging
directory before conversion; per-entry zlib compression and plugins that nest everything under a single
top-level folder are both handled.

`--pnx` points at a PowerNukkitX checkout or JAR so the generated Gradle project can compile against it.

Building the result:

```powershell
C:\path\to\PowerNukkitX\gradlew.bat -p <output> build --no-daemon
```

## What gets generated

```text
<output>/
  build.gradle.kts            Java 21 project targeting PowerNukkitX API 3.0.0
  src/main/resources/         plugin.yml, configs, language files, icon
  src/main/java/              translated plugin classes
    converted/runtime/        PHP runtime: PhpArray, PhpCallable, ~150 standard functions,
                              SQLite/MySQL drivers, promises, scheduler glue
    converted/runtime/world/  PocketMine world API compatibility layer (51 classes)
  CONVERSION_REPORT.txt       per-class log, plus anything needing review
```

## Status and known limitations

Read this section before trusting the output.

**Validation is narrow.** The converter is exercised against a small set of reference plugins. Two of
them - an economy plugin (80 PHP files) and a coinflip minigame (20 files) - have been taken end to end:
converted, compiled, loaded under PowerNukkitX with no exceptions, commands registered, SQLite schema
created, and individual methods verified against their PHP behaviour. A third, a world-management
plugin of 169 files, converts and is being brought to a compiling state.

**That is a small sample, and it shows.** Every new plugin has surfaced entirely new classes of bugs.
String interpolation was silently emitted as a literal - a fundamental PHP feature, wrong in every
converted plugin, and it produced no compile error at all; it went unnoticed through two complete
plugins. C-style `for` loops declared none of their variables. Assume plugin number four will surface
something comparable.

**Type inference is heuristic.** PHP is dynamically typed and Java is not. Where a type cannot be
derived from a declaration, a hoisted variable, or already-generated code, the converter falls back to
`Object`, which usually surfaces as a compile error rather than silent breakage - but not always.

**Review the report.** `CONVERSION_REPORT.txt` lists every class and flags method bodies that could not
be translated. A flagged body is emitted as a safe default (`return 0`, `return null`) and will compile
while doing nothing useful.

The most valuable contribution to this project is not a feature - it is a regression corpus. A dozen
varied plugins, converted and compiled on every change, would catch the class of bug described above
before it ships.

## Project layout

```text
include/pmmp2pnx/    public headers
src/main.c           entry point
src/cli/             banner, commands, interactive prompt
src/core/            application and console bootstrap
src/archive/         PHAR reader and DEFLATE decompressor
src/converter/       manifest, PHP parser, Java project generator, emitted runtime
src/translator/      expressions, statements, types, constants, imports
src/generator/       library adapters and the world compatibility layer
src/resolver/        Poggit dependency resolution
```

## Licensing of converted output

The generated Java is a derivative work of the plugin you feed in. Converting a plugin does not change
its licence, and redistributing the resulting JAR is subject to the original author's terms. The
compatibility layers mirror PocketMine-MP and PowerNukkitX APIs; both projects are LGPL, and any data
transcribed from them stays under their licence.
