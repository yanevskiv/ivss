 _
(_)_   _____ ___
| \ \ / / __/ __|
| |\ V /\__ \__ \
|_| \_/ |___/___/

System software!

SYNOPSIS

    ivss is an assembler, linker, and emulator for a hypothetical 32-bit CPU.
    Assembly programs are translated into objects, linked with explicit
    section addresses, and executed in the emulator.

        asembler - Assembler
        linker   - Linker
        emulator - CPU emulator with optional timer and terminal devices

    The project includes examples with interrupt handlers, timer interrupts,
    and terminal input/output. The architecture and assignment are described
    in projekat-postavka-v1.0.pdf (in Serbian).

BUILDING

    Build on Linux with GCC, GNU make, and POSIX threads support:

        make

    This creates asembler, linker, and emulator in the project root, with
    intermediate objects in dist/. No parser generator is required.

    The assembler executable is spelled "asembler", with one s, as defined
    in the Makefile. Run the commands below from the project root.

    To remove the tools and intermediate objects:

        make clean

QUICK START

    Save this program as hello.s:

        .section text
            ld $2, %r1
            ld $3, %r2
            add %r1, %r2
            halt
        .end

    Assemble it, place its code at the emulator's starting address, and run:

        ./asembler -hex -o hello.o hello.s
        ./linker -hex -place=text@0x40000000 -o hello.hex hello.o
        ./emulator hello.hex

    The emulator prints its final register state after halt. In this example,
    r1 is 0x00000002 and r2 is 0x00000005.

    Use -hex for both assembly and linking in this pipeline. Despite the .o
    extension, the intermediate file contains a textual hexadecimal encoding
    of ELF data. Without -hex, the tools write a readable section, symbol,
    and relocation dump, which is for inspection rather than pipeline input.

ASSEMBLER

        ./asembler [-hex] -o OUTPUT INPUT.s

    The assembler accepts one source file and requires an explicit output.
    Use - for standard input, or -o - for standard output.

    Source uses labels, # comments, % register names, and $ immediate values:

        ld $42, %r1         # Load an immediate value
        ld value, %r2       # Load a word from memory
        st %r1, value       # Store a word to memory
        add %r1, %r2        # Add r1 to r2

    Register operands include %r0 through %r15, with %sp and %pc aliases.
    Control registers are %status, %handler, and %cause. Memory operands
    also support register addressing, such as [%r1] and [%r1 + 4].

    Supported instructions:

        halt    Halt execution
        int     Raise a software interrupt
        iret    Return from an interrupt
        call    Call a subroutine
        ret     Return from a subroutine
        jmp     Jump unconditionally
        beq     Branch if equal
        bne     Branch if not equal
        bgt     Branch if greater
        push    Push a register onto the stack
        pop     Pop a value into a register
        xchg    Exchange register values
        add     Add
        sub     Subtract
        mul     Multiply
        div     Divide
        not     Bitwise complement
        and     Bitwise AND
        or      Bitwise OR
        xor     Bitwise XOR
        shl     Shift left
        shr     Shift right
        ld      Load a value
        st      Store a value
        csrrd   Read a control register
        csrwr   Write a control register
        probe   Print debug output

    Supported directives:

        .global  Export symbols
        .globl   Export symbols (alias of .global)
        .extern  Declare external symbols
        .section Select a named section
        .byte    Emit byte values
        .half    Emit halfword values
        .word    Emit word values
        .skip    Reserve bytes
        .ascii   Emit a string
        .equ     Define a symbol from an expression
        .end     End assembly

    See test/ for complete examples of data, symbols, addressing modes,
    and interrupt handlers.

LINKER

        ./linker -hex [-place=SECTION@0xADDRESS] -o OUTPUT INPUT.o...

    The linker combines sections, resolves symbols, and applies relocations.
    Supply all required objects on the command line. Repeat -place to assign
    addresses to multiple sections:

        ./linker -hex -place=my_code@0x40000000 -place=math@0xF0000000 \
            -o program.hex main.o math.o handler.o

    Place the section containing the first instruction at 0x40000000: the
    emulator starts at that fixed address. An entry label alone does not
    select the initial program counter. Unplaced sections are assigned
    addresses starting at zero.

    Output defaults to standard output when -o is omitted. Without -hex,
    the linker prints a readable dump. It also accepts -relocatable to
    merge objects without final placement; this option cannot be combined
    with -hex and currently produces a readable dump.

EMULATOR

        ./emulator [-timer] [-terminal] [-debug] PROGRAM.hex

        -timer      Enable timer interrupts
        -terminal   Enable terminal input
        -debug      Print instruction and register diagnostics

    Devices are disabled unless their options are supplied. Use - as the
    input filename to read a hex image from standard input.

    The CPU has 16 general-purpose 32-bit registers and four-byte
    instructions. Execution starts with pc = 0x40000000 and sp = 0x80000000.
    Programs using interrupts must initialize a suitable stack and install
    an interrupt handler through the handler control register.

    Memory-mapped device addresses:

        0xFFFFFF00  Terminal output
        0xFFFFFF04  Terminal input
        0xFFFFFF10  Timer configuration

    The emulator prints the register and control register state when the
    program halts. Programs that wait for input or loop indefinitely need
    external interaction or termination.

EXAMPLES AND TESTING

    There is no make tests target or automated assertion runner. The test/
    directories contain assembly examples for manual inspection and execution.
    For example, build the supplied terminal and timer demonstration:

        ./asembler -hex -o test/example/main.o test/example/main.s
        ./asembler -hex -o test/example/handler.o test/example/handler.s
        ./linker -hex -place=my_code@0x40000000 \
            -o test/example/program.hex \
            test/example/handler.o test/example/main.o
        ./emulator -timer -terminal test/example/program.hex

    This program echoes terminal input, prints T on timer interrupts, and
    halts after five terminal input interrupts. Run it in an interactive
    terminal and type five characters.

    Other examples are in test/nivo-a, test/nivo-b, test/nivo-c, and
    test/devel. Their start.sh scripts refer to an executable named assembler
    and omit the assembler's -hex option. Use the commands above as a model
    when running them with the current Makefile and file formats. The older
    test/build helper also uses stale source and include paths; use make.

SOURCE LAYOUT

        src/asembler.c             Assembly parsing and code generation
        src/linker.c               Symbol resolution, relocation, placement
        src/emulator.c             CPU, memory, timer, and terminal emulation
        src/elf.c                  ELF structures, linking, and text formats
        src/util.c                 Shared utilities
        h/                         Headers and architecture definitions
        test/                      Assembly examples and helper scripts
        projekat-postavka-v1.0.pdf  Project and architecture specification

AUTHOR

    Copyright (C) 2023-2026 Ivan Janevski 
