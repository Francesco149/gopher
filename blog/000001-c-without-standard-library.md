this is an exploration of programming without libc on x86_64 I did back in 2016. i reformatted it
and updated a few things

credits to http://betteros.org/ which got me into researching libc-free programming.

when we learn C, we are taught that main is the first function called in a C program. in reality,
main is simply a convention of the standard library.

let's write a simple hello world and debug it. we will compile with debug information (flag -g) as
well as no optimization (-O0) to be able to see as much as possible in the debugger.

    $ cat > hello.c << "EOF"
    #include <stdio.h>

    int main(int argc, char* argv[]) {
        printf("hello\n");
        return 0;
    }
    EOF

    $ gcc -O0 -g hello.c
    $ ./a.out
    hello

    $ gdb a.out
    (gdb) break main
    (gdb) run
    (gdb) backtrace
    #0  main (argc=1, argv=0x7fffffffd7f8) at hello.c:6

seems like gdb is hiding stuff from us. let's tell it that we actually care about seeing libc stuff

    (gdb) set backtrace past-main on
    (gdb) set backtrace past-entry on
    (gdb) bt
    #0  main (argc=1, argv=0x7fffffffd7f8) at hello.c:6
    #1  0x00007ffff7a5f630 in __libc_start_main (main=0x400556 <main>,
        argc=1, argv=0x7fffffffd7f8, init=<optimized out>,
        fini=<optimized out>, rtld_fini=<optimized out>,
        stack_end=0x7fffffffd7e8)
        at libc-start.c:289
    #2  0x0000000000400489 in _start ()

that's much better! as we can see, the first function that's really called is _start, which then
calls __libc_start_main which is clearly a standard library initialization function which then calls
main

you can go take a look at _start __libc_start_main in glibc source if you want, but it's not very
interesting for us as it sets up a bunch of stuff for dynamic linking and such that we will never
use since we want a static executable

let's recompile our hello world with optimization (-O2), without debug information and with
stripping (-s) to see how large it is:

    $ gcc -s -O2 hello.c
    $ wc -c a.out
    6208 a.out

6kb for a simple hello world? that's a lot! (note: on some systems this is even bigger due to
binary sections alignment, more on this later)

even if I add other size optimization flags such as

    -Wl,--gc-sections -fno-unwind-tables -fno-asynchronous-unwind-tables -Os

it just won't go below 6kb.

we will now progressively strip this program down by first getting rid of the standard library and
then learning how to invoke syscalls without having to include any headers

so how do we get rid of the standard library? If we try to compile our current code with -nostdlib
we will run into linker errors:

    $ gcc -s -O2 -nostdlib hello.c
    /usr/lib/gcc/x86_64-pc-linux-gnu/4.9.3/../../../../x86_64-pc-linux-
    gnu/bin/ld: warning: cannot find entry symbol _start; defaulting to
    0000000000400120
    /tmp/ccTn8ClC.o: In function `main':
    hello.c:(.text.startup+0xa): undefined reference to `puts'
    collect2: error: ld returned 1 exit status

the linker is complaining about _start missing, which is what we would expect from our previous
debugging

we also have a linker error on puts, which is to be expected since it's a libc function. but how do
we print "hello" without puts?

the linux kernel exposes a bunch of syscalls, which are functions that user-space programs can enter
to interact with the OS

you can see a list of syscalls by running "man syscalls" or visiting this site:
http://man7.org/linux/man-pages/man2/syscalls.2.html

how do we find out which syscall puts uses? we can either look through the syscall list, or simply
install strace to trace syscalls and write a simple program that uses puts

the strace method is extremely useful. if you don't know how to do something with syscalls, do it
with libc and then strace it to see which syscalls it uses on the target architecture.

    $ cat > puts.c << "EOF"
    #include <stdio.h>

    int main(int argc, char* argv[]) {
        puts("hello");
        return 0;
    }
    EOF

    $ gcc puts.c
    $ strace ./a.out > /dev/null
    - stuff we don't care about -
    write(1, "hello\n", 6)                  = 6
    exit_group(0)                           = ?
    +++ exited with 0 +++

so it's using the write syscall.

note how i pipe stdout to /dev/null in strace? that's because strace output is in stderr and we
don't want to have it mixed with a.out's output

let's check the manpage for write:

    $ man 2 write
    SYNOPSIS
           #include <unistd.h>

           ssize_t write(int fd, const void *buf, size_t count);

    DESCRIPTION
           write()  writes  up  to  count bytes from the buffer pointed
           buf to the file referred to by the file descriptor fd.

in linux and other unix-like OSes, there are 3 standard file descriptors:
* stdin: used to pipe data into the program or to read user input
* stdout: output
* stderr: alternate output for error messages

if we read `man stdout`, we will see that they are simply hardcoded as 0, 1 and 2

so all we have to do is replace our puts with a write to stream 1 (stdout)

    #include <unistd.h>

    int main(int argc, char* argv[]) {
        write(1, "hello\n", 6);
        return 0;
    }

let's try to compile it again:

    $ gcc -s -O2 -nostdlib hello.c
    hello.c: In function ‘main’:
    hello.c:6:5: warning: ignoring return value of ‘write’, declared
    with attribute warn_unused_result [-Wunused-result]
         write(1, "hello\n", 6);
         ^
    /usr/lib/gcc/x86_64-pc-linux-gnu/4.9.3/../../../../x86_64-pc-linux-
    gnu/bin/ld: warning: cannot find entry symbol _start; defaulting to
    0000000000400120
    /tmp/ccJXwSsr.o: In function `main':
    hello.c:(.text.startup+0x14): undefined reference to `write'
    collect2: error: ld returned 1 exit status

oh no, the "write" function is part of the standard library. how do we do syscalls without having to
link the standard lib?

let's take a look at section "A.2.1 Calling Conventions" of the AMD64 ABI specification

if you're completely clueless about asm, you should still be able to understand once you see the
example. i'm not that good at asm myself.

https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf

    1. User-level applications use as integer registers for passing the sequence %rdi, %rsi, %rdx,
    %rcx, %r8 and %r9. The kernel interface uses %rdi, %rsi, %rdx, %r10, %r8 and %r9.

    2. A system-call is done via the syscall instruction. The kernel destroys registers %rcx and
    %r11.

    3. The number of the syscall has to be passed in register %rax.

    4. System-calls are limited to six arguments, no argument is passed directly on the stack.

    5. Returning from the syscall, register %rax contains the result of the system-call. A value in
    the range between -4095 and -1 indicates an error, it is -errno.

    6. Only values of class INTEGER or class MEMORY are passed to the kernel.

paraphrasing, we must:

* set rax to the syscall number
* set rdi, rsi, rdx, r10, r8 and r9 to the parameters (not all of them unless you actually have 6)
* execute `syscall`
* return value is in rax

modern linux kernels also memory map fast trampolines to syscalls (this is called the vdso) but
I won't get into that as this is simpler and more portable, and we're trying to understand how this
stuff works to begin with

now if we read section 3.4 of the specification or the quick cheatsheet at
http://wiki.osdev.org/Calling_Conventions , we will see that on AMD64 the registers used to pass
parameters to regular functions are almost the same as the syscalls, except for rcx which
is replaced with r10. the return register is also the same (rax)

we can exploit this. if we make a syscall wrapper that just sets eax to the syscall number and then
call it as a function, we will have all the params in the right registers except for r10 which must
be moved to rcx

the abi also states that:

    Registers %rbp, %rbx and %r12 through %r15 "belong” to the calling function and the called
    function is required to preserve their values.

    In other words, a called function must preserve these registers’ values for its caller.

    Remaining registers "belong" to the called function. If a calling function wants to preserve
    such a register value across a function call, it must save the value in its local stack frame.

which means that we don't have to worry about saving and restoring the values of rdi, rsi, rdx, r10,
r8, rcx r9 and other regs we use for our syscall because it's up to the caller to save them and gcc
will take care of that

putting it all together, this will be our syscall wrapper (in intel syntax):

    write:
      mov rax,1
      jmp _syscall

    _syscall:
      mov r10, rcx /* the one register that differs between syscall and function calls */
      syscall
      ret

with this setup you can easily add new syscall numbers by simply adding a new trampoline similar to
write. all of the syscall trampolines will just put the number into eax and jump to _syscall which
just sets up the parameter registers for a syscall (which amounts to moving rcx to r10 as everything
else is already set up by the function call) and performs the syscall

this isn't the most efficient way to do it: you have an extra call and jmp on every syscall and a
mov even when we don't use enough parameters to reach r10, but it's quite compact and simple

if you're aiming for the smallest possible binary you could cascade a bunch of add's to build the
syscall number based on where you jump and then fall through all the way to _syscall without an
extra jmp. this means that you only use 1 add instruction for each syscall number you add instead
of a mov and a jmp. this is however slower and less readable. here's an example with a few syscall
numbers using this method. if you don't get it don't worry about it, we won't be using this method
and it's really just an extreme binary size optimization thing

    open:
      add r9, 1  /* open = 2. when we jump here, the add for write is also exec'd, 1+1 = 2 */
    write:
      add r9, 1  /* write = 1 */
    read:        /* read = 0 */
      mov r10,rcx
      mov rax,r9 /* the syscall number is finally moved into rax */
      xor r9,r9  /* zero r9 for the next syscall */
      syscall
      ret

    /* note: you would also zero r9 before main is called */
    /* you don't have to use r9, just make sure it's a register that isn't touched by anything */


if you're instead aiming for maximum speed, you might want to just make a macro that puts the
asm directly into your code so you don't have an extra call instruction. I personally don't like
the syntax on gcc asm macros, so I'm just going to do it like this. having multiple instances of
the syscall number mov for the same syscall would probably take up more space too in the long run

so let's actually put this syscall wrapper into a gnu asm file

    cat > hello.S << "EOF"

    /* enable intel asm syntax without the % prefix for registers */
    .intel_syntax noprefix

    /* this marks the .text section of the binary, which contains program code */
    .text

    .global write /* export write to other compilation units (files) */
    write:
      mov rax,1
      jmp _syscall

    _syscall:
      mov r10, rcx /* the one register that differs between syscall and function calls */
      syscall
      ret

    EOF

you can find syscalls numbers here:
* http://betteros.org/ref/syscall.php
* https://filippo.io/linux-syscall-table/

or by simply letting the C preprocessor print it for you:

    $ printf "#include <sys/syscall.h>\nblah SYS_write" | \
      gcc -E - | grep blah
    blah 1

-E runs the preprocessor on the file, expanding all macros and therefore replacing #define consts
with their value, while - means that we use stdin as input (which we pipe in from printf).

then we just mark a line with blah so we can grep it, followed by the constant we want to know

unfortunately this trick doesn't seem to work anymore in latest gcc, it seems to be doing some
magic with syscall number defines. it works fine with tcc though

remember the prototype for write from earlier?

    ssize_t write(int fd, const void *buf, size_t count);

ssize_t and size_t are types defined by unistd. a quick inspection reveals that they are 64-bit
integers and that the extra 's' in ssize means signed:

    $ printf "#include <unistd.h>" | gcc -E - | grep size_t
    typedef long int __blksize_t;
    typedef long int __ssize_t;
    typedef __ssize_t ssize_t;
    typedef long unsigned int size_t;

if we try -m32 we will also see that this will be a 32-bit integer on 32-bit.

I'm just gonna go with `int` on everything for simplicity, but keep in mind that if you want to deal
with big numbers you should make it a 64-bit integer, ideally matching the spec

now we can just declare the prototype for write and use it as long as we are compiling and linking
against our `hello.S`

    int write(int fd, void *buf, int count);

    int main(int argc, char* argv[]) {
      write(1, "hello\n", 6);
      return 0;
    }

if we compile now, we are finally only missing _start!

    $ gcc -s -O2 -nostdlib hello.S hello.c
    /usr/lib/gcc/x86_64-pc-linux-gnu/4.9.3/../../../../
    x86_64-pc-linux-gnu/bin/ld: warning: cannot find entry symbol
    _start; defaulting to 0000000000400120

so, how do we define _start? where do we get argc and argv from?  we need to know the initial state
of registers and the stack

back to the amd64 abi document. in figure 3.9, we can see the initial state of the stack:

    0 to rsp     : undefined
    rsp          : argc      <- top of the stack (last pushed value)
    rsp+8        : argv[0]
    rsp+16       : argv[1]
    rsp+24       : argv[2]
    ...          : ...
    rsp+8*argc   : argv[argc - 1]
    rsp+8+8*argc : 0
    * more stuff we don't care about *

and right below it we have the initial state of the registers:

    %rbp: The content of this register is unspecified at process initialization time, but the user
          code should mark the deepest stack frame by setting the frame pointer to zero.

    %rsp: The stack pointer holds the address of the byte with lowest address which is part of the
          stack. It is guaranteed to be 16-byte aligned at process entry.

    %rdx: a function pointer that the application should register with atexit (BA_OS).

so we know that rbp must be zeroed and that rsp points to the top of the stack.
we don't care aboutrdx.

if you don't understand how the stack works, it's basically a chunk of memory where data is appended
(pushed) or retrieved (pop) at the end.

in amd64's convention we're actually prepending and removing data at the beginning of the block of
memory since the stack is said to "grow downwards", which means that when we push something on the
stack, the stack pointer decreases.

putting it all together, our _start function needs to:
* zero rbp
* put argc into rdi (1st parameter for our function call to main)
* put the stack address of argv[0] into rsi (2nd param for main),
  which will be interpreted as an array of char pointers.
* call main

here's our new hello.S:

    .intel_syntax noprefix
    .text

    .global write
    write:
      mov rax,1
      jmp _syscall

    _syscall:
      mov r10, rcx
      syscall
      ret

    .global _start
    _start:
      xor rbp,rbp /* zero rbp (xoring a value with itself = 0) */
      pop rdi     /* rdi = argc, also moves rsp to the next param */
      mov rsi,rsp /* rest of the stack as an array of char ptr */
      call main
      ret

it finally compiles! It runs correctly, but we crash when we exit:

    $ gcc -s -O2 -nostdlib hello.S hello.c
    $ ./a.out
    hello
    Segmentation fault

but why?

when we execute a call instruction, the return address (address of the intruction to jump to after
the function returns) is pushed onto the stack implicitly and the ret instruction implicitly pops
it and jumps to it.

the _start function is very special, as it has no return address, so our ret instruction in _start
is trying to jump back to an invalid memory location, executing garbage data as code or triggering
access violations.

we need to tell the os to kill our process and never reach the ret in _start. the exit syscall
is just what we need:

    $ man 2 _EXIT
    NAME
           _exit, _Exit - terminate the calling process

    SYNOPSIS
           #include <unistd.h>

           void _exit(int status);

           #include <stdlib.h>

           void _Exit(int status);

the syscall number for exit is 60

the status code will simply be the return value of main, which is stored in rax as we know.

    .intel_syntax noprefix
    .text

    .global write
    write:
      mov rax,1
      jmp _syscall

    _syscall:
      mov r10, rcx
      syscall
      ret

    .global _start
    _start:
      xor rbp,rbp
      pop rdi
      mov rsi,rsp
      call main

      /* pass return value of main to exit syscall */
      mov rdi,rax
      mov rax,60
      syscall

our program finally runs and terminates correctly! let's give ourselves a good pat on the back

    $ gcc -s -O2 -nostdlib hello.S hello.c
    $ ./a.out
    hello

let's check the executable size now:

    $ wc -c a.out
    1008 a.out

we're almost below 1kb and it's 6 times smaller than before, but we can shrink it further.

first of all, gcc generates unwind tables by default, which are used for exception handling and
other stuff we don't care about. we can disable these with
`-fno-unwind-tables -fno-asynchronous-unwind-tables`.

also, on some systems ld will align sections to some size by default (for example on my void machine
it's 8kb). according to `man ld` this can be disabled by passing `-n` or `--nmagic` to the linker,
which through gcc would be `-Wl,n`.

   -n
   --nmagic
       Turn off page alignment of sections, and disable linking against
       shared libraries.  If the output format supports Unix style magic
       numbers, mark the output as "NMAGIC"

there's also `-N/--omagic` which doesn't do much on x86_64 but it will help on i386

let's rebuild with these parameters:

    $ gcc -s -O2 \
        -nostdlib \
        -fno-unwind-tables \
        -fno-asynchronous-unwind-tables \
        -Wl,-n,-N \
        hello.S hello.c

    $ wc -c a.out
    808 a.out

we shaved 200 bytes off (8+kb if your linker was aligning previously)

as a last step, we can check the executable for useless sections:

    $ objdump -x a.out

    a.out:     file format elf64-x86-64
    a.out
    architecture: i386:x86-64, flags 0x00000102:
    EXEC_P, D_PAGED
    start address 0x000000000040011a

    Program Header:
        LOAD off    0x0000000000000000 vaddr 0x0000000000400000
             paddr 0x0000000000400000 align 2**21
             filesz 0x0000000000000153 memsz 0x0000000000000153
             flags r-x
       STACK off    0x0000000000000000 vaddr 0x0000000000000000
             paddr 0x0000000000000000 align 2**4
             filesz 0x0000000000000000 memsz 0x0000000000000000
             flags rwx
    PAX_FLAGS off    0x0000000000000000 vaddr 0x0000000000000000
             paddr 0x0000000000000000 align 2**3
             filesz 0x0000000000000000 memsz 0x0000000000000000
             flags --- 2800

    Sections:
    Idx Name          Size      VMA               LMA               ...
      0 .text         0000005c  00000000004000f0  00000000004000f0  ...
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      1 .rodata       00000007  000000000040014c  000000000040014c  ...
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      2 .comment      0000002a  0000000000000000  0000000000000000  ...
                      CONTENTS, READONLY
    SYMBOL TABLE:
    no symbols

`.text` is the code, `.rodata` is read only data (such as the string "hello" in our case),
so we need both of these.

but what's that .comment section?

    $ objdump -s -j .comment a.out

    a.out:     file format elf64-x86-64

    Contents of section .comment:
     0000 4743433a 20284765 6e746f6f 20342e39  GCC: (Gentoo 4.9
     0010 2e332070 312e352c 20706965 2d302e36  .3 p1.5, pie-0.6
     0020 2e342920 342e392e 3300               .4) 4.9.3.

just information about the compiler, it seems. that's 1 byte for every character of that string,
let's get rid of it!

    $ strip -R .comment a.out
    $ wc -c a.out
    720 a.out

there we go, we have achieved a nearly ten-fold size improvement on our little hello world.

let's set up a build script with all those compiler flags and let's also make it output the
executable with a proper name.

also, i'm going to add the following useful flags:

* `-Os` (instead of O2): optimize for size
* `-Wl,--gc-sections`: get rid of any unused code sections
* `-fdata-sections -ffunction-sections`: separate each function and data entry into its own section.
  this lets gc-sections do its job. these two options combined will get rid of any dead code you
  might accidentally leave in your program. it also gets rid of unused functions in statically
  linked libraries
* `-fno-stack-protector`: doesn't generate extra code to guard against overflows overwriting the
  return address
* `-Wl,-z,noexecstack`: mark the stack memory as non-executable. this is just extra security since we
  don't need to be executing code off the stack's memory
* `-ffreestanding`: disable all builtin libc-reliant stuff
* `-std=c89 -pedantic`: follow the old c89 standard strictly. this should force us to write code
  that's more compatible with old compilers
* `-Wall`: enable all warnings
* `-Werror`: treat all warnings as error. can't let our code build with unchecked warnings
* `-Wl,--build-id=none`: save a few bytes by not adding a build id section to the binary

    $ cat > build.sh << "EOF"
    #!/bin/sh

    exename="hello"

    gcc -std=c89 -pedantic \
        -s -Os -Wall -Werror \
        -nostdlib \
        -ffreestanding \
        -fno-unwind-tables \
        -fno-asynchronous-unwind-tables \
        -fdata-sections -ffunction-sections \
        -Wl,--build-id=none,-n,-N,--gc-sections,-z,noexecstack \
        -fno-stack-protector \
        hello.S hello.c \
        -o $exename &&
    strip -R .comment $exename
    EOF

    $ chmod +x ./build.sh
    $ ./build.sh
    $ wc -c hello
    536 hello
    $ ./hello
    hello

536 bytes!

having to pass the string length every time is annoying, so let's
implement our own strlen and puts.

    int write(int fd, void *buf, int count);

    int strlen(char* s) {
      char* p;
      for (p = s; *p; ++p);
      return p - s;
    }

    int puts(char* s) {
      return write(1, s, strlen(s)) + write(1, "\n", 1);
    }

    int main(int argc, char* argv[]) {
      puts("hello");
      return 0;
    }

if you don't understand my strlen function: C strings are null-terminated (the byte after the last
character is zero), so I just iterate the characters through a pointer until I find a zero byte, and
then I subtract the current address from the beginning of the string

libc does all kinds of crazy tricks to optimize this for large strings, which I haven't looked into

at this point, imagination is the limit. you could implement libc or just make your own c library
exactly the way you want it

one last thing: once you start adding more syscalls to your `.S` files you could use a macro like
so for brevity and use eax instead of rax for a higher chance that the compiler optimizes it to
a smaller mov

    #define c(x, n) \
    .global x; \
    x:; \
      mov eax,n; \
      jmp _syscall

    c(write, 1)
    c(stat, 4)

# porting to i386

let's also explore i386 (x86 32-bit) and port our little program to it

let's rename our platform specific stuff

    $ mv hello.S amd64.S

modify the build script to use the first command line arg to pick architecture (defaults to amd64)

also added a check to pass `-m32` if we're cross compiling from x86_64 to i386 as well as some
align params to make the binary smaller on i386

    #!/bin/sh

    exename="hello"
    arch="${1:-amd64}"
    machine="$(gcc -dumpmachine)"

    case "$arch" in
      i386)
        case "$machine" in
          x86_64*)
            cflags="-m32"
          ;;
        esac
        cflags="$cflags -falign-functions=1 -falign-jumps=1 -falign-loops=1
          -mpreferred-stack-boundary=2"
        ;;
    esac

    gcc $cflags -std=c89 -pedantic \
        -s -Os -Wall -Werror \
        -nostdlib \
        -ffreestanding \
        -fno-unwind-tables \
        -fno-asynchronous-unwind-tables \
        -fdata-sections -ffunction-sections \
        -Wl,--build-id=none,-n,-N,--gc-sections,-z,noexecstack \
        -fno-stack-protector \
        "${arch}.S" hello.c \
        -o $exename &&
    strip -R .comment $exename

let's now grab syscall numbers for i386 which are 4 for write and 1 for exit

we need to write a i386 start.s and you guessed it, it's time to look at the abi specification once
again!

http://www.sco.com/developers/devspecs/abi386-4.pdf

this time I will just summarize the differences from amd64:

* registers are 32-bit (obviously)
* the stack is aligned to 4 bytes
* ebp needs to be zeroed (32-bit version of rbp)
* esp is the stack pointer (32-bit version of rsp)
* return values for functions and syscalls are in eax
* the instruction to enter syscalls is "int 0x80". there's also sysenter but I don't like it
* syscall parameters are passed in ebx, ecx, edx, esi, edi, ebp
* function parameters are passed entirely through the stack by pushing them in reverse order, which
  means that we will be able to access them sequentially every 4 bytes on the stack.
  we can't rely on calls to set up our params here
* functions are expected to preserve ebx, esi, edi, ebp, esp on their own. we will have to save and
  restore these registers manually in our syscall wrappers!
* function callers are expected to clean up the parameters off the stack after the call
* as explained earlier, the return address is implicitly pushed on the stack so the function
  parameters will start at esp+4

in short, our _start will look something like this:

    xor ebp,ebp
    pop esi     /* esi = argc */
    mov ecx,esp /* ecx = argv */
    push ecx    /* push argv */
    push esi    /* push argc */
    call main

    /* pass return value of main to exit */
    mov ebx,eax
    mov eax,1
    int 0x80
    ret

... and our syscall wrapper will look like this:

    write:
      mov eax,4
      jmp _syscall

    _syscall:

      /* preserve whatever is in ebp by pushing it onto the stack */
      push ebp

      /* this creates a "stack frame". because we will push stuff to the stack, we want to save the stack
       * pointer before we do, that way we don't have to calculate where the arguments were */
      mov ebp,esp

      /* other registers we must preserve */
      push ebx
      push esi
      push edi

      /* arguments start at +8 instead of +4 because we pushed ebp earlier. +4 would be the ret addr */
      mov ebx,[ebp+8]
      mov ecx,[ebp+12]
      mov edx,[ebp+16]
      mov esi,[ebp+20]
      mov edi,[ebp+24]
      mov edi,[ebp+28]
      int 0x80

      pop edi
      pop esi
      pop ebx
      pop ebp
      ret

as you can see you must take extra care to push the params in reverse order since the stack is
LIFO, and also keep in mind that esp moves around every time you push/pop

also i386 is extra inefficient as we have to back up so many registers on every syscall and push
params we might not even use

and here's the complete `i386.S`

    .intel_syntax noprefix
    .text

    .global write
    write:
      mov eax,4
      jmp _syscall

    _syscall:
      push ebp
      mov ebp,esp
      push ebx
      push esi
      push edi
      mov ebx,[ebp+8]
      mov ecx,[ebp+12]
      mov edx,[ebp+16]
      mov esi,[ebp+20]
      mov edi,[ebp+24]
      mov ebp,[ebp+28]
      int 0x80
      pop edi
      pop esi
      pop ebx
      pop ebp
      ret

    .global _start
    _start:
      xor ebp,ebp
      pop esi
      mov ecx,esp
      push ecx
      push esi
      call main
      mov ebx,eax
      mov eax,1
      int 0x80

let's run it

    $ ./build.sh
    $ file hello
    hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
    $ wc -c hello
    592 hello
    $ ./hello
    hello

    $ ./build.sh i386
    $ file hello
    hello: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, stripped
    $ wc -c hello
    552 hello
    $ ./hello
    hello

as you can see, the 32-bit executable is slightly smaller. this is mostly because pointers are half
as large compared to 64-bit

since the syscall trampolines can share the same code between amd64 and i386, let's make a header
with a macro to quickly declare syscalls as explained earlier

    .intel_syntax noprefix
    .text

    #define c(x, n) \
    .global x; \
    x:; \
      mov eax,n; \
      jmp _syscall

save this as `x86.S` and now our amd64 and i386 files can include it:

    #include "x86.S"

    c(write, 4)

    _syscall:
    /* ....... */

# quirks with legacy syscalls and structs

some syscalls, such as stat, might return their stuff in a struct.  be extremely careful to check
the struct layout and size of the types used, because it will often change completely between
architectures

let's take a look at stat

    $ man 2 stat
    NAME
           stat, fstat, lstat, fstatat - get file status

    SYNOPSIS
           #include <sys/types.h>
           #include <sys/stat.h>
           #include <unistd.h>

           int stat(const char *pathname, struct stat *buf);
           int fstat(int fd, struct stat *buf);
           int lstat(const char *pathname, struct stat *buf);


as we will now see, the stat struct is substantially different for i386 and amd64 and the field
types are also different in size

64-bit stat types

    $ printf "#include <sys/stat.h>" | gcc -E - | grep -A 1 "int stat"
    extern int stat (const char *__restrict __file,
       struct stat *__restrict __buf) __attribute__ ((__nothrow__ ,
       __leaf__)) __attribute__ ((__nonnull__ (1, 2)));

    $ printf "#include <sys/stat.h>" \
      | gcc -E - | grep -A 60 "struct stat"
    struct stat
    {
        __dev_t st_dev;
        __ino_t st_ino;
        __nlink_t st_nlink;
        __mode_t st_mode;
        __uid_t st_uid;
        __gid_t st_gid;
        int __pad0;
        __dev_t st_rdev;
        __off_t st_size;
        __blksize_t st_blksize;
        __blkcnt_t st_blocks;
    # 91 "/usr/include/bits/stat.h" 3 4
        struct timespec st_atim;
        struct timespec st_mtim;
        struct timespec st_ctim;
    # 106 "/usr/include/bits/stat.h" 3 4
        __syscall_slong_t __glibc_reserved[3];
    # 115 "/usr/include/bits/stat.h" 3 4
    };

    $ printf "#include <sys/stat.h>" | gcc -E - \
      | grep '__dev_t\|__ino_t\|__nlink_t\|__mode_t\|__uid_t\|__gid_t'
    typedef unsigned long int __dev_t;
    typedef unsigned int __uid_t;
    typedef unsigned int __gid_t;
    typedef unsigned long int __ino_t;
    typedef unsigned int __mode_t;
    typedef unsigned long int __nlink_t;

    $ printf "#include <sys/stat.h>" | gcc -E - \
      | grep '__blksize_t\|__blkcnt_t\|__syscall_slong_t\|__off_t'
    typedef long int __off_t;
    typedef long int __blksize_t;
    typedef long int __blkcnt_t;
    typedef long int __syscall_slong_t;

    $ printf "#include <sys/stat.h>" | gcc -E - \
      | grep -A 10 "struct timespec"
    struct timespec
    {
        __time_t tv_sec;
        __syscall_slong_t tv_nsec;
    };

    $ printf "#include <sys/stat.h>" | gcc -E - | grep "__time_t"
    typedef long int __time_t;

32-bit stat types

    $ printf "#include <sys/stat.h>" \
      | gcc -m32 -E - | grep -A 60 "struct stat"
    struct stat
    {
        __dev_t st_dev;
        unsigned short int __pad1;
        __ino_t st_ino;
        __mode_t st_mode;
        __nlink_t st_nlink;
        __uid_t st_uid;
        __gid_t st_gid;
        __dev_t st_rdev;
        unsigned short int __pad2;
        __off_t st_size;
        __blksize_t st_blksize;
        __blkcnt_t st_blocks;
    # 91 "/usr/include/bits/stat.h" 3 4
        struct timespec st_atim;
        struct timespec st_mtim;
        struct timespec st_ctim;
    # 109 "/usr/include/bits/stat.h" 3 4
        unsigned long int __glibc_reserved4;
        unsigned long int __glibc_reserved5;
    };

    $ printf "#include <sys/stat.h>" | gcc -m32 -E - \
      | grep '__dev_t\|__ino_t\|__nlink_t\|__mode_t\|__uid_t\|__gid_t'
    __extension__ typedef __u_quad_t __dev_t;
    __extension__ typedef unsigned int __uid_t;
    __extension__ typedef unsigned int __gid_t;
    __extension__ typedef unsigned long int __ino_t;
    __extension__ typedef unsigned int __mode_t;
    __extension__ typedef unsigned int __nlink_t;

    $ printf "#include <sys/stat.h>" \
      | gcc -m32 -E - | grep '__u_quad_t'
    __extension__ typedef unsigned long long int __u_quad_t;

    $ printf "#include <sys/stat.h>" | gcc -m32 -E - \
      | grep '__blksize_t\|__blkcnt_t\|__syscall_slong_t'
    __extension__ typedef long int __off_t;
    __extension__ typedef long int __blksize_t;
    __extension__ typedef long int __blkcnt_t;
    __extension__ typedef long int __syscall_slong_t;

    $ printf "#include <sys/stat.h>" | gcc -m32 -E - \
      | grep -A 10 "struct timespec"
    struct timespec
    {
        __time_t tv_sec;
        __syscall_slong_t tv_nsec;
    };

    $ printf "#include <sys/stat.h>" | gcc -m32 -E - | grep "__time_t"
    __extension__ typedef long int __time_t;


this is not all there is to it though. some syscalls have multiple versions of them with different
structs for historical reasons, and gcc might wrap them in some weird way, using its own struct.

stat is one of them. let's implement the stat struct as we're told above, print it to stdout and
analyze it

again, I'm cutting corners with the int types for simplicity and readability. feel free to match
the headers 1:1 if you want to be safer

(I added `-Wno-long-long` to the compiler flags for unsigned long long)

    struct timespec {
      long tv_sec;
      long tv_nsec;
    };

    #if __386__
    struct stat {
      unsigned long long st_dev;
      short __pad1;
      unsigned st_ino, st_mode, st_nlink, st_uid, st_gid;
      unsigned long long st_rdev;
      short __pad2;
      int st_size, st_blksize, st_blocks;
      struct timespec st_atim, st_mtim, st_ctim;
    };
    #else
    struct stat {
      unsigned long st_dev, st_ino, st_nlink;
      int st_mode, st_uid, st_gid;
      int __pad0;
      unsigned long st_rdev;
      long st_size, st_blksize, st_blocks;
      struct timespec st_atim, st_mtim, st_ctim;
    };
    #endif

    int write(int fd, void *buf, int count);
    int stat(char* path, struct stat* s);

    int main(int argc, char* argv[]) {
      struct stat s;
      if (!stat("/etc/hosts", &s)) {
        write(1, &s, sizeof(s));
      }
      return 0;
    }


now if we hexdump output from amd64 and i386, we will see that
something is not quite right on i386:

    $ ./build.sh && ./hello | hexdump -C
    00000000  82 08 00 00 00 00 00 00  15 00 06 00 00 00 00 00  |................|
    00000010  01 00 00 00 00 00 00 00  a4 81 00 00 00 00 00 00  |................|
    00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
    00000030  d9 00 00 00 00 00 00 00  00 10 00 00 00 00 00 00  |................|
    00000040  08 00 00 00 00 00 00 00  bd 88 11 5f 00 00 00 00  |..........._....|
    00000050  a6 17 7c 02 00 00 00 00  11 dc 8a 5e 00 00 00 00  |..|........^....|
    00000060  00 00 00 00 00 00 00 00  32 8b b6 5e 00 00 00 00  |........2..^....|
    00000070  f4 9c 3f 04 00 00 00 00                           |..?.....|
    00000078
    $ ./build.sh i386 && ./hello | hexdump -C
    00000000  82 08 00 00 15 00 06 00  a4 81 01 00 00 00 00 00  |................|
    00000010  00 00 00 00 d9 00 00 00  00 10 00 00 08 00 00 00  |................|
    00000020  bd 88 11 5f a6 17 7c 02  11 dc 8a 5e 00 00 00 00  |..._..|....^....|
    00000030  32 8b b6 5e f4 9c 3f 04  00 00 00 00 00 00 00 00  |2..^..?.........|
    00000040  81 80 04 08                                       |....|
    00000044

we know that at least the first 8 bytes should match because st_dev is a 64-bit int on both.

so why does the i386 version have st_ino at the 4th byte?

also there's garbage data at the end of the i386 output

if you scroll through the stat manpage, you will find this:

    Over  time, increases in the size of the stat structure have led to three successive versions of
    stat():  sys_stat()  (slot  __NR_oldstat),  sys_newstat()  (slot  __NR_stat),  and  sys_stat64()
    (slot __NR_stat64) on 32-bit platforms such as i386.  The first two  versions  were  already
    present  in  Linux 1.0 (albeit with different names); the last was added in Linux 2.4.  Similar
    remarks apply for fstat() and lstat().

    The  kernel-internal  versions  of the stat structure dealt with by the different versions are,
    respectively:

          __old_kernel_stat
                 The original structure, with  rather  narrow  fields,
                 and no padding.

          stat   Larger  st_ino  field  and  padding  added to various
                 parts of the structure to allow for future expansion.

          stat64 Even larger st_ino field, larger  st_uid  and  st_gid
                 fields to accommodate the Linux-2.4 expansion of UIDs
                 and GIDs to  32  bits,  and  various  other  enlarged
                 fields  and further padding in the structure.  (Vari-
                 ous padding bytes were eventually consumed  in  Linux
                 2.6,  with  the  advent  of  32-bit  device  IDs  and
                 nanosecond components for the timestamp fields.)

    The glibc stat() wrapper function hides these details from applications, invoking the most
    recent version of the system call provided by the kernel, and repacking the returned information
    if required for old binaries.

so it's likely that glibc is tampering with stat instead of just forwarding the syscall.

you can actually check this by writing a small libc stat test and using strace to trace syscalls:

    $ cat > stattest.c << "EOF"
    #include <sys/stat.h>

    int main() {
      struct stat s;
      stat("/etc/hosts", &s);
      return 0;
    }
    EOF

    $ gcc -m32 stattest.c
    $ strace ./a.out
    execve("./a.out", ["./a.out"], [/* 83 vars */]) = 0
    [ Process PID=22487 runs in 32 bit mode. ]
    ... stuff we don't care about ...
    stat64("/etc/hosts", {st_mode=S_IFREG|0644, st_size=1212, ...}) = 0
    exit_group(0)                           = ?
    +++ exited with 0 +++

yep, as expected, the stat call is getting translated to stat64

so how do we fix this? by not trusting libc headers and digging into the kernel headers (which i
found by googling the kernel struct names):

    $ printf "#include <asm/stat.h>" \
      | gcc -m32 -E - | grep -A 30 "struct stat"
    struct stat {
     unsigned long st_dev;
     unsigned long st_ino;
     unsigned short st_mode;
     unsigned short st_nlink;
     unsigned short st_uid;
     unsigned short st_gid;
     unsigned long st_rdev;
     unsigned long st_size;
     unsigned long st_blksize;
     unsigned long st_blocks;
     unsigned long st_atime;
     unsigned long st_atime_nsec;
     unsigned long st_mtime;
     unsigned long st_mtime_nsec;
     unsigned long st_ctime;
     unsigned long st_ctime_nsec;
     unsigned long __unused4;
     unsigned long __unused5;
    };

that's very different than what glibc headers were telling us there is no padding and st_dev is 4
bytes instead of 8, as well as a lot of other fields having smaller sizes.

what about the 64-bit version of it?

    $ printf "#include <asm/stat.h>" \
      | gcc -E - | grep -A 30 "struct stat"
    struct stat {
     __kernel_ulong_t st_dev;
     __kernel_ulong_t st_ino;
     __kernel_ulong_t st_nlink;

     unsigned int st_mode;
     unsigned int st_uid;
     unsigned int st_gid;
     unsigned int __pad0;
     __kernel_ulong_t st_rdev;
     __kernel_long_t st_size;
     __kernel_long_t st_blksize;
     __kernel_long_t st_blocks;

     __kernel_ulong_t st_atime;
     __kernel_ulong_t st_atime_nsec;
     __kernel_ulong_t st_mtime;
     __kernel_ulong_t st_mtime_nsec;
     __kernel_ulong_t st_ctime;
     __kernel_ulong_t st_ctime_nsec;
     __kernel_long_t __unused[3];
    };

this one seems to have the correct layout, except that some of the
values are unsigned rather than signed.

here's our fixed stat struct:

    struct timespec {
      unsigned long tv_sec;
      unsigned long tv_nsec;
    };

    #ifdef __i386__
    struct stat {
      unsigned st_dev, st_ino;
      unsigned short st_mode, st_nlink, st_uid, st_gid;
      unsigned st_rdev, st_size, st_blksize, st_blocks;
      struct timespec st_atime, st_mtime, st_ctime;
      int __unused[2];
    };
    #else
    struct stat {
      unsigned long st_dev, st_ino, st_nlink;
      unsigned st_mode, st_uid, st_gid, __pad0;
      unsigned long st_rdev;
      long st_size, st_blksize, st_blocks;
      struct timespec st_atim, st_mtim, st_ctim;
      long __unused[3];
    };
    #endif

now we can run it again and verify that the struct is properly populated in both architectures
(I added comments to show where fields are, those aren't actually part of hexdump)

    $ ./stat | hexdump -C
    00000000  12 08 00 00 00 00 00 00  50 59 0a 00 00 00 00 00
             |         dev           |          ino           |

    00000010  01 00 00 00 00 00 00 00  a4 81 00 00 00 00 00 00
             |        nlink          |    mode    |    uid    |

    00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
             |    gid    |  __pad0   |          rdev          |

    00000030  bc 04 00 00 00 00 00 00  00 10 00 00 00 00 00 00
             |          size         |        blksize         |

    00000040  08 00 00 00 00 00 00 00  24 b2 e9 57 00 00 00 00
             |         blocks        |        atim.sec        |

    00000050  d1 f4 e1 2f 00 00 00 00  e8 d8 5e 57 00 00 00 00
             |       atim.nsec       |        mtim.sec        |

    00000060  a0 3a b4 24 00 00 00 00  e8 d8 5e 57 00 00 00 00
             |       mtim.nsec       |        ctim.sec        |

    00000070  20 c8 0f 25 00 00 00 00  00 00 00 00 00 00 00 00
             |       ctim.nsec       |       __unused[0]      |

    00000080  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
             |      __unused[1]      |       __unused[2]      |

    00000090

    $ ./build.sh i386
    $ ./stat | hexdump -C
    00000000  12 08 00 00 50 59 0a 00  a4 81 01 00 00 00 00 00
              |    dev   |    ino    | mode |nlink| uid | gid |

    00000010  00 00 00 00 bc 04 00 00  00 10 00 00 08 00 00 00
              |   rdev   |   size    |   blksize  |   blocks  |

    00000020  24 b2 e9 57 d1 f4 e1 2f  e8 d8 5e 57 a0 3a b4 24
              | atim.sec | atim.nsec |  mtim.sec  | mtim.nsec |

    00000030  e8 d8 5e 57 20 c8 0f 25  00 00 00 00 00 00 00 00
              | ctim.sec | ctim.nsec | __unused4  | __unused5 |

    00000040

in short, try getting structs from kernel headers instead of libc.

# legacy sockets on i386

another thing you should be aware of, is that some syscalls might work entirely differently on i386
because of historical reasons

Socket syscalls are a perfect example. i386 doesn't have accept and as far as I know the other
socket syscalls are also not guaranteed to exist

instead, i386 multiplexes all socket syscalls through a single syscall named "socketcall", which
takes an additional param which specifies which socket operation we want do to, (accept, connect,
etc...) followed by the usual syscall params that we find on amd64

also, parameters for socketcall are passed through a void* array, so the socketcall syscall just
takes two parameters: the call number and the pointer to the parameters array.

googling the socketcall numbers was a bit difficult, but i
eventually found them in `linux/net.h`

    $ man socketcall
    SYNOPSIS
    int socketcall(int call, unsigned long *args);

    DESCRIPTION
    socketcall()  is  a common kernel entry point for the socket system
    calls.  call determines which  socket  function  to  invoke.   args
    points to a block containing the actual arguments, which are passed
    through to the appropriate call.

    User programs should call the appropriate functions by their  usual
    names.   Only standard library implementors and kernel hackers need
    to know about socketcall().

    $ awk '/SYS_(CONNECT|SOCKET)/' /usr/include/linux/net.h
    #define SYS_SOCKET      1               /* sys_socket(2)                */
    #define SYS_CONNECT     3               /* sys_connect(2)               */

here's an example socket application for i386 and amd64 that connects to sdf.org's gopherspace
(205.166.94.16:70) and dumps the output for the root folder.

I got the sockaddr_in struct from netinet/in.h and the socket constants from sys/socket.h

    int read(int fd, void *buf, int count);
    int write(int fd, void *buf, int count);
    void close(int fd);
    int socket(short family, int type, int proto);
    int connect(int fd, void* addr, int addr_size);

    #define AF_INET 2
    #define SOCK_STREAM 1
    #define IPPROTO_TCP 6

    struct sockaddr_in {
      short family;
      short port; /* big endian (!) */
      unsigned char addr[4];
      char zero[8];
    };

    #ifdef __i386__
    int socketcall(int call, void* args);

    int socket(short family, int type, int proto) {
      void* args[3];
      args[0] = (void*)(int)family;
      args[1] = (void*)type;
      args[2] = (void*)proto;
      return socketcall(1, args);
    }

    int connect(int fd, void* addr, int addr_size) {
      void* args[3];
      args[0] = (void*)fd;
      args[1] = addr;
      args[2] = (void*)addr_size;
      return socketcall(3, args);
    }
    #endif

    int main(int argc, char* argv[]) {
      int fd, i;
      struct sockaddr_in addr;
      char buf[1024];

      addr.family = AF_INET;
      addr.port = 70 << 8; /* big endian port */
      /* 205.166.94.16 */
      addr.addr[0] = 205;
      addr.addr[1] = 166;
      addr.addr[2] = 94;
      addr.addr[3] = 16;
      for (i = 0; i < sizeof(addr.zero); ++i) {
        addr.zero[i] = 0;
      }

      fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

      if (fd >= 0 && connect(fd, &addr, sizeof(addr)) >= 0) {
        write(fd, "/\r\n", 3); /* request '/' */
        while (1) {
          i = read(fd, buf, sizeof(buf));
          if (i <= 0) {
            break;
          }
          write(1, buf, i);
        }
      }

      close(fd);
      return 0;
    }

and as you can see, we are running flawlessly on both architectures

    $ ./build.sh && ./hello
    iWelcome to the SDF Public Access UNIX System .. est. 1987...

    $ ./build.sh i386 && ./hello
    iWelcome to the SDF Public Access UNIX System .. est. 1987...

# a few more tricks

* struct assignments may generate calls to memcpy, so you should define your own memcpy with the
  same signature as libc's and make sure it's non-static, otherwise you will crash
* 64-bit integer divisions on i386 will generate calls to `i64 __divdi3(i64,i64)` and
  `u64 __udivdi3(u64,64)` which you should implement yourself. If you don't need to do 64bit
  division often you can just cast to double, do the division and cast back
