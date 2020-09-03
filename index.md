### gdb basic commands
`gdb -q PROGRAM --tui` => to use a terminal user interface

`layout asm`        => in the view above will show the assembly on top!

`r` or `run`        => runs the program

`stepi`             => execute one instruction

`display /x $eax`   => displays `eax` in as integer in hexadecimal form each
time the program runs an instruction

`display /t $ah`    => displays `ah` in binary each time the program runs an
instruction

`display /d $sp`    => displays `sp` as signed decimal each time the program
runs an instruction

`display /o $bl`    => displays `bl` register in octal each time the program
runs an instruction

`display /s 0x804a0000` => displays the contents of memory location `0x804a0000`
as a string each time the program runs an instruction

`x/s 0x804a0000`    => examines memory at loc `0x804a0000` as string

`x/x 0x0804a000`    => examines memory at loc `0x0804a000` as hexadecimal int.
You can use `&varname` (from `info variables`) instead, if you don't want to use
a memory address

`x/3x &var2`        => examines memory location of var2 and displays 3 bytes as
hex ints

`disassemble $eip`  => disassembles at `eip` location

`disassemble`       => disassembles at current program location (e.g. where the
bp is hit)

`set disassembly-flavor intel` => sets intel syntax at asm

`info registers`    => shows all registers but `fpu`, `mmx` and `xmm` ones

`info all-registers` => shows all registers

`info functions`    => shows information and names and entry points of functions

`info variables`    => shows information about variables

`break main`        => set a breakpoint in `main` function. Note that if this
was an `asm` program, it wouldn't have a main, it would have a `_start` section

`break _start`      => as above, but for `asm`

### gdb defining hooks

Once you start debugging with gdb, you come to the place where you step into
instructions and each time you step, you want to examine some values of
registers, memory, variables, etc. 

gdb has hooks that allow for this type of automation. Let's say that you debug a
program and each time you step, you want to do the following:
 1. print the contents of `eax`
 2. print the contents of `ebx`
 3. print the contents of `ecx`
 4. print 12 bytes at a memory location pointed by `sample` variable
 5. disassemble 0x10 bytes ahead from `eip` in every iteration

you would do this:
```
(gdb) define hook-stop
Type commands for definition of "hook-stop".
End with a line saying just "end".
>print/x $eax
>print/x $ebx
>print/x $ecx
>x/12b &sample
>disassemble $eip,+10
>end
```

Instead of defining a hook-stop, you could direclty use the `display` command if
you wanted to simply print registers and the contents of the `sample` variable,
without disassemblying:

```
display/x $eax
display/x $ebx
display/x $ecx
display/12b &sample
```

After these commands, you could define a hook with only the `disassemble
$eip,+10` command and you would achieve the same result as with the larger hook
definition

 > NOTE: the `display` can be run before the program is `run`, as you noticed

### memory 

32-bit Linux uses a flat memory model in protected mode.

To view the structure of memory of a program u can use:

`cat /proc/PID/maps` or `pmap`, or attach and view in `gdb`

In `/proc/PID/maps` you can view the columns:
 1. memory ranges
 2. permissions (read, write, execute and shared or private memory)
 3. offset in the file for memory mapped files, otherwise 0
 4. major-minor number from device where the file was loaded
 5. inode number
 6. file path

in the same output, you can see shared libraries mapped in a program, like
`/usr/lib/libc-2.32.so` or `/usr/lib/ld-2.32.so` and others. Note that each time
you run `cat /proc/self/maps` the memory address where each of these shared
libraries is, changes, due to ASLR.

Using `gdb` to view the same info, just debug the desired binary and:
`info proc mappings`

 > NOTE: for the above to work, you need to `break main` and `run` the
 program first 

To use `pmap`, just do a `pmap PID`


### system calls in linux

Syscalls in 32bit Linux can be done in 3 ways:
 1. using the `int 0x80` instruction
 2. using the `SYSENTER` instruction
 3. using [vDSO](https://man7.org/linux/man-pages/man7/vdso.7.html)

In 32bit Linux, syscalls are `#define`d in:
`/usr/include/i386-linux-gnu/asm/unistd_32.h`

### writing assembly

The main steps are the following:
 1. setup `.text` and `.data` sections accordingly
 2. check `/usr/include/i386-linux-gnu/asm/unistd_32.h` for the syscalls you
 want to make
 3. check the relevant man page of the syscall to see how many and what
 arguments takes
 4. pass arguments to registers
 5. put the syscall number in `eax`
 6. do `int 0x80` to execute the syscall

Defining initialized data (`.data` segment) in NASM, examples:
 * `db    0x55`                ; just the byte 0x55 
 * `db    0x55,0x56,0x57`      ; three bytes in succession 
 * `db    'a',0x55`            ; character constants are OK 
 * `db    'hello',13,10,'$'`   ; so are string constants 
 * `dw    0x1234`              ; 0x34 0x12 
 * `dw    'a'`                 ; 0x61 0x00 (it's just a number) 
 * `dw    'ab'`                ; 0x61 0x62 (character constant) 
 * `dw    'abc'`               ; 0x61 0x62 0x63 0x00 (string) 
 * `dd    0x12345678`          ; 0x78 0x56 0x34 0x12 
 * `dd    1.234567e20`         ; floating-point constant 
 * `dq    0x123456789abcdef0`  ; eight byte constant 
 * `dq    1.234567e20`         ; double-precision float 
 * `dt    1.234567e20`         ; extended-precision float

Defining uninitialized data (`.bss` segment), examples:
 * buffer:         `resb    64`              ; reserve 64 bytes 
 * wordvar:        `resw    1`               ; reserve a word 
 * realarray       `resq    10`              ; array of ten reals 
 * ymmval:         `resy    1`               ; one YMM register

Special tokens:
 * `$` => evaluates to the current line
 * `$$` => evaluates to the beginning of the current section

