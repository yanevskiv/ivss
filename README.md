<div align="center">

![ivss](docs/images/splash.jpg)


**An assembler, linker, and emulator for a hypothetical 32-bit CPU.**

[![Platform: Linux](https://img.shields.io/badge/Platform-Linux-black)](#building)
[![Architecture specification](https://img.shields.io/badge/Specification-PDF-red)](projekat-postavka-v1.0.pdf)
[![Language: C](https://img.shields.io/badge/Language-C-green)](src/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

</div>

# ivss (Ivan's System Software)

**ivss** is a system software project that follows an assembly program from source
code to execution. The assembler produces object files, the linker resolves
symbols and places sections in memory, and the emulator runs the resulting
program on a hypothetical 32-bit CPU.

The examples include interrupt handlers, timer interrupts, and terminal I/O.
They show how the three tools work together, from placing the first instruction
at the CPU's starting address to installing a handler for device interrupts.
The [project and architecture specification](projekat-postavka-v1.0.pdf) is in Serbian language.

| Tool | Responsibility |
| --- | --- |
| `asembler` | Parse assembly source and produce object files |
| `linker` | Combine sections, resolve symbols, apply relocations, and assign addresses |
| `emulator` | Execute instructions, with optional timer and terminal devices |

## Building

Build on Linux with GCC, GNU Make, and POSIX threads support:

```sh
make
```

This creates `asembler`, `linker`, and `emulator` in the project root, with
intermediate objects in `dist/`. No parser generator is required. The assembler
is spelled **`asembler`**, with one `s`, as defined in the Makefile.

Run the commands below from the project root. To remove the tools and
intermediate objects, run `make clean`.

## Quick start

Save this program as `hello.s`:

```asm
.section text
    ld $2, %r1
    ld $3, %r2
    add %r1, %r2
    halt
.end
```

Assemble it, place its code at the emulator's starting address, and run:

```sh
./asembler -hex -o hello.o hello.s
./linker -hex -place=text@0x40000000 -o hello.hex hello.o
./emulator hello.hex
```

After `halt`, the emulator prints its final register state. Here, `r1` contains
`0x00000002` and `r2` contains `0x00000005`.

Use **`-hex` for both assembly and linking** in this pipeline. Despite the `.o`
extension, the intermediate file contains a textual hexadecimal encoding of ELF
data. Without `-hex`, the tools write a readable section, symbol, and relocation
dump for inspection; that dump is not pipeline input.

## Assembler

```sh
./asembler [-hex] -o OUTPUT INPUT.s
```

The assembler accepts one source file and requires an explicit output. Use `-`
for standard input, or `-o -` for standard output.

Source uses labels, `#` comments, `%` register names, and `$` immediate values:

```asm
ld $42, %r1         # Load an immediate value
ld value, %r2       # Load a word from memory
st %r1, value       # Store a word to memory
add %r1, %r2        # Add r1 to r2
```

Register operands include `%r0` through `%r15`, with `%sp` and `%pc` aliases.
Control registers are `%status`, `%handler`, and `%cause`. Memory operands also
support register addressing, such as `[%r1]` and `[%r1 + 4]`.

<details>
<summary>Instruction reference</summary>

| Instruction | Operation |
| --- | --- |
| `halt` | Halt execution |
| `int` | Raise a software interrupt |
| `iret` | Return from an interrupt |
| `call` | Call a subroutine |
| `ret` | Return from a subroutine |
| `jmp` | Jump unconditionally |
| `beq` | Branch if equal |
| `bne` | Branch if not equal |
| `bgt` | Branch if greater |
| `push` | Push a register onto the stack |
| `pop` | Pop a value into a register |
| `xchg` | Exchange register values |
| `add` | Add |
| `sub` | Subtract |
| `mul` | Multiply |
| `div` | Divide |
| `not` | Bitwise complement |
| `and` | Bitwise AND |
| `or` | Bitwise OR |
| `xor` | Bitwise XOR |
| `shl` | Shift left |
| `shr` | Shift right |
| `ld` | Load a value |
| `st` | Store a value |
| `csrrd` | Read a control register |
| `csrwr` | Write a control register |
| `probe` | Print debug output |

</details>

<details>
<summary>Assembler directive reference</summary>

| Directive | Purpose |
| --- | --- |
| `.global` | Export symbols |
| `.globl` | Export symbols (alias of .global) |
| `.extern` | Declare external symbols |
| `.section` | Select a named section |
| `.byte` | Emit byte values |
| `.half` | Emit halfword values |
| `.word` | Emit word values |
| `.skip` | Reserve bytes |
| `.ascii` | Emit a string |
| `.equ` | Define a symbol from an expression |
| `.end` | End assembly |

</details>

See [the assembly examples](test/) for data, symbols, addressing modes, and
interrupt handlers.

## Linker

```sh
./linker -hex [-place=SECTION@0xADDRESS] -o OUTPUT INPUT.o...
```

The linker combines sections, resolves symbols, and applies relocations. Supply
all required objects on the command line. Repeat `-place` to assign addresses to
multiple sections:

```sh
./linker -hex -place=my_code@0x40000000 -place=math@0xF0000000 \
    -o program.hex main.o math.o handler.o
```

Place the section containing the first instruction at **`0x40000000`**: the
emulator starts at that fixed address. An entry label alone does not select the
initial program counter. Unplaced sections are assigned addresses starting at zero.

Output defaults to standard output when `-o` is omitted. Without `-hex`, the linker
prints a readable dump. It also accepts `-relocatable` to merge objects without
final placement; this option cannot be combined with `-hex` and currently produces
a readable dump.

## Emulator

```sh
./emulator [-timer] [-terminal] [-debug] PROGRAM.hex
```

| Option | Effect |
| --- | --- |
| `-timer` | Enable timer interrupts |
| `-terminal` | Enable terminal input |
| `-debug` | Print instruction and register diagnostics |

Devices are disabled unless their options are supplied. Use `-` as the input
filename to read a hex image from standard input.

The CPU has 16 general-purpose 32-bit registers and four-byte instructions.
Execution starts with `pc = 0x40000000` and `sp = 0x80000000`. Programs using
interrupts must initialize a suitable stack and install an interrupt handler
through the `handler` control register.

| Memory-mapped address | Device register |
| --- | --- |
| `0xFFFFFF00` | Terminal output |
| `0xFFFFFF04` | Terminal input |
| `0xFFFFFF10` | Timer configuration |

The emulator prints the register and control register state when the program
halts. Programs that wait for input or loop indefinitely need external interaction
or termination.

## Examples and testing

The `test/` directories contain assembly examples for manual inspection and
execution. There is no `make tests` target or automated assertion runner.

To build the supplied [terminal and timer demonstration](test/example/):

```sh
./asembler -hex -o test/example/main.o test/example/main.s
./asembler -hex -o test/example/handler.o test/example/handler.s
./linker -hex -place=my_code@0x40000000 \
    -o test/example/program.hex \
    test/example/handler.o test/example/main.o
./emulator -timer -terminal test/example/program.hex
```

This program echoes terminal input, prints `T` on timer interrupts, and halts
after five terminal input interrupts. Run it in an interactive terminal and type
five characters.

Other examples are in [nivo-a](test/nivo-a/), [nivo-b](test/nivo-b/),
[nivo-c](test/nivo-c/), and [devel](test/devel/). Their `start.sh` scripts refer to
an executable named `assembler` and omit the assembler's `-hex` option. Use the
commands above as a model when running them with the current Makefile and file
formats. The older `test/build` helper also uses stale source and include paths;
use `make`.

## Source layout

| Path | Contents |
| --- | --- |
| [src/asembler.c](src/asembler.c) | Assembly parsing and code generation |
| [src/linker.c](src/linker.c) | Symbol resolution, relocation, and placement |
| [src/emulator.c](src/emulator.c) | CPU, memory, timer, and terminal emulation |
| [src/elf.c](src/elf.c) | ELF structures, linking, and text formats |
| [src/util.c](src/util.c) | Shared utilities |
| [h/](h/) | Headers and architecture definitions |
| [test/](test/) | Assembly examples and helper scripts |
| [Specification](projekat-postavka-v1.0.pdf) | Project and architecture specification in Serbian |

## License

Copyright (C) 2023–2026 Ivan Janevski. Licensed under the [MIT License](LICENSE).
