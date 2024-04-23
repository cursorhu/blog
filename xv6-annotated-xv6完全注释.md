---
title: 'xv6-annotated:xv6完全注释'
date: 2023-05-17 10:34:55
tags: xv6
categories: xv6

---

xv6是mit6.828操作系统课程的基于unix v6重新实现的教学操作系统。

本文英文部分是x86版本的xv6(mit6.828 2018及以前的版本)完全注释，github：[xv6-annotated](https://github.com/palladian1/xv6-annotated)

本文中文部分是我学习xv6过程中添加了部分中文注解

# DAS BOOT

First things first: in order for a computer to run xv6, we need to load it from
disk into memory and tell the processor to start running it. So how does this
all happen?

## The Boot Process

When you press the power button, the hardware gets initialized by a piece of
firmware called the BIOS (Basic Input/Output System) that comes pre-installed on
the motherboard on a ROM chip. Nowadays, your computer probably uses UEFI loaded
from flash memory, but xv6 pretends like it's 1995 and sticks with BIOS. Since
xv6 runs on x86 hardware, we're gonna have to satisfy all the janky requirements
that come with that architecture, in addition to the BIOS's requirements.

Now the BIOS has to load some *other* code called the boot loader from disk;
then it's the boot loader's job to load the OS and get it running. The boot
loader has to act as a middle-man because the BIOS has no idea where on the disk
you decided to put the OS.

The BIOS will look for the boot loader in the very first sector (512 bytes) of
whatever mass storage device you told it to boot from, which we'll call the boot
disk. The processor will execute the instructions it finds there. This means
you have to make a choice: either your boot loader has to be less than 512 bytes
or you can split it up into smaller parts and have each part load the next one.
xv6 takes the first approach.

The BIOS loads the boot loader into memory at address 0x7C00, then sets the
processor's `%ip` register to that address and jumps to it. Remember that `%eip`
is the instruction pointer on x86? Okay cool. But why did I write `%ip` instead
of `%eip`? Well, the BIOS assumes we're gonna be using 16 bits because of the
hellscape known as backwards-compatibility, so we've gotta pretend like it's
1975 before we can pretend it's 1995. The irony here is that this initial 16-bit
mode is called "real mode". So on top of loading the OS, the boot loader will
also have to shepherd the processor from real mode to 32-bit "protected mode".

One last detail: we'll look at the Makefile and linker script later on, but for
now just keep in mind that the boot loader will be compiled separately from the
kernel, which will be compiled separately from all the user-space programs. This
makes it easier to make sure that the entire boot loader will fit in the first
512 bytes on disk. Eventually, the boot loader and the kernel will be stored on
the same boot disk together, and the user-space programs will be on a separate
disk that holds the file system.

## bootasm.S

Boot loader space is tight, and we want to make sure our instructions are exact,
so we're gonna start off in assembly. The ".S" file extension means it's gonna
be assembled by the GNU assembler `as`, and we're allowed to use C preprocessor
directives like `#include` or `#define` or whatever in the assembly code. Also,
xv6 uses AT&T syntax, so if you read CS:APP or took the online course then it'll
be familiar; if you don't know what that means, then don't worry about it.

### Getting Started

First we include some header files to use some constants; I'll point them out
later. Next up, we gotta tell the assembler to generate 16-bit code, and set a
global label to tell the BIOS where to start executing code.
```asm
.code16         # Tell compiler to generate 16-bit code
.globl start
start:
```

Next up: you know how sometimes you can press a special key to tell the BIOS to
stop what it's doing and let you pick a disk to boot from? Or you move your
mouse around in the BIOS menu and you see the pointer moving? Yeah, that needs
hardware interrupts in order to work, but right now, we don't have the faintest
clue how to handle those if they happen, so let's go ahead and turn those off.
There's an x86 instruction to disable them by clearing the interrupt flag in
the CPU's flags register.
```asm
    cli
```

Now we've gotta handle some of x86's quirks. First off, we're gonna need 20-bit
memory addresses, but we only have 16 bits to work with. x86 uses six segment
registers `%cs` (code segment), `%ds` (data segment), `%ss` (stack segment),
`%es` (extra segment), `%fs` and `%gs` (general-purpose segments) to create 20-
bit addresses from 16-bit ones; we're gonna need the first four. The BIOS
guarantees that `%cs` will be set to zero, but it doesn't make any promises
about the others, so we have to clear them ourselves. We're not using `%eax` for
anything yet, so we'll use that to clear the others. The `w` at the end of `xorw`
and `movw` means we're operating on 16-bit words.
```asm
    xorw    %ax,%ax
    movw    %ax,%ds     # Data segment
    movw    %ax,%es     # Extra segment
    movw    %ax,%ss     # Stack segment
```

This next part is a total hack for backwards-compatibility: sometimes a virtual
address might get converted to a 21-bit physical address, and oh no, what are we
gonna do? Well, some hardware can't deal with 21 bits, so it just ignores it,
but it's 1995, so we've got fancy hardware that can use that extra bit. Wow, you
really know we're in the future when you've got a whole 2 MB of RAM to work
with! So we have to tell the processor not to throw away that 21st bit. The way
we do that is by setting the second bit of the keyboard controller's output port
to line high. I don't know. Don't ask me why. The output ports are 0x64 and
0x60, so we're gonna wait until they're not busy, then set the magic values that
will make this all work.
```asm
seta20.1:
    inb     $0x64,%al   # Wait for not busy
    testb   $0x2,%al
    jnz     seta20.1

    movb    $0xd1,%al   # 0xD1 -> port 0x64
    outb    %al,$0x64

seta20.2:
    inb     $0x64,%al   # Wait for not busy
    testb   $0x2,%al
    jnz     seta20.2

    movb    $0xdf,%al   # 0xDF -> port 0x60
    outb    %al,$0x60
```

### Segmentation

Now it's time to switch to 32-bit "protected mode". Up until now, the processor
has been converting virtual addresses to physical ones using those segment
registers which we cleared, so the mapping has been an identity map. But let's
talk about how x86 converts 32-bit virtual addresses to physical ones; this is
important for the rest of the boot loader code as well as the OS, so you're
gonna have to bear with me for this maelstrom of x86-specific details.

The x86 architecture does the conversion in two steps: first segmentation, then
paging. A virtual address starts off life as a *logical address*. Segmentation
converts that to a *linear address*, and paging converts that to a physical one.

A logical address consists of a 20-bit *segment selector* and a 12-bit offset,
with the segment bits before the offset bits, like `segment:offset`. The CPU's
segmentation hardware uses those segment bits to pick one of those four segment
registers we cleared earlier, which acts as an index into a *Global Descriptor
Table* or GDT. Each entry of this GDT tells you where that segment is found in
memory using a base physical address and a virtual address for the maximum or
limit.

The GDT entry also has some permission bits for that segment; the segmentation
hardware will check whether each address can be written to and whether the
process generating the virtual address has the right permissions to access it.
These checks compare the GDT entry's *Descriptor Privilege Levels*, also known
as *ring levels*, against the *Current Privilege Level*. x86 has four privilege
levels (0-3), so if you've ever heard of the kernel operating in ring 0 or user
code in ring 3, this is where it comes from.

Okay, so the GDT entry will give us the first 20 bits of the new linear address;
the offset bits stay the same. After that, the linear address is ready to be
converted to a physical address by the paging hardware. We'll go over this
second half of the story in the virtual memory section. For now, the point is
this: xv6 is mostly gonna say no thank you to segmentation and stick to paging
alone for memory virtualization.

So we're gonna set up our GDT to map all segments the exact same way: with a
base of zero and the maximum possible limit (with 32 bits, that works out to a
grand total of 4 GB, wow so much RAM, I can't imagine ever needing more). We
have to stick this GDT somewhere in our code so we can point the CPU to it, so
we'll put it at the end and throw a `gdtdesc` label on it. Now we can tell the
CPU to load it up with a special x86 instruction for that.
```asm
    lgdt    gdtdesc
```

### Protected Mode

Good news, everyone! We're finally ready to turn on protected mode, which we do
by setting the zero bit of the `%cr0` control register. Note that the `l` at the
end of the instructions here means we're now using long words, i.e. 32 bits;
`CR0_PE` is defined in the [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h)
header file as 0x1.
```asm
    movl    %cr0, %eax      # Copy %cr0 into %eax
    orl     $CR0_PE, %eax   # Set bit 0
    movl    %ax, %cr0       # Copy it back
```

Oh wait, I lied. Enabling protection mode like we just did doesn't change how
the processor translates addresses. We have to load a new value into a segment
register to make the CPU read the GDT and change its internal segmentation
settings. We can do that by using a long jump instruction, which lets us specify
a code segment selector. We're just gonna jump to the very next line anyway, but
in doing so we'll force the CPU to start using the GDT, which describes a 32-bit
code segment, so *now* we're finally in 32-bit mode! Here, `SEG_KCODE` is a
constant defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as segment 1, for `%cs`; we bitshift it left by 3.
```asm
    ljmp    $(SEG_KCODE<<3), $start32
```

First we signal the compiler to start generating 32-bit code. Then we initialize
the data, extra, and stack segment registers to point to the `SEG_KDATA` entry
of the GDT; that constant is defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as the segment for the kernel
data and stack. We're not required to set up `%fs` and `%gs`, so we'll just zero
them.
```asm
.code 32    # Tell assembler to generate 32-bit code now
start32:
    movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
    movw    %ax, %ds    # Data segment
    movw    %ax, %es    # Extra segment
    movw    %ax, %ss    # Stack segment
    movw    $0, %ax     # Zero the segments not ready for use
    movw    %ax, %fs
    movw    %ax, %gs
```

### The Kernel Stack

Okay, last step in the assembly code now: we have to set up a stack in an unused
part of memory. In x86, the stack grows downwards, so the "top" of the stack--
that is, the most-recently-added byte--is actually at the bottom of the stack in
physical memory. It's annoying, but we're gonna have to keep track of that. The
`%ebp` register points to the base of the stack (i.e., the first byte we pushed
onto the stack), and the `%esp` register holds the address of the top of the
stack (most-recently-pushed byte).

But where should we put the stack? The memory from 0xA_0000 to 0x10_0000 is
littered with a memory regions that I/O devices are gonna be checking, so that's
out. The boot loader starts at 0x7C00 and takes up 512 bytes, so that means it
ends at 0x7E00. So xv6 is gonna start the stack at 0x7C00 and have it grow down
from there, toward 0x0000 and away from the boot loader. Remember how back in
the beginning, we started off the assembly code with a `start` label? That means
that `start` is conveniently located at 0x7C00.
```asm
    movl    $start, %esp
```

And we're done with assembly! Time to move on to C code for the rest of the boot
loader. We'll take over with a C function called `bootmain()`, which should
never return. The linker will take care of connecting the call here to its
definition in [bootmain.c](https://github.com/mit-pdos/xv6-public/blob/master/bootmain.c).
```asm
    call    bootmain
```

### Handling Errors

Wait, what? There's more assembly code after this? Why?

Well, if something goes wrong in `bootmain()`, then the function will return, so
we have to handle that here. Since we usually run OSes we're developing in an
emulator like Bochs or QEMU, we'll trigger a breakpoint and loop. Bochs listens
on port 0x8A00, so we can transfer control back to it there; this wouldn't do
anything on real hardware.
```asm
    movw    $0x8a00, %ax    # 0x8a00 -> port 0x8a00
    movw    %ax, %dx
    outw    %ax, %dx
    movw    $0x8ae0, %ax    # 0x8ae0 -> port 0x8a00
    outw    %ax, %dx
spin:
    jmp     spin            # loop forever
```

### The Global Descriptor Table

Oh, and remember when we promised the hardware that we were gonna give it a GDT?
We even told it to load it from address `gdtdesc`, remember? Well, we have to
deliver on that promise now by defining the GDT here.

x86 expects that the GDT will be aligned on a 32-bit boundary, so we tell the
assembler to do that. Then we use the macros `SEG_NULLASM` and `SEG_ASM` defined
in [asm.h](https://github.com/mit-pdos/xv6-public/blob/master/asm.h) to create three segments: a null segment, a segment for executable
code, and another for writeable data. The null segment has all zeroes; the first
argument to `SEG_ASM` has the permission bits, the second is the physical base
address, and the third is the maximum virtual address. As we said before, xv6
relies mostly on paging, so we set the segments to go from 0 to 4 GB so they
identity-map all the memory.
```asm
.p2align 2      # force 4-byte alignment
gdt:
    SEG_NULLASM                             # null segment
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code segment
    SEG_ASM(STA_W, 0x0, 0xffffffff)         # data segment

gdtdesc:
    .word   (gdtdesc - gdt - 1)     # sizeof(gdt) - 1
    .long   gdt                     # address of gdt
```

## bootmain.c

Okay, the rest of the boot loader is in C now! Most of the code here is just to
interact with the disk in order to read the kernel from disk and load it into
memory. Let's start off by looking at `waitdisk()`.

### waitdisk
```c
void waitdisk(void)
{
    while ((inb(0x1F7) & 0xC0) != 0x40)
        ;
}
```
HEAD. DESK. Why all the magic numbers? At least we're lucky that the name makes
it obvious what this function does; this won't always be true in xv6. Okay, so
this function does only one thing: it loops until the disk is ready. Disk specs
are boring as all hell, so feel free to skip to the next section if you don't
care about the particulars (I don't blame you).

The usual way to talk to the disk is with Direct Memory Access (DMA), in which
devices are hooked up directly to RAM for easy communication. But we haven't
initialized the disk at all or set up any drivers for it; that's the OS's
responsibility, not the boot loader's. Even if we could ask the disk to give us
some data through memory-mapped I/O, we disabled all interrupts, so we wouldn't
know when it's ready. So instead, we have to go back to assembly code (ugh, I
know) to access the disk directly.

Storage disks have all kinds of standardized specifications, among them IDE
(Integrated Drive Electronics) and ATA (Advanced Technology Attachment). The
ATA specs include a Programmed I/O mode where data can be transferred between
the disk and CPU through I/O ports. This is usually a huge waste of resources
because every byte has to be transferred through a port and the CPU is busy the
entire time, but right now beggars can't be choosers.

Each disk controller chip has two buses (primary and secondary) for use with ATA
PIO mode; the primary bus sends data on port 0x1F0 and has control registers on
ports 0x1F1 through 0x1F7. In particular, port 0x1F7 is the status port, which
will have some flags to let us know what it's up to. The sixth bit (or 0x40 in
hex) is the RDY bit, which is set when it's ready to receive more commands. The
seventh bit (i.e., 0x80) is the BSY bit, which if set says the disk is busy.

Since interrupts are disabled, we'll have to manually poll the status port in an
infinite loop until the BSY bit is not set but the RDY bit is: `inb()` is a C
wrapper (defined in [x86.h](https://github.com/mit-pdos/xv6-public/blob/master/bootmain.c)) for the x86 assembly instruction `inb`, which reads
from a port. We don't care about any of the other status flags, so we'll get rid
of them by bitwise-ANDing the result with 0xC0 = 0x40 + 0x80. If the result of
that is 0x40, then only the RDY bit is set and we're good to go.

Phew. That was a lot for just one line of code.

### readsect
```c
void readsect(void *dst, uint offset)
{
    // Issue command
    waitdisk();
    outb(0x1F2, 1);
    outb(0x1F3, offset);
    outb(0x1F4, offset >> 8);
    outb(0x1F5, offset >> 16);
    outb(0x1F6, (offset >> 24) | 0xE0);
    outb(0x1F7, 0x20);

    // Read data
    waitdisk();
    insl(0x1F0, dst, SECTSIZE/4);
}
```
If you skipped the last section: this function reads a sector (which in the
current-year-according-to-xv6 of 1995 is 512 bytes) from disk. Good to see you
again, on to the next section for you!

If you powered through the pain and read about ATA PIO mode above, some of the
magic numbers here might be familiar. First we call `waitdisk()` to wait for the
RDY bit, then we send some stuff over ports 0x1F2 through 0x1F7, which we know
are the command registers for the primary ATA bus.

Note that `uint` is just a type alias for C's `unsigned int`, defined in the
header file [types.h](https://github.com/mit-pdos/xv6-public/blob/master/bootmain.c). The `offset` argument is in bytes, and determines which
sector we're gonna read; sector 0 has to hold the boot loader so the BIOS can
find it, and in xv6 the kernel will start on disk at sector 1.

`outb()` is another C wrapper for an x86 instruction from [x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h); this one's
the opposite of `inb()` because it sends data out to a port. The disk controller
register at port 0x1F2 determines how many sectors we're gonna read. Ports 0x1F3
through 0x1F6 are where the sector's address goes. If you *really* must know
(why?) they're the sector number register, the cylinder low and high registers,
and the drive/head register, in order. Port 0x1F7 was the status port above, but
it also doubles as the command register; we send it command 0x20, aka READ
SECTORS.

Then we wait for the RDY bit again before reading from the bus's data register
at port 0x1F0, into the address pointed to by `dst`. Once again, `insl()` is a
C wrapper for the x86 instruction `insl`, which reads from a port into a string.
The `l` at the end means it reads one long-word (32 bits) at a time.

### readseg
```c
void readseg(uchar *pa, uint count, uint offset)
{
    uchar *epa = pa + count;

    // Round down to sector boundary
    pa -= offset % SECTSIZE;

    // Translate from bytes to sectors; kernel starts at sector 1
    offset = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time. We'd write
    // more to memory than asked, but it doesn't matter -- we load in increasing
    // order.
    for (; pa < epa; pa += SECTSIZE, offset++) {
        readsect(pa, offset);
    }
}
```
Okay, finally, we're done with assembly and disk specs. We're gonna read `count`
bytes starting from `offset` into physical address `pa`. Note that `uchar` is
another type alias for `unsigned char` from [types.h](https://github.com/mit-pdos/xv6-public/blob/master/types.h); this means that `pa` is a
pointer (which is 32 bits in x86) to some data where each piece is 1 byte.

`epa` will point to the end of the part we want to read. Now, `count` might not
be sector-aligned, so we fix that. Declaring `pa` as a `uchar *` lets us do this
pointer arithmetic easily because we know that adding 1 to `pa` makes it point
at the next byte; if it were a `void *` like in `readsect()`, pointer arithmetic
would be undefined. (Actually, GCC lets you do it anyway, but GCC lets you get
away with a lot of crazy stuff, so let's not go there.)

Now that we've got everything set up, we just call `readsect()` in a for loop to
read one sector at a time, and that's it!

Some people have asked about the structure of some of the for loops in xv6,
because they don't always use obvious index variables like `int i`. There are
plenty of reasons to hate C, but I think the way it structures for loops is by
far one of its most powerful features:
```c
for (initialization; test condition; update statements) {
    code
}
```
When evaluating the for loop, C first executes anything in the initialization.
Then it checks whether the test condition is true; if so, it executes the code
inside the loop. Then it carries out the update statements before checking the
test condition again and runnning the code if it's still true.

In the for loop above, the initialization is just an empty statement; all the
variables we want to use have already been set up, so we don't need it and C
will just move on to the next step. The test condition is simple enough. But the
update statement actually increments both `pa` and `offset` at once before going
through the loop again.

Okay great, so now we can read from the disk into memory, so we're all set up to
load the kernel and start running it!

### ELF Files

Before we move on to the star of the show, `bootmain()`, we need to talk about
how a computer can actually recognize a file as executable. When you compile
some code, the result gets spit out in a format that your machine can recognize,
load into memory, and run; it's usually the linker's job to do this. Most Unix
and Unix-like systems use the standardized Executable and Linkable Format, or
ELF, for this purpose.

ELF divides the executable file into sections: `text` (the code's instructions),
`data` (initialized global variables), `bss` (statically-allocated variables
that have been declared but not initialized), `stab` and `stabstr` (debugging
symbols and similar info), `rodata` (read-only data, usually stuff like string
literals).

An ELF file starts with a header which has a magic number: 0x7F followed by the
letters "ELF" represented as ASCII bytes; an OS can use this to recognize an ELF
file. The header also tells you the file's type: it could be an executable, or a
library to be linked with executables, or something else. There's a whole bunch
of other info in the header, like the architecture it's made to run on, version,
etc., but we're gonna ignore most of that.

The most important parts of the header are the part where it tells us where in
the file the processor should start executing instructions and the part that
describes the number of entries, on-disk offset, and size of the program header
table.

The program header table is an array that has one entry for each of the file
sections above that's found in this program. It describes the offset in the file
where each section can be found along with the physical and virtual address at
which that section should be loaded into memory and the size of the section,
both in the file and in memory; these might differ if, e.g. the program contains
some uninitialized variables which don't need to be stored in the file but do
need to have space in memory.

The kernel (along with all the user-space programs) will be compiled and linked
as ELF files, so `bootmain()` will have to parse the ELF header to find the
program header table, then parse that to load each section into memory at the
right address. xv6 uses a `struct elfhdr` and a `struct proghdr`, both defined
in [elf.h](https://github.com/mit-pdos/xv6-public/blob/master/elf.h), for this purpose.

Okay, back to the boot loader to finish up now!

### bootmain

This is the C function that gets called by the first part of the boot loader
written in assembly. Its job will be to load the kernel into memory and start
running it at its entry point, a program called `entry()`.

Next up, we're gonna use `readseg()` to load the kernel's ELF header into memory
at physical address 0x1_0000; the number isn't too important because the header
won't be used for long; we just need some scratch space in some unused memory
away from the boot loader's code, the stack, and the device memory-mapped I/O
region. We'll read 4096 bytes first at offset 0; `readseg()` turns that offset
into sector 1. Remember that we have to convert `elf` into a `uchar *` so that
the pointer arithmetic in `readseg()` works out the way we want it to.

```c
void bootmain(void)
{
    struct elfhdr *elf = (struct elfhdr *) 0x10000;
    readseg((uchar *) elf, 4096, 0);
    // ...
}
```

While we're at it, let's go ahead and make sure that what we're loading really
is an ELF file and not some random other garbage because any of a million things
went wrong during the compilation process, or we got some rootkit that totally
corrupted the kernel or something. It's not really the most robust of checks,
but *eh*. If something went wrong we'll just return, since we know that the code
in `bootasm.S` is ready to handle that with some Bochs breakpoints.
```c
void bootmain(void)
{
    // ...
    if (elf->magic != ELF_MAGIC) {
        return;
    }
    // ...
}
```

Now we have to look at the program header table to know where to find each of
the kernel's segments. The `elf->phoff` field tells us the program header
table's offset from the start of the ELF header, so we'll set `ph` to point to
that and `eph` to point to the end of the table.

```c
void bootmain(void)
{
    // ...
    struct proghdr *ph = (struct proghdr *) ((uchar *) elf + elf->phoff);
    struct proghdr *eph = ph + elf->phnum;
    // ...
}
```

Each entry in the program header table tells us where to find a segment, so
we'll iterate over the entries, reading each one from disk and loading it up. In
this for loop, note that `ph` is a `struct proghdr *`, so incrementing it with
`ph++` increments it by the size of a `struct proghdr` and not by one byte; this
makes it automatically point at the next entry in the table.
```c
void bootmain(void)
{
    // ...
    for (; ph < eph; ph++) {
        uchar *pa = (uchar *) ph->paddr;    // address to load section into
        readseg(pa, ph->filesz, ph->off);   // read section from disk

        // Check if the segment's size in memory is larger than the file image
        if (ph->memsz > ph->filesz) {
            stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
        }
    }
    // ...
}
```
That if statement at the end checks if the section's size in memory should be
larger than its size in the file, in which case it calls `stosb()`, which is yet
another C wrapper from [x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h) for the x86 instruction `rep stosb`, which block
loads bytes into a string. It's used here to zero the rest of the memory space
for that section. Okay, but why would we want to do that? Well, if the reason
it's larger is because it has some uninitialized static variables, then we want
to make sure those start off holding zero (as the C standard requires) and not
whatever garbage value may have been there before.

Last part of the bootloader: let's call the kernel's entry point, `entry()`, and
get it running! But remember how the boot loader is compiled and linked
separately from the kernel? Yeah, that means we can't just call `entry()` as a
function, because then the linker would go "Huh? What entry function? I don't
have any `entry` function here in your symbol table. REJECTED." And then it
would throw a huge error.

Luckily, the ELF header tells us where to find the entry point in memory, so we
could get a pointer to that address. That means... function pointers! If you've
never used function pointers in C before, then this won't be the last time
you'll see them in xv6, so check it out.

A C function is just a bunch of code to be executed in order, right? That means
it shows up in the ELF file's `text` section, which will end up in memory. When
you call a regular old C function, the compiler just adds some extra assembly
instructions to throw a return address on the stack and update the registers
`%ebp` and `%esp` to point to the new function's stack on top of the old one. If
the function getting called has any arguments or local variables, they'll get
pushed onto the stack too. Then the instruction register `%eip` gets updated to
point to the new function section, and that's it. After the compiler is done,
the linker will replace the function's name with its memory address in the
`text` section, and voila, a function call.

The point of all this is that in C we can use pointers to functions; they just
point to the beginning of that function's instructions in memory, where the
`%eip` register would end up pointing if the function gets called. So in this
case, even though we're not linking with the kernel, we can still call into the
entry point by getting its address from the ELF header, creating a function
pointer to that address, then calling the function pointer. The compiler will
still add all the usual stack magic, but instead of the linker determining where
`%eip` should point, we'll do that ourselves.

The first line below declares `entry` as a pointer to a function with argument
type `void` and return type `void`. Then we set `entry` to the address from the
ELF header, then we call it.

Again, this shouldn't return, but if it does then it's the last part of this
function, so this function will return back into the assembly boot loader code.
```c
void bootmain(void)
{
    // ...
    void (*entry)(void);
    entry = (void(*) (void)) (elf->entry);
    entry();
}
```

That's it! Starting from `entry()`, we're officially out of the boot loader and
into the kernel.

## Summary

To summarize, the assembly part of the boot loader (1) disabled interrupts, (2)
set up the GDT and segment registers so the segmentation hardware is happy and
we can ignore it later, (3) set up a stack, and (4) got us from 16-bit real mode
to 32-bit protected mode.

Then the C part of the boot loader just loaded the kernel from disk and called
into its entry point.

ELF headers will continue to haunt us in the kernel's linker script and when we
load user programs from disk in `exec()`, and function pointers will make
another appearance when we get around to handling interrupts. The good news: the
boot loader is one of the most opaque parts of the xv6 code, full of boring
hardware specs and backwards-compatibility requirements, so if you made it this
far, it does get better!

(But it also gets worse... looking at you, [mp.c](https://github.com/mit-pdos/xv6-public/blob/master/mp.c) and [kbd.c](https://github.com/mit-pdos/xv6-public/blob/master/kb.c)...)

# The Beginning: Entry and Paging

## xv6's Memory Layout

The whole point of virtualizing memory is to give users the illusion that they
can roam freely across a limitless field of memory without worrying their pretty
little heads about such boring details as how much physical memory their machine
actually has, or where kernel code is stored, or the fact that their seemingly-
continuous heap space is actually shattered into tons of tiny pages spread out
in possibly random parts of physical memory. As long as user code is well-
behaved, that illusion should hold up; if they do a no-no we'll just smack them
with a segmentation fault.

One downside is that the kernel also has to use virtual memory, so we're faced
with the potentially-complicated challenge of setting things up in physical
memory without knowing where anything is actually located in physical memory! So
xv6 does something that a lot of OSes do: it sets itself up as a higher-half
kernel. That means that in the virtual address space (from 0 to 4 GB), the
kernel will reside in the upper half starting at 2 GB, i.e. address 0x8000_0000
and up; user code will start at 0 and end at 2 GB. Because of this, `KERNBASE`
is defined in [memlayout.h](https://github.com/mit-pdos/xv6-public/blob/master/memlayout.h) as 0x8000_0000.

Then it sets up paging so that all of physical memory is identity-mapped to
virtual memory starting at 0x8000_0000. This makes it really convenient for the
kernel to figure out the physical address of a virtual address it's using; just
subtract `KERNBASE` and you're done. The `V2P` and `V2P_WO` macros defined in
[memlayout.h](https://github.com/mit-pdos/xv6-public/blob/master/memlayout.h) do just that, and the `P2V` and `P2V_WO` add `KERNBASE` to a
physical address to get the kernel virtual address.

Note that I said "kernel virtual address", not just any old virtual address.
Users don't get these kinds of fancy privileges, because they shouldn't be
worrying about where anything is in physical memory. They're running through a
limitless field of virtual memory, remember? So user virtual addresses between 0
and 2 GB will get mapped to totally arbitrary locations in physical memory.

One consequence of this is that xv6 is limited to no more than 2 GB of physical
memory (instead of the 4 GB that 32-bit addresses allow for) in order to map it
all into the top 2 GB of virtual memory. In reality, it's even less, for two
reasons: (1) we also need to map device I/O regions into virtual memory, so
it'll be a little less than 2 GB, and (2) it's hard and annoying to figure out
how much physical memory is actually present on any given machine, so xv6 just
says to hell with all that and picks the totally arbitrary value of a puny 224
MB as the amount of available physical memory (that's `PHYSTOP`, defined in
[memlayout.h](https://github.com/mit-pdos/xv6-public/blob/master/memlayout.h)).

## Paging

Remember when we talked about segmentation, and how we said we'd come back to
paging later? Guess what? It's later.

So all virtual addresses are really "logical addresses", and segmentation turns
those into "linear addresses". In xv6, the boot loader set up the segmentation
hardware to use an identity map, so virtual addresses are the same as logical
addresses are the same as linear addresses. Now paging has to turn those linear
addresses into physical addresses. Just like segmentation uses a GDT and the
segment registers for its mapping, paging uses a page directory, page tables,
and the `%cr3` register.

First, imagine a world where every single time some user code throws up an
address (maybe it looks up a variable, or it calls a function, or it simply
needs to execute the next instruction), the CPU has to stop what it's doing,
save all the user's register contents, load up some kernel code, restore its
register contents, find out where its stack is, get it running, and then ask the
OS where that virtual address is actually located in physical memory. That would
be *so* slow. We don't want that. We want the hardware to do all the address
conversions by itself, and involve the OS only minimally to set up a new page
directory when it starts a new process.

Instead, the x86 hardware uses one of its control registers, `%cr3`, to store a
pointer to a page directory in memory. Then every time it needs to map a linear
address to a physical one, it goes to that page directory and grabs the relevant
entry. That entry is a pointer to a page *table* somewhere else in memory, so
the processor grabs the right entry from there, which points to a 4096-byte page
in some other location.

A linear address has a three-part structure: the 10 most significant bits are an
index that picks an entry from the page directory, the next 10 bits are an index
to pick an entry from whatever page table we've been directed to, and the last
12 bits are an offset that determines where to look in the page that the page
table entry pointed to.

For example, let's say we have a virtual address like 0x9C4A_02BF. If we convert
to binary, split it up, and convert back to hex, we can see that the 10 most
significant bits are 0x271, the next 10 are 0x0A0, and the last 12 are 0x2BF. So
the paging hardware would look at wherever `%cr3` is pointing to find the page
directory; let's just call it `pgdir`. Then it would take entry `pgdir[0x271]`
and go look wherever that's pointing to find the right page table; let's call
that `pgtab271`. Then it would take entry `pgtab271[0x0A0]` and look wherever
that's pointing to find the right page, `pg`. *Then* it would finally
know that the corresponding physical address is `pg + 0x2BF`. Whew.

This still sounds super slow, so the paging hardware uses a cache called the
Translation Lookaside Buffer (TLB) to store recently-used mappings and make them
faster in the future. Since pages are 4096 bytes, it only needs to map a new
page if the addresses some code is asking for crosses a page boundary.

xv6 provides two macros, `PDX` and `PTX` defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h), to recover just the
page directory index bits or the page table index bits, respectively, from a
virtual address.

Finally: an important aspect of virtual memory is that each process should be
isolated from the others, and the kernel should be isolated from user processes.
So each process will get its own page directory, and each entry of that page
directory will say whether it's present (i.e., mapped) or not. If it's present,
then it points to a page table for that process; if it's not present and we try
to access it, we'll get a page fault or a general protection fault. Each entry
in a page table will also say whether that page is present and what kinds of
permissions it has. The bit flags for the permissions are (in order from least
to most significant bit):
* Bit 0: present.
* Bit 1: read/write.
* Bit 2: user (otherwise only the kernel can access it).
* Bit 3: write-through.
* Bit 4: cache disabled.
* Bit 5: accessed (for the TLB).
* Bit 6: page size (we'll talk about this later).
* Bit 7: (unused).

This way, since each process has its own page directory, page tables, and pages,
and each level has specific permissions set, they should never be able to
interfere with each other.

Again, most of the time, the kernel will just happily ignore all this and use
the mapping in the higher half of virtual memory for simplicity. Each user
process's page directory will have the same mapping in the higher half so that
the kernel can keep doing what it's doing no matter which user process is
currently running.

Anyway, back to the code! We left off after the boot loader had finished loading
the kernel into memory; it ended by calling an `entry()` function in the kernel.
We haven't set up paging yet, so that's next on our to-do list. But first, the
kernel is compiled and linked using a *linker script*, so we'll have to look at
that to understand how that sets up memory the way we want it.

## kernel.ld

The gory details of linker scripts as a whole are outside the scope of these
posts, so I'm gonna gloss over a lot of the parts of this file and focus on
the important pieces.

It's important to understand what a linker does in a rough sense, so I'll just
generalize and wave my hands around and say that a compiler takes code in a
high-level language and converts it to assembly, an assembler takes that
assembly code and turns it into machine code, and a linker takes a whole bunch
of machine code files (including any code for library functions) and links them
all together into a single executable file.

注释: compiler -> assembler -> linker, high-level language -> asm code -> machine code

Linking involves three steps that are important for us here: first, the linker
has to assign each piece of code a location in memory, so that different
variables, functions, etc. don't end up colliding; then it replaces references
to that object with its address. Second, it has to resolve any outstanding
symbols (variables, functions, etc.) in each file by looking them up in all the
other files and replacing them with those addresses; the linker can define its
own symbols too. Third, it has to create an output file in a format that the OS
can use, like ELF.

注释: 链接的本质是将符号(symbol)替换为地址值(address)，symbol主要指函数；链接输出二进制程序例如elf

xv6 has decided that command-line flags are too basic for it, so instead it'll
use a linker script [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld) for the GNU linker.

We start off by specifying the output format (32-bit ELF), the architecture
(x86, also known as i386), and the entry point to start executing code. The
convention is to call the entry point `_start`; the ELF header will include its
address, which is how we were able to call it from the boot loader.
```ld
OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
OUTPUT_ARCH(i386)
ENTRY(_start)
```

Next up come the sections. Remember the ELF sections `text`, `rodata`, `data`,
`bss`, and `stab`? Well we've gotta tell the linker where to set them up in
memory, using commands like `. = address`. These are virtual addresses, so since
we want to set up our kernel in the higher half of virtual memory, we'll tell it
to link the code start at 0x8010_0000. Again, we use that address instead of
0x8000_0000 (which maps to physical address 0) because we have to avoid the
address spaces of the boot loader and the memory-mapped I/O devices.

We can also tell the linker where in physical memory the code should be placed
(in linker script lingo, its "load address") using the `AT(address)` command.
We'll use the physical address 0x0010_0000, since that maps to virtual address
0x8010_0000.
```ld
SECTIONS {
    . = 0x80100000;

    .text : AT(0x100000) {
        /* this part tells the linker which files to include in this section */
    }

    /* more sections here... */
}
```

There's one other detail we should check out: the linker can create its own
symbols using the `PROVIDE(symbol = .)` command. If the code happens to declare
its own variable `symbol`, then the linker will just throw away its own version
of it, but if the code uses `symbol` without defining it, then the linker will
replace those references with the contents of that memory location.
```ld
SECTIONS {
    /* virtual address and text sections are defined as above */

    PROVIDE(etext = .);     /* etext will be at the address right after the end
                            of the text section */

    /* rodata, stab, and stabstr sections defined here */

    PROVIDE(data = .);      /* data will be at the address at the very beginning
                            of the data section */

    /* data section defined here */

    PROVIDE(edata = .);     /* edata will be at the address right after the end
                            of the data section */

    /* bss section defined here */

    PROVIDE(end = .);       /* end will be at the very last address at the end
                            of the entire kernel code */
}
```

Those variables will be used later in the kernel code; not so much for their
contents but for their addresses, as pointers to the virtual addresses of
specific parts of the kernel's code in memory. On to the kernel!

## entry.S

I have bad news. That `entry()` function that the boot loader called? It's in
assembly again. :(

### Multiboot Header

Okay, so first off, we've got some more hideous specs to deal with for a bit in
the form of a multiboot header. Multiboot is a specification that lets boot
loaders load up kernel code in a standardized way; the GNU boot loader GRUB uses
it. So this part is mostly here in case you want to run xv6 on real hardware
using GRUB; feel free to skip to `entry()` below.

The original Multiboot specification has since been replaced with Multiboot 2,
but again, it's 1995, so we don't know about that yet.

Multiboot helps compliant kernels and boot loaders identify each other using a
special header. The header must be completely contained in the first 8192 bytes
of the kernel's image, and it must be 32-bit aligned. The header contains three
things: (1) a magic number used for mutual identification and recognition
(0x1BADB002 for kernels, 0x2BADB002 for boot loaders), (2) some flags for the
kernel to inform the boot loader what the kernel requires in order to run
successfully, and (3) a 32-bit unsigned checksum which when added to the other
two fields must have a 32-bit unsigned sum of zero. Depending on the flags that
are set, there may be other components to the Multiboot header.

So we'll start by creating a `multiboot_header` label at the beginning of the
file (and thus, the beginning of the kernel image) and making sure it's aligned
to 32 bits.
```asm
.p2align 2      # Force 4-byte alignment
.text
.globl multiboot_header
multiboot_header:
    # ...
```

Now we'll just add the magic number, set the flags to 0 to indicate no special
requirements, and add the checksum.
```asm
    #define magic 0x1badboo2
    #define flags 0
    .long magic
    .long flags
    .long (-magic-flags)
```

And that's it!

### entry

Back in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld), we said that the linker would set up the kernel's ELF
header to specify the kernel's entry point using `_start`, but `_start` itself
wasn't actually defined there, so we have to do that first. We don't know where
this code will end up in memory, so we'll define an `entry` label and set
`_start` to the address of `entry`. Note that the linker script used virtual
addresses in the higher half, but we haven't set up paging yet, so we'll have to
convert it to a physical address using one of the macros we mentioned earlier.
```asm
.globl _start
_start = V2P_WO(entry)
.globl entry
```

Next up we want to finish setting up virtual memory by enabling paging, but
that's all kinds of complicated, so we're gonna start off with a super simple
version of paging. Part of that difficulty is that there's a bootstrap problem:
we need to allocate pages to hold the page tables themselves, but we can't use
pages without page tables... uhh...

We'll solve that by starting off with a basic, super-simple page directory where
only two entries are mapped: the first entry maps virtual addresses 0 to 4 MB to
physical addresses 0 to 4 MB, and the second entry maps virtual addresses
`KERNBASE` to `KERNBASE` + 4MB to physical addresses 0 to 4 MB. One consequence
is that the entire kernel code and data has to fit in 4 MB.

Why the two entries pointing to the same place? It's to solve another bootstrap
problem. The kernel is currently running in physical addresses close to 0. Once
we enable paging and start using virtual addresses in the higher half, the stack
pointer `%esp`, instruction pointer `%eip`, even the pointer in `%cr3` to the
page directory itself will all still point to low addresses until we update
them. But updating them requires executing instructions, which would require
accessing low addresses a few more times. If we left out the low addresses, we'd
get a page fault, and since we don't have exception handlers set up yet, that
would cause a double fault, which would turn into the dreaded **TRIPLE FAULT**,
in which the processor enters an infinite reboot loop. So yeah, point is, we
need both the low and high mappings for now; we'll get rid of the low mappings
once we're done setting up.

But wait! Aren't page directory entries supposed to point to page tables? How
can they point directly to pages here? It turns out that x86 can skip that
second layer altogether if we use so-called "huge" pages of 4 MB in size instead
of the usual 4 KB. In the long run, this could lead to internal fragmentation,
but it does cut down on the overhead and allows a faster set-up. Plus we're only
gonna use them for a minute while we get ready for the full paging ordeal.

To use 4 MB pages, we have to enable x86's Page Size Extension (PSE) by setting
the fourth bit in the `%cr4` register. `CR4_PSE` is defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as 0x10,
or 00010000 in binary.
```asm
entry:
    movl    %cr4, %eax
    orl     $(CR4_PSE), %eax
    movl    %eax, %cr4
```

We need a page directory before we can set up paging; again, basic version now,
full glorious page directory later. We're gonna do the same thing we did in the
boot loader where we tell the processor to load the page directory now but then
procrastinate actually writing it; this time, we'll write it in C and call it
`entrypgdir`. Then we'll load its physical address into register `%cr3`.
```asm
    movl    $(V2P_WO(entrypgdir)), %eax
    movl    %eax, %cr3
```

Now we can enable (a basic version of) paging! We tell the CPU to start using
the page directory in `%cr3` by setting bit 31 (paging) of register `%cr0`; we
can also set bit 16 (write protect) of the same register to prevent writing to
any pages that the page directory and page tables have marked as read-only.
`CR0_PG` and `CR0_WP` are defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) to set these bits.
```asm
    movl    %cr0, %eax
    orl     $(CR0_PG|CR0_WP), %eax
    movl    %eax, %cr0
```

Now remember how the processor is still running at low addresses? Yeah, let's
fix that. First we'll make a new kernel stack in the higher half that will still
be valid even after we get rid of the lower address mappings. We'll have the
linker save some space for us under the symbol `stack` and set it up there;
`KSTACKSIZE` is defined in [param.h](https://github.com/mit-pdos/xv6-public/blob/master/param.h) as 4096 bytes. So we just set the stack
pointer register `%esp` to the top of that section in order to let the stack
grow down toward the address of `stack`. Again, we'll procrastinate actually
defining `stack`.
```asm
    movl    $(stack + KSTACKSIZE), %esp
```

Now we want to call into the `main()` function, but we don't just want to do
that the usual assembly way of `call main`. That would generate a jump relative
to the current value of `%eip`, which is still in low addresses. We'll use an
indirect jump instead.
```asm
    mov     $main, %eax
    jmp     *%eax
```

Finally, we need to get around to reserving space for the stack. We can do that
with the assembler instruction `.comm symbol, size`:
```asm
.comm stack, KSTACKSIZE
```

## main.c

Awesome, back to C code now! Remember how we procrastinated actually defining
`entrypgdir`? Let's do that now; it's at the bottom of [main.c](https://github.com/mit-pdos/xv6-public/blob/master/main.c).

### entrypgdir
What in the world is this?!
```c
__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {
    [0] = (0) | PTE_P | PTE_W | PTE_PS,
    [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};
```
Okay, bear with me; I promise it's not too bad.

First, the `__attribute__` tells the compiler and linker that the page directory
should be placed in memory at an address that's a multiple of `PGSIZE` (4096
bytes); that's just a requirement of the paging hardware.

Next, we define `entrypgdir` as an array of `NPDENTRIES` (1024, according to
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h)), each of type `pde_t` (a type alias for `unsigned int`, according to
[types.h](https://github.com/mit-pdos/xv6-public/blob/master/types.h)).

Then we initialize the entries: in C, you're allowed to initialize an array by
specifying the values of specific enties; all other enties become zero. You
specify an entry by putting its index in square brackets before its value, so
`[2] 5` will set the entry with index 2 to be 5. Here we initialize the entries
with indices 0 and `KERNBASE >> PDXSHIFT`, which is the same thing as
`PDX(KERNBASE)`, AKA the page directory index corresponding to the virtual
address `KERNBASE`, AKA 0x8000_0000. So basically, we've initialized the page
directory entries corresponding to the low virtual address 0 and the high
virtual address `KERNBASE`.

We set their value to 0, because we want them to map to physical addresses from
0 up to 4 MB. Oh, and remember how page directories and page tables can also
hold permission flags? We want to set flags to say that these pages are present
(so that accessing them doesn't cause a page fault), writeable, and 4 MB in
size; those are defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as `PTE_P`, `PTE_W`, and `PTE_PS`. We can
combine them all together by bitwise-ORing them.

And we're done!

### main

The code in [entry.S](https://github.com/mit-pdos/xv6-public/blob/master/entry.S) finished up by calling into the C function `main()`, which
is where the core set-up happens before we can start running processes. It calls
into basically every single part of the xv6 kernel, so we can't go through all
the functions line-by-line yet; instead I'll just give you an overview of what
they do.

* `kinit1()` solves another bootstrap problem around paging: we need to allocate
    pages in order to use the rest of memory, but we can't allocate those pages
    without first freeing the rest of memory, which requires allocating them...
    You see what I mean. This function will free the rest of memory between the
    `end` of the kernel code (defined in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld), remember?) and 4 MB.
* `kvmalloc()` allocates a page of memory to hold the fancy full-fledged page
    directory, sets it up with mappings for the kernel's instructions and data,
    all of physical memory, and I/O space, then switches to that page directory
    (leaving poor old `entrypgdir` in the trash).
* `mpinit()` detects hardware components like additional CPUs, buses, interrupt
    controllers, etc. Then it determines whether this machine supports this
    crazy new idea where you can have multiple CPU cores. Wow, 1995 is crazy.
* `lapicinit()` programs this CPU's local interrupt controller so that it'll
    deliver timer interrupts, exceptions, etc. when we're ready for them later.
* `seginit()` sets up this CPU's kernel segment descriptors in its GDT; we still
    won't really use segmentation, but we'll at least use the permission bits.
* `picinit()` disables the *ancient* PIC interrupt controller that literally no
    one has ever used since the APIC was introduced in 1989. I don't even know
    what to say. I guess I was mistaken when I assumed it was 1995; I don't
    know.
* `ioapicinit()` programs the I/O interrupt controller to forward interrupts
    from the disk, keyboard, serial port, etc., when we're ready for them later.
    Each device will have to be set up to send its interrupts to the I/O APIC.
* `consoleinit()` initializes the console (display screen) by adding it to a
    table that maps device numbers to device functions, with entries for reading
    and writing to the console. It also sets up the keyboard to send interrupts
    to the I/O APIC.
* `uartinit()` initializes the serial port to send an interrupt if we ever
    receive any data over it. xv6 uses the serial port to communicate with
    emulators like QEMU and Bochs.
* `pinit()` initializes an empty process table so that we can start allocating
    slots in it to processes as we spin them up.
* `tvinit()` sets up and interrupt descriptor table (IDT) so that the CPU can
    find interrupt handler functions to deal with exceptions and interrupts when
    they come.
* `binit()` initializes the buffer cache, a linked list of buffers holding
    cached copies of disk data for more efficient reading and writing.
* `fileinit()` sets up the file table, a global array of all the open files in
    the system. There are other parts of the file system that need to be
    initialized like the logging layer and inode layer, but those might require
    sleeping, which we can only do from user mode, so we'll do that in the first
    user process we set up.
* `ideinit()` initializes the disk controller, checks whether the file system
    disk is present (because both the kernel and boot loader are on the boot
    disk, which is separate from the disk with user programs), and sets up disk
    interrupts.
* `startothers()` loads the entry code for all other CPUs (in [entryothers.S](https://github.com/mit-pdos/xv6-public/blob/master/entryothers.S))
    into memory, then runs the whole setup process again for each new CPU.
* `kinit2()` finishes initializing the page allocator by freeing memory between
    4 MB and `PHYSTOP`.
* `userinit()` creates the first user process, which will run the initialization
    steps that have to be done in user space before spinning up a shell.
* `mpmain()` loads the interrupt descriptor table into the CPU so that we're
    finally completely ready to receive interrupts, then calls the `scheduler()`
    function in [proc.c](https://github.com/mit-pdos/xv6-public/blob/master/proc.c), which enables interrupts on this CPU and starts
    scheduling processes to run. `scheduler()` never returns, so at that point
    we're completely done with setup and we're running the OS proper.

## Summary

The entry code in the xv6 kernel had one job: to set up paging. It kind of
failed at that job, but not for lack of trying! There are just all kinds of
Catch-22s when it comes to paging, so at least it got us partway there by making
a temporary page directory to tide us over until we can throw it away and never
look back.

We also took a sneak peek at all the setup code in `main()`; we're gonna end up
going through it all, but at least now you should have enough of an idea of
what's going on that you can more or less skip around and look at what you need.

# Detour: Spin-Locks

So I know I said I wasn't expecting you to have finished the OSTEP section on
concurrency, but xv6 uses locks all over the place, so we're gonna have to get
comfortable with them right away. Luckily, xv6 primarily uses spin-locks, which
are super simple and work on bare metal; a lot of the more complex/more awesome
locks that OSTEP talks about require an OS beneath them.

I'll give a brief intro to concurrency first in case you haven't made it to that
part in OSTEP; then we'll turn to the spin-lock implementation in xv6.

## A Very Brief, Poor-Man's Intro to Concurrency

TL;DR: Concurrency is your worst nightmare. It'll cause bugs in the places where
you least expect it, and they won't even be consistent: your code might work 95%
of the time, but every once in a while it'll randomly fail and you'll have no
idea why. The good news: xv6 handles it in a super-simple way, so we'll get to
appreciate it as we go along. If you're like me, you might also see the code use
locks when you wouldn't have thought they were needed, and then you'll come to
appreciate just how clever the xv6 authors are.

First off, stop reading this and go watch the discussion of data races and locks
in [the last few minutes of the CS 50 2021 lecture on SQL](https://www.youtube.com/watch?v=LzElj46saa8&t=8762s).
I'm serious, go watch it right now; this post will still be here.

Okay, I'm gonna assume you've seen it now; you should have a decent sense of the
main issues with data races and how locks solve them. But the CS 50 lecture
skipped some details about locks: (1) what Brian (the TA) does when he finds a
locked fridge, (2) how locks are implemented in code, and (3) deadlocks.

### What Does Brian Do?

Let's say process `david` is running on one thread, and it needs to use some
resource (a global variable maybe, or an I/O device like the disk or console)
that other threads might want to use too, so `david` acquires the lock for that
resource. Then process `brian` comes along and wants to use the same resource at
the same time. This could cause a data race, but luckily we've thought ahead and
used a lock, so `brian` can't access it until `david` is done with it and
releases the lock.

First of all, we better hope `david` remembers to release the lock; otherwise
`brian` (and all other processes, even the kernel) will *never* be able to use
that resource. But assuming we're smart and remembered to release it, what does
the `brian` process do in the meantime?

Well, maybe `brian` has some other work to do that he can get started on in the
meantime. But what would that mean for an OS? How would we know, in general,
whether the lines of code that follow the use of a shared resource can be safely
executed if we haven't used that resource yet? That sounds impossible to figure
out without knowing ahead of time what the resource is and how it's used, so
let's just go ahead and skip that idea.

Another option that's actually used often in the real world is for `brian` to
stop trying and go to sleep. Maybe he can put a note on himself asking `david`
to wake him up when he gets back with the milk. So in code, that might look like
`brian` signaling the OS and letting it run a different process until the lock
is released. That sounds nice and all, but at this early stage in our kernel, we
don't even have processes or a scheduler yet, let alone a notion of sleeping.

Okay, another option: what if `brian` just spins around in circles, or twiddles
his thumbs, or does jumping jacks or whatever until `david` releases the lock?
In code, that means looping over and over forever until the lock is released.
That would be horribly inefficient; think of all the CPU time wasted when one
process just loops over and over again while another process does something slow
while holding a lock! But it's also the approach that xv6 is gonna take, because
at the end of the day, our kernel is still in baby stages and beggars can't be
choosers. So xv6 uses *spin-locks* with loops that only stop when we acquire a
lock.

This means we should be careful when using locks to acquire them only at the
last possible moment when they're absolutely needed, and release them as soon as
they're no longer required, in order to limit the amount of wasted CPU cycles.

### Implementing Locks

We can implement locks as a simple boolean variable: if it's true, then someone
else is using the resource behind the lock. If it's false, then it's unused and
you can go ahead and take it. So an `acquire()` function sets the lock to `true`
and a `release()` function sets it back to `false`. Done!

But it's not so simple: there's actually a race condition hidden in the very
idea of a lock. Think about it for a second: a lock protects some shared
resource, right? And a shared resource is something that more than one process
wants to use? But a lock is itself a thing that more than one process wants to
use... so we haven't actually gotten rid of the race condition. (FLIPS TABLE.)

We have another Catch-22 on our hands, but this time we can't get rid of it with
a clever software trick like we did with the `entrypgdir`. The issue is that no
matter how well we write our code, it will always require more than one step:
first we have to check whether the lock is `true`, then we have to set it to
`true`. But if someone else is doing the same thing at the same time, our
instructions might get executed in parallel and then we'd both acquire the lock
at the same time -> RACE CONDITION.

The solution will require hardware support, using *atomic* instructions -- these
are hardware instructions that are indivisible; no other code can execute in
between ours. One example is the x86 instruction `xchg`, which atomically reads
a value from memory, updates it to a new value, and returns the old value.

Now we're good! A lock can still be a boolean variable but now `acquire` has to
use `xchg`: it should get the old value while simultaneously updating it to
`true`.

Atomic instructions have more overhead than regular ones, so we should only use
them when they're required, like in locks, but otherwise we can stick to the
regular instructions we've always used.

There's one other detail we should be careful about: a lot of the locks in xv6
protect resources that are needed by both interrupt handlers and kernel or user
code. For example, we might use a process table lock to protect the list of all
currently running processes; suppose some kernel code has acquired the lock in
order to run a new process. What happens if a timer interrupt goes off at that
moment? The timer interrupt handler function might need to acquire the lock in
order to switch processes, but it's already being held by the kernel thread. But
the timer interrupt might take priority over the kernel thread and refuse to
return to the kernel until it finishes executing. The result: that CPU comes to
a total halt as the timer interrupt handler function spins forever, never to get
the lock it so desperately needs to move on. So sad. :(

xv6 avoids this issue in a really simple way: every time we acquire a lock,
we'll just disable interrupts altogether. Problem solved: now a thread can't get
interrupted until it's done using the lock and releases it. This does mean that
a process which grabs locks often might stick around longer than it should,
since we won't have timer interrupts to tell the scheduler to swap it out with
another process, but we're just gonna cross our fingers and hope that doesn't
happen too often.

### Deadlocks

The last concurrency issue we need to be aware of is the problem of deadlocks.
Suppose two threads each need locks A and B; this happens often, e.g. when
loading a user program the kernel will need to hold a lock for the disk and
another for the process table, or a process might be reading from disk and
printing to the console at the same time.

Suppose they're running at the same time, and one process acquires lock A while
the other one acquires lock B. If they each need the other lock to keep going,
they'd spin forever waiting for it. This is a deadlock.

The way to avoid these is to make sure that, if we use more than one lock, we
*always* acquire them in the same order. That way, one process would acquire
lock A, the second one would be unable to acquire it and would spin, then the
first process acquires lock B with no issues. When it's done, it releases both
locks and the second process can continue.

This can get complicated though: if we ever acquire a lock in a function, we'd
have to check any functions that that function calls to see whether they use any
locks, and so on. If they do, and if the order conflicts with another chain of
function calls, we'd have to refactor the code until the orders match. xv6 has
been carefully written so that the lock acquisition order is always consistent.

## spinlock.c

xv6's spin-locks are set up as a `struct spinlock`, defined in
[spinlock.h](https://github.com/mit-pdos/xv6-public/blob/master/spinlock.h). The
`locked` field acts as the boolean variable to determine whether the lock is
held; the other fields are for debugging, since we can expect concurrency issues
to be the one of the most common causes of bugs in the kernel code because,
again, concurrency is your worst nightmare.

Note that `locked` is an `unsigned int` instead of a `bool`; C requires the
standard library header *stdbool.h* in order to use the `bool` type, but on
bare metal we can't assume we have a standard library to use.

### initlock

```c
void initlock(struct spinlock *lk, char *name)
{
    lk->name = name;
    lk->locked = 0;
    lk->cpu = 0;
}
```

This function is pretty straightforward; it just stores the string `name` in
the lock and starts it off as unlocked; the `cpu` field is 0 because no CPU is
holding it yet. Next.

### pushcli and popcli

For reasons mentioned above, we need to disable interrupts whenever we're using
a lock and re-enable them when we release a lock. But if we're not careful, we
could end up enabling interrupts too early when we release one lock while still
holding another; or if interrupts were already disabled when we acquired a lock,
we could unintentionally re-enable them upon releasing it.

xv6 uses paired functions `pushcli()` and `popcli()`.
```c
void pushcli(void)
{
    int eflags = readeflags();
    cli();
    if (mycpu()->ncli == 0) {
        mycpu()->intena = eflags & FL_IF;
    }
    mycpu()->ncli += 1;
}
```
`readeflags()` is a C wrapper for some x86 assembly code that reads from the
`eflags` register; the 9th bit is the interrupt flag, which is set whenever
interrupts are enabled. `cli` is another x86 instruction that clears that flag,
thus disabling interrupts.

`mycpu()` returns a pointer to a `struct cpu` with information about the CPU
running this code; we'll go over these when we talk about processes; here we
increment the `ncli` field in every call to `pushcli()`. If this is the first
call, we save the value of the interrupt flag in the `intena` field.

```c
void popcli(void)
{
    if (readeflags() & FL_IF) {
        panic("popcli - interruptible");
    }
    if (--mycpu()->ncli < 0) {
        panic("popcli");
    }
    if (mycpu()->ncli == 0 && mycpu()->intena) {
        sti();
    }
}
```
`popcli()` first checks to make sure interrupts aren't already enabled and we're
not popping without having pushed. Then it decrements the `ncli` field of the
`struct cpu` for this CPU. If this is the last call to `popcli()`, it checks the
`intena` field; if it was set (i.e., interrupts were enabled before the first
`popcli()`), then it enables interrupts again.

Check out how these two functions are carefully written so that they're matched:
it takes two calls to `popcli()` to undo two calls to `pushcli()`. Also, if
interrupts were already off before the first call to `pushcli()`, they'll stay
off after the last `popcli()`. Pretty neat, right?

### holding

This function checks whether this CPU is holding the lock.
```c
int holding(struct spinlock *lock)
{
    pushcli();
    int r = lock->locked && lock->cpu == mycpu();
    popcli();
    return r;
}
```
Not much to talk about here; it just checks (inside calls to `pushcli()` and
`popcli()`) whether the lock is being held and this is the CPU holding it. If
both conditions are true it'll return 1; otherwise 0.

### acquire

The first step in this function is to disable interrupts to avoid deadlocks. We
also make sure we're not already holding the lock; otherwise we'd deadlock
ourselves.
```c
void acquire(struct spinlock *lk)
{
    pushcli();

    if (holding(lk)) {
        panic("acquire");
    }

    // ....
}
```

Next up, we've gotta acquire the lock using the atomic `xchg` instruction,
defined in [x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h).
Like we said before, the trick is to atomically set `locked`
to 1 while returning the old value. If the returned old value is 1, that
means it was already 1 before we got to it, so it's currently being held and we
can't acquire it yet -- gotta spin. But if the returned old value is 0, that
means the lock was free before we got to it, and our `xchg` just updated it to
1, so we've successfully acquired it. No other instruction can occur between
checking the old value and updating it to the new one, so we can be confident
that no one else will be holding the lock at the same time.
```c
void acquire(struct spinlock *lk)
{
    // ...
    while (xchg(&lk->locked, 1) != 0)
        ;
    // ...
}
```

We do have to be careful about one other thing: compiler optimizations can get
pretty wild nowadays, so the order of code on the page isn't necessarily the
order it'll get compiled to or executed in. This is a critical section of code,
so we need to make sure acquiring the lock forms a barrier between the code that
comes before it and the code after it so any reordering doesn't cross the lock
acquisition point. We can do that with a special compiler instruction:
```c
void acquire(struct spinlock *lk)
{
    // ...
    __sync_synchronize();
    // ...
}
```

Finally, we'll record some info about the CPU and process holding the lock for
debugging purposes. Don't worry about `mycpu()` for now, but we'll talk about
`getcallerpcs()` below.
```c
void acquire(struct spinlock *lk)
{
    // ...
    lk->cpu = mycpu();
    getcallerpcs(&lk, lk->pcs);
}
```

### release

Releasing a lock is a little easier than acquiring it: to acquire it, we need to
check whether it's already held and update its value, with both steps together
as an atomic instruction. To release it, we only have to set the value to false.
That's only one instruction, so it's automatically atomic!

Well, almost, but not quite. The compiler works some serious magic behind the
scenes, so there's no guarantee that a single C operation like `lk->locked = 0`
will actually get compiled down to a single assembly instruction. So we're gonna
have to make sure it does by writing it directly in assembly.

We start off by making sure we are already holding the lock before releasing a
lock held by someone else. Then we clear the debug info stored in the lock, and
tell the compiler and processor not to reorder code past the lock release.
```c
void release(struct spinlock *lk)
{
    if (!holding(lk)) {
        panic("release");
    }

    lk->pcs[0] = 0;
    lk->cpu = 0;

    __sync_synchronize();

    // ...
}
```

Next we need to release the lock, i.e. an assembly instruction equivalent to
`lk->locked = 0` in C. C allows in-line assembly code using the `asm` keyword.
We mark it as `volatile`, which prevents the compiler from optimizing the write
away and ensures it'll get written to memory. Finally, we call `popcli()` to
enable interrupts again.
```c
void release(struct spinlock *lk)
{
    // ...
    asm volatile("movl $0, %0" : "+m" (lk->locked) : );

    popcli();
}
```

### getcallerpcs

This function exists to store information about the current process in the lock
for use in debugging. In particular, we want to record the program counters of
the last 10 functions on the call stack so we can try to figure out which
functions were called in which order when concurrency issues inevitably bring
our world crashing down with data races, or to a grinding halt with deadlocks.

In order to get the program counters, we're gonna have to know a bit about how
x86 handles function calls. The `%eip` register (or instruction pointer) holds
the program counter, which tracks the next instruction to be executed. The
`%ebp` register (or base pointer) holds the address of the base of the stack
(i.e., its highest address, since it grows down).

When a function gets called all its arguments are pushed on the stack in reverse
order, so that the first argument is at the top (lowest address) of the stack.
Then the previous function's `%eip` is pushed on the stack, followed by its
`%ebp`:
```
<- low addresses                                               high addresses ->
...  [new function's data]  [old %ebp]  [old %eip]  [new arg1]  [new arg2]  ...
<- top of stack                                               bottom of stack ->
```

Anyway, the point is that if we have the address of the first argument to the
current function, then we can recover the contents of the previous function's
`%ebp` and `%eip` registers: `%eip` is one spot below it on the stack and `%ebp`
is two spots below it.
```c
void getcallerpcs(void *v, uint pcs[])
{
    uint *ebp = (uint *) v - 2;
    // ...
}
```
Note the type casts here -- `v` is a pointer to the first argument, which can be
of any type and size, so we use a `void *`. But both of the `%eip` and `%ebp`
registers hold 32-bit pointers, so `ebp` is declared as a pointer to a `uint`
(a type alias for `unsigned int`, remember?), which makes the pointer arithmetic
work out nicely so that subtracting 2 returns a pointer to the right spot on the
stack.

Now, what we really want is the program counter `%eip`, not the pointer to the
stack base `%ebp`. But we can use the address of `%ebp` to make sure we haven't
gone too far back in the function call history. Remember, we wanna get the
program counters for the last 10 functions in the call stack, then save them in
the `pcs` array.
```c
void getcallerpcs(void *v, uint pcs[])
{
    // ...
    int i;
    for (i = 0; i < 10; i++) {
        // Stop if the %ebp pointer is null or out of range
        if (ebp == 0 || ebp < (uint *) KERNBASE || ebp == (uint *) 0xffffffff) {
            break;
        }
        pcs[i] = ebp[1];
        ebp = (uint *) ebp[0];
    }
    // ...
}
```
Let's talk about those last two lines: the `ebp` pointer in the code holds the
location of the saved `%ebp` register, so `ebp[0]` is the value at that address
(i.e., the actual value of the saved `%ebp` register) and `ebp[1]` is the value
stored one spot above that, i.e. the value of the saved `%eip` register. So
each iteration of the loop will get one `%eip` and store it in a `pcs` entry.

Then we update `ebp` to the actual value at the address it points to, which
means `ebp` will now point to the address of the saved `%ebp` register for the
function one step further back in the call chain. Okay sorry, I know that's
confusing, but basically each iteration of the for loop moves us back to the
function that called this function, then the function that called that one, and
so on.

Okay, whew. So what happens if we break out of the for loop early because we
went all the way back in the call stack? The other entries of `pcs` might hold
some garbage values, so let's just make them null pointers so we know to ignore
them when debugging.
```c
void getcallerpcs(void *v, uint pcs[])
{
    // ...
    for (; i < 10; i++) {
        pcs[i] = 0;
    }
}
```
One last little trick: the previous for loop declared the loop variable `i`
before the loop -- this means `i` will be in scope for the rest of the function
body. If it had been declare inside the for loop like `for (int i = 0; ...)`, it
would fall out of scope at the end of the loop. So we can keep using the same
`i` in this second for loop (without an initialization statement) and know it'll
hold the value it had after finishing the first for loop. If we finished all the
iterations, that value will be 10; otherwise it'll be less. So we use that to
clear any remaining entries of `pcs`.

## Summary

You'll learn to hate concurrency issues in C; newer languages like Rust make
data races a thing of the past, though deadlocks can still rear their ugly
heads. But for now, the xv6 authors have done all the dirty work for us, so we
can just sit back and watch. Note, though, that even the xv6 authors say it's
totally possible that something has slipped past them and the thousands of other
students and instructors that have looked at xv6, so it's probable that xv6
still has some lingering race conditions. See, even the masters struggle with
it. -_-

Anyway, we saw that locks have to be implemented with hardware support using
atomic instructions. C and most languages provide high-level atomics that real-
world operating systems use, but the point of xv6 is elegance in simplicity, not
being a total show-off, so the xv6 spin-locks just use the basic `xchg`.

We took this detour into spin-locks to make sure we all understand some basic
details because we're gonna be seeing a lot of them in the rest of the kernel
code. They're inefficient (because the processor just spins around waiting for
the lock to be released, WHEEEEE), but we gotta make do with the machinery we've
built up so far. xv6 will also use some fancier locks called sleep-locks, but
we'll cross that bridge when we get to it.

# Page Allocation

When we left off before the lock detour, the boot loader had set up a GDT to
ignore segmentation, and the entry code set up some barebones paging with an
`entrypgdir`. But that initial page directory is too limiting to keep for long;
it only mapped the first 4 MB of physical memory. So we want a new one, but we
have to set it up and allocate pages in it before we can actually use it. And
until we switch to it, everything has to happen in those first 4 MB.

## kalloc.c

We start off in this file by declaring the function `freerange()`, which will be
defined below. We have to do this in C in order to call a function in the code
before the compiler has actually seen the function's definition, which comes
below, or maybe in another file. A *declaration* tells the C compiler "I know I
haven't shown you this symbol before, but don't worry; it's just a function that
takes this number of arguments with these types and has a return value of this
type." That lets the compiler keep calm and carry on with its usual type-checks
(weak as they may be in C). A *definition* tells the compiler that this is the
function (or variable) we were talking about, so it'll reserve some space in
memory for it; it also tells the compiler how to evaluate that function whenever
it's called (for variables, an *initialization* will have to tell the compiler
what the value the variable should hold). The linker will take care of matching
function calls (and variable uses) to their definitions, possibly across files.

Usually you'd stick declarations in a C header file and tell the preprocessor to
copy-paste the header into your code with an `#include` directive; then other
files could `#include` that header too. So header files should really be more of
an API kind of thing, for functions that you want other code to be able to call.
This one is just a local helper function, so we'll declare it here instead of in
a header so other code can't use it.
```c
void freerange(void *vstart, void *vend);
```

Okay okay, I know function declarations are like 101-level C, but I wanted to
mention them because we're about to see something similar but a little off next
when we declare `end` as a global array of characters.
```c
extern char end[];
```
The C keyword `extern` lets you define a global variable or function in one file
and use it in another, so in that sense it's similar to the function declaration
above. In fact, the compiler implicitly assumes there's an `extern` before each
function declaration. The difference is that an explicit `extern` lets us do the
same thing for global variables: we tell the compiler and linker "hey, I'm gonna
use a variable of this type with symbol `end`, but don't worry about reserving a
spot in memory for it; that already happened elsewhere."

The really cool thing about `extern` is that the function or variable might not
even be defined in C -- it could come from any other language! We just pass the
compiled object files from the other language together with the C object files
to the linker and it'll match up the definitions and calls.

In this case if you try looking for the place where `end` is defined in the C or
assembly code, you're gonna be disappointed. Turns out it's actually defined in
[kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld), remember? Back then, we said it was gonna be located at the very
first memory address right after the end of the kernel code and data in memory.
We're about to see why it's needed.

Next up, we define a new `struct` type:
```c
struct run {
    struct run *next;
};
```
Hmm, the only member of this `struct run` is a pointer to another `struct run`.
Hopefully, you've seen some singly-linked lists before so you can recognize it
as one of those. Usually it would have another member to hold the data in the
list, but we won't need any extra data here; we'll find out why soon enough.

Last thing before we get to the functions: we define another `struct` type and
declare the global variable `kmem` to be of that type.
```c
struct {
    struct spinlock lock;
    int use_lock;
    struct run *freelist;
} kmem;
```
The syntax here is the usual C thing where we say the type of a variable, then
an identifier, like `int i`; it just looks more confusing because we're also
defining the type at the same time. This `struct` type doesn't get a name like
`struct run` did because we're only gonna need it this one time. The fields are
a spin-lock (hence the detour before coming here), a `use_lock` variable that
we'll treat as a boolean, and a pointer to a `struct run` called `freelist`.

I'm just gonna go ahead and spoil the next two functions for you: we want to use
a better page directory than `entrypgdir`, right? Well then we need to assign
a page of memory for it, plus a page for each of its page tables, plus a page
for each entry in those page tables that's mapped. That means we'll need some
bookkeeping to track which pages have already been assigned. We're gonna use a
linked list of free pages (that's what `struct run` is for); we'll allocate a
page by popping one off the free list, and we'll free a page by pushing it onto
the top of the list.

Note that `kfree()` here is *not* supposed to be a kernel version of the usual C
standard library function `free()`, nor is `kalloc()` supposed to be a kernel
version of `malloc()`. We have no concept of a heap yet, so heap allocation
wouldn't make sense. These functions allocate and free *whole physical pages* to
be added to the current page directory and its page tables.

### kfree

This function will free a single page (4096 bytes, or `PGSIZE`) of memory by
adding it to the front of the free list. It takes an argument `char *v` which is
a virtual address; we're using `char *` here instead of `uint *` or `void *` or
whatever so that the pointer arithmetic increments by a single byte instead of
4 bytes for `uint` or whatever.

First, some sanity checks: `v` should be page-aligned (because we're freeing a
whole page), it should be above `end` (because we don't want to accidentally
overwrite the kernel code), and its corresponding physical address should be
below `PHYSTOP` (because the only addresses we'll use above the top of physical
memory are for memory-mapped I/O devices and we shouldn't be freeing those pages
anyway).
```c
void kfree(char *v)
{
    if ((uint) v % PGSIZE || v < end || V2P(v) >= PHYSTOP) {
        panic("kfree");
    }
    // ...
}
```

Now, if you've programmed in C, you might have come across the dreaded (but oh-
so-common) bug known as a *use-after-free*. This means you called `free()` on
some variable (hopefully one you had `malloc()`-ed before), and then used it
again. Hmm, very naughty! The problem is that that memory might have been re-
allocated to some other variable or even another process, so you might read the
wrong values or overwrite something important. This is a *very* common cause of
security vulnerabilities in C and C++ to this day; it's also not always easy to
spot because huge projects might have you call `malloc()` in one file, then use
the variable somewhere else thousands of lines of code later in some other file,
then call `free()` in yet another file -- plus it's unlikely that all of these
pieces were written by the same person. So let's make this a little easier on
ourselves by filling the freed page with junk (a bunch of 1s everywhere) in the
hope that a use-after-free leads to a crash (and thus debugging and detection)
sooner than it would otherwise.
```c
void kfree(char *v)
{
    // ...
    memset(v, 1, PGSIZE);
    // ...
}
```
You might be familiar with `memset()` from the C standard library in *string.h*,
but we can't risk using standard library functions here because they assume the
code will be provided by the OS, and the implementation might require any of a
million features we haven't implemented yet. So we have to make our own version
for the kernel in [string.c](https://github.com/mit-pdos/xv6-public/blob/master/string.c). We'll get around to looking at that code later on
in an optional detour, but for now just know that it sets the memory starting at
`v` and continuing for `PGSIZE` bytes to hold a bunch of repeated 1s.

Now let's talk concurrency. At any time, multiple threads might want to allocate
or free pages simultaneously; if we're not careful we might accidentally use the
same page twice, which would cause bugs in addition to security vulnerabilities,
because all the per-process isolation that paging gets us would be lost. So much
work down the drain! This is why `kmem` has a lock, which we should use any time
we push to or pop from the free list.

But in the early stages of the kernel we only use a single CPU and interrupts
are disabled, so there's nothing to fear. Plus, locks add overhead, and the
`acquire()` function needs to call `mycpu()`, which we haven't even defined yet,
so let's just go ahead and skip them in the beginning. So `kmem.use_lock` is a
boolean that will tell us whether we need a lock right now or not.
```c
void kfree(char *v)
{
    // ...
    if (kmem.use_lock) {
        acquire(&kmem.lock);
    }
    // ...
}
```

Okay, we're finally at the point where we can free the page. We'll make a
`struct run *r` that points to virtual address `v`, then make its `next` point
to the first entry of the free list. Then we'll update the head of the list to
point at the newly-freed page. This is the standard C idiom to add to the front
of a singly-linked list.
```c
void kfree(char *v)
{
    // ...
    struct run *r = (struct run *) v;
    r->next = kmem.freelist;
    kmem.freelist = r;
    // ...
}
```

There's something interesting here: where are we storing this entry for the free
list? Why, in the free page itself! So each unused page will hold the address of
the next one in its first few bytes.

Finally, we're out of the critical section where we updated the free list, so we
can release the lock.
```c
void kfree(char *v)
{
    // ...
    if (kmem.use_lock) {
        release(&kmem.lock);
    }
}
```

### kalloc

Allocating a page means popping off the head of the free list. We acquire the
lock first, if we need one.
```c
char *kalloc(void)
{
    if (kmem.use_lock) {
        acquire(&kmem.lock);
    }
    // ...
}
```

Next, we get a pointer to the first free page in the list and update the head to
point to the next one in the list. But what if the list is empty? In that case,
the head would be a null pointer, and dereferencing a null pointer (like we do
here in `r->next`) is undefined behavior in C, which means BAD THINGS HAPPEN.
I'm serious -- there are absolutely no restrictions on what might happen, so the
compiler could literally set your computer on fire if it wanted to. In the real
world, that usually means either a segmentation fault or security vulnerability,
or both if you're unlucky. So we should check whether `r` is null (i.e. zero).
if it's nonzero then we can update `r->next`; otherwise we should just return
`r` and hope whoever called us checks whether it's null. Moral of the story:
any call to `kalloc()`, just like any call to `malloc()` in regular C code,
should always be followed by checking whether the returned pointer is null.
```c
char *kalloc(void)
{
    // ...
    struct run *r = kmem.freelist;
    if (r) {
        kmem.freelist = r->next;
    }
    // ...
}
```

Okay, so now we just release the lock, and we're done!
```c
char *kalloc(void)
{
    // ...
    if (kmem.use_lock) {
        release(&kmem.lock);
    }
}
```

### freerange

`kalloc()` and `kfree()` both handle only one page at a time, which can get
annoying if we're trying to free tons of pages at once; also, they can only use
page-aligned virtual addresses, which have to be typecast to `char *`. Let's
simplify our lives with a simple wrapper function to free multiple pages between
two virtual memory addresses `vstart` and `vend` that may not be page-aligned.

Let's assume that `vstart` is the first address after some other data in an
already-allocated page; we don't want to free that page, but the next one, so we
align it to a page boundary by rounding up, then cast that to a `char *`.
```c
void freerange(void *vstart, void *vend)
{
    char *p = (char *) PGROUNDUP((uint) vstart);
    // ...
}
```

Now we can iterate over the pages, starting at `p` and incrementing by `PGSIZE`
until we reach or pass `vend`, freeing pages as we go.
```c
void freerange(void *vstart, void *vend)
{
    // ...
    for (; p + PGSIZE <= (char *) vend; p += PGSIZE) {
        kfree(p);
    }
}
```
Done, next.

### kinit1 and kinit2

Both of these functions get called by the kernel's `main()`. Quick reminder:
we've got an `entrypgdir` that maps two virtual address ranges (0 to 4 MB and
`KERNBASE` to `KERNBASE` + 4 MB) to the physical addresses range from 0 to 4 MB.
We want to leave this baby page directory behind for a grown-up page directory
that maps all of physical memory, but first we needed to figure out how to
allocate pages.

Okay cool, we already did that. But allocation needs a free list, which for now
is just sitting around chilling as an empty list. But we can't free pages if
they're not already allocated, right? Ahh, bootstrap problems! This one's not an
issue; we'll just cheat this one time and free all the memory between `end` (the
end of the kernel code and data in memory) and `PHYSTOP`, even though we didn't
get it from a call to `kalloc()`. Sounds good, right?

I hate to burst your bubble, but kernel development *loves* bursting bubbles.
Turns out there's yet another bootstrap problem: each page has to store the
pointer to the next free page, which means we have to write to that page, which
means that page must already be mapped... but we can't map all of memory until
we initialize the free list by freeing all of memory...

HEAD. DESK. We're screwed.

Okay, obviously the xv6 authors figured this out already. The trick is that we
do have *some* physical memory we can write to: everything between `end` and 4
MB. So we can free that part for now, allocate some of those pages for a fresh
page directory and some pages, then use those pages to map the rest of physical
memory, then come back later and free those pages.

So we'll have to split up the work of setting up the new page directory into two
very similar functions, `kinit1()` and `kinit2()`. The first one will initialize
the lock for the free list but make `kmem.use_lock` false so we don't use a lock
in the early stages of kernel setup. The second one will set it to true so we
start using a lock to allocate and free pages once we have multiple CPUs, a
scheduler, interrupts, etc.

Both of them will use `freerange()` to free the pages in a section of physical
memory. `main()` calls `kinit1()` with arguments to free the range from `end` to
4 MB, and calls `kinit2()` with arguments for the range from 4 MB to `PHYSTOP`.
```c
void kinit1(void *vstart, void *vend)
{
    initlock(&kmem.lock, "kmem");
    kmem.use_lock = 0;
    freerange(vstart, vend);
}

void kinit2(void *vstart, void *vend)
{
    freerange(vstart, vend);
    kmem.use_lock = 1;
}
```

## Summary

This whole file was just to set up page allocation for the new page directory
we're gonna replace `entrypgdir` with. It uses a free list in `kmem`; freeing a
page adds it to the front of the list and allocation pops a page off the front.
We have to populate the free list will pages for all of physical memory, but we
do that in two steps to avoid some bootstrap issues.

Again, this is a *page* allocator, not a *heap* allocator like `malloc()`, but
many heap allocator implementations use linked lists of free heap regions in the
same way. We talked about use-after-free bugs above, but now we can also see why
*double-frees* (in which you free the same memory region more than once) can
cause bugs and security vulnerabilities: they add the same region to it twice,
which then might get allocated to two different variables or processes, which
might ruin the per-process isolation that virtualization is supposed to provide.
In addition, our page allocator handles fixed-size regions, but a heap allocator
needs to use variable regions, so when a memory region gets allocated twice
after a double-free, it might get split up into differently-sized pieces, of
which some parts get allocated to other processes, etc... It's just a nightmare.

Next up, we'll see the full story of virtual memory.

# More Paging: The Kernel Side

We've already talked *plenty* about virtual memory, and I bet you're probably so
over `entrypgdir` by now; let's wrap up its story and get rid of it!

The [vm.c](https://github.com/mit-pdos/xv6-public/blob/master/vm.c) file is HUGE; only [proc.c](https://github.com/mit-pdos/xv6-public/blob/master/proc.c) and [sysfile.c](https://github.com/mit-pdos/xv6-public/blob/master/sysfile.c) match its length. Some
parts deal with the general paging implementation; we'll look at those here. The
rest handles the details of paging for processes and user code, we'll need to
know a bit more about processes in xv6 for that.

## vm.c

After the include directives for the preprocessor, we have a declaraction for
an external symbol defined in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld). This one is the beginning of the
data section for the kernel.
```c
extern char data[];
```

Next we have a definition for a pointer to a global page directory: this is the
fancy new one that's gonna replace `entrypgdir`. Note that `pde_t` is a type for
page directory entries defined in [types.h](https://github.com/mit-pdos/xv6-public/blob/master/types.h); it's just a type alias for `int`.
```c
pde_t *kpgdir;
```

### seginit

This first function gets called directly by the kernel's `main()`; it sets up
the segment descriptors in the GDT as identity maps to all of memory so that we
can ignore them from now on. Wait, didn't we already do that in the boot loader?

Yes, kind of, but that was before the kernel took over, so back then we had no
notion of kernel space versus user space. Now that we do, we want to set the
permission flags for each segment so that we can use the privilege ring levels,
with the kernel in ring 0 and user code in ring 3. That way any misbehaving user
code will get slapped with a segmentation fault the way we've all come to know
and love in C.

We also have some permission flags for protection in the page directory and page
table entries, so maybe we could get away without it? I mean, both kernel code
and user code are read-only anyway, so maybe they could both have a Descriptor
Privilege Level of 3. But no, x86 is gonna shut that right down by forbidding
interrupts that take you from ring level 0 to ring level 3, so all the interrupt
handler functions have to be in kernel space with a kernel code segment selector
at ring level 0.

So we're just gonna have to do it all over again. Great. Well, maybe it's not
too bad, let's take a look... oh god, it's awful. Okay, deep breath.

Each processor has its own GDT, so we're gonna need to call this function once
per CPU. First we figure out which CPU we're on with with the `cpuid()` function
that we'll see later on; for now it... (drumroll)... gets the CPU's ID. Then we
look that up in a global table of CPUs (there's an `extern` declaration for this
in the included [proc.h](https://github.com/mit-pdos/xv6-public/blob/master/proc.h)) and store it in a `struct cpu`; we saw that before in
the spin-lock code, but we'll get around to talking about it more later.
```c
void seginit(void)
{
    struct cpu *c = &cpus[cpuid()];
    // ...
}
```

That `struct cpu` has a field to hold the GDT, so we're gonna add entries for
the kernel code, kernel data, user code, and user data segment descriptors;
those entries are `SEG_KCODE`, `SEG_KDATA`, `SEG_UCODE`, and `SEG_UDATA`,
respectively. Recall that the permission bits are `STA_X` (executable), `STA_R`
(readable), and `STA_W` (writeable); now we're gonna pile on the descriptor
privilege levels for the kernel (0) and user (3, or `DPL_USER`) on top. Besides
those ring levels, we want to ignore segmentation, so each segment should be an
identity map for all virtual memory from 0 to 4 GB (0xffff_ffff).
```c
void seginit(void)
{
    // ...
    c->gdt[SEG_KCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, 0);
    c->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
    c->gdt[SEG_UCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, DPL_USER);
    c->gdt[SEG_UDATA] = SEG(STA_W, 0, 0xffffffff, DPL_USER);
    // ...
}
```
The only difference between the `SEG` macro used here and the `SEG_ASM` one from
the boot loader is that this one is for C code and the other is for assembly.

Finally, we load up the new GDT into the processor with a C wrapper for the
x86 instruction `lgdt`.
```c
void seginit(void)
{
    // ...
    lgdt(c->gdt, sizeof(c->gdt));
}
```

Done with segmentation, now on to more paging.

### walkpgdir

A page directory lets the paging hardware convert virtual addresses to physical
ones, but we're gonna need those mappings in the kernel too while we set up the
page directory, so this function does the conversion manually. Wait, but aren't
we setting up paging so that all of physical memory is mapped in the higher half
of the virtual address space? Can't we just add or subtract `KERNBASE` to do the
conversion? Well, that would work for kernel virtual addresses, but user virtual
addresses actually will use page directories and page tables in a non-obvious
way, so if we want to figure out where those go, we'll need a function for it.

In C, using the `static` keyword before a function limits its scope and makes it
visible only within its own file. The function returns a `pte_t *`, a pointer to
a page table entry (the type is defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as a type alias for `uint`).

Its arguments are a pointer to a page directory, a virtual address, and `alloc`
(a boolean variable, but as an `int` instead of `bool`). This `alloc` lets the
function play a dual role: if it's set, the function will allocate a page table
if needed; otherwise it reports failure if the page table doesn't exist. The
`const` keyword lets the compiler know a variable shouldn't be mutated so it'll
throw an error if we do. Here, `const void *va` is a pointer to a constant value
of any type; the address the pointer holds might change, but we can never write
to that address. The opposite is a `void *const va`: the address being pointed
to will never change, but we can overwrite the contents of that address all we
want. You can combine the two with `const void *const va`. What's that I hear? C
syntax is the worst? No, never...
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
}
```

Remember way back when, when we talked about how "linear" addresses are set up
and converted to physical ones? The first 10 bits are an index for the page
directory to pick a page directory entry, which points to a page table; the next
10 bits pick a page table entry that points to a page, and the last 12 bits are
an offset within that page; the `PDX()` and `PTX()` macros get first 10 bits and
the next 10 bits from a linear address, respectively. So we start by getting the
page directory index and using that to get the page directory entry.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    pde_t *pde = &pgdir[PDX(va)];
    // ...
}
```

Okay, so now `pde` points to a page directory entry which has two parts: a
pointer to the physical address of a page table, and some flags. But who knows
if this page table even exists; most page directory (and page table) entries
aren't mapped in order to save space. So we have to check whether `*pde` has the
`PTE_P` (present) flag set.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        // ...
    }
    // ...
}
```

If the page table exists, we should get rid of the flags and recover the pointer
to the page table using the `PTE_ADDR()` macro. But the hardware uses physical
addresses for these pointers, so we need to convert it to a virtual address
first, which is what this function does... recursion? Bootstrap problem? No,
it's actually easy because we can access the page table from within the kernel's
virtual address space in the higher half by adding `KERNBASE` to the physical
address with the `P2V()` macro.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        pgtab = (pte_t *) P2V(PTE_ADDR(*pde));
    } else {
        // ...
    }
    // ...
}
```

Now for the else clause, which happens if the page directory entry doesn't have
the `PTE_P` bit set. Well, if the boolean `alloc` is false (zero), then we're
done and we should just report failure by returning a null pointer. On the other
hand, if it's true, we just allocate a page for the page table. But wait,
remember how page allocation might fail and return a null pointer if we're out
of free pages in the free list? And remember how I said we should always check
for that? Okay well let's check for that; if allocation fails, we also return a
null pointer. Oh, and because this is C, we're gonna do a jillion things at once
in a single line: check if `alloc` is false, try to allocate a page table, and
check if that allocation failed. C lets us assign to a variable and then test
that variable's value in a single statement.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        if (!alloc || (pgtab = (pte_t *) kalloc()) == 0) {
            return 0;
        }
        // ...
    }
    // ...
}
```

Okay, so now suppose: (1) the page table wasn't present, (2) alloc was set, and
(3) we successfully allocated a page. Now what? Remember how we filled all free
pages with garbage in `kfree()` using `memset()`? Let's undo that now by zeroing
it.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        // ...
        memset(pgtab, 0, PGSIZE);
        // ...
    }
    // ...
}
```

Now we'll update the page directory entry to point to this new page table and
add the `PTE_P` flag so it knows it's present. Wait, while we're at it, what
other permissions will it need? Is it writeable? Can users access it? Hmm, we'd
have to know whether we're looking up a user virtual address or a kernel one,
and whether it's gonna be used for code or data. Ah, screw it, we'll just throw
all the flags on there at once. Either way, the page table entries will have
their own flags too, so we can restrict the page's permissions there instead of
here at the page directory entry.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        // ...
        *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
    }
    // ...
}
```
This probably isn't the safest thing ever, because we're saying that only the
page table will restrict permissions, so we're throwing all that responsibility
over there, but hey, xv6 is supposed to be simple, not ultra-secure. Just don't
do this at home, kids.

Finally, we return the address of the corresponding page table entry using the
index from the middle bits of `va`:
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    return &pgtab[PTX(va)];
}
```

### mappages

Okay, so `walkpgdir()` returns a pointer to a page table entry and can even
crate a page table if if it doesn't exist. That's not quite enough to add new
mappings for pages though; the page itself might not be mapped, and if we just
created a new page table, then certainly none of the pages are mapped yet.
`mappages()` will finish the job by installing mappings in page tables (possibly
newly-allocated ones) for a range of virtual addresses.

The arguments are a page directory, a virtual address for the beginning of the
range, the size of the range, a physical address to map it to, and the flags for
permissions we want to set. We start off by rounding the virtual address down to
the nearest page boundary and getting a pointer to the end of the range, also
page-aligned.
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    char *a = (char *) PGROUNDDOWN((uint) va);
    char *last = (char *) PGROUNDDOWN(((uint) va) + size - 1);
    // ...
}
```

Now we're gonna iterate over the pages in that range; `for (;;)` is a common C
idiom for an infinite loop. In this case, we need to increment `a` and `pa` by
`PGSIZE` each time, and we'll break out of the loop when `a` reaches `last`. To
be completely honest, I'm not really sure why the authors chose to write this as
an infinite loop with the condition/break statement and update statements inside
the loop rather than as a regular old for loop; I think the latter would be more
clear, but oh well, I didn't write this.
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        // ...
        if (a == last) {
            break;
        }
        a += PGSIZE;
        pa += PGSIZE;
    }
    // ...
}
```

Inside the for loop, we'll start each iteration by looking up the right page
table entry with `walkpgdir()`, with `alloc` set to true. Remember how that
function called `kalloc()`, which might fail, in which case it returns a null
pointer? Well that means we've gotta check for a null pointer here too. This
time however, we'll return -1 for failure and 0 for success, because why not?
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        pte_t *pte;
        if ((pte = walkpgdir(pgdir, a, 1)) == 0) {
            return -1;
        }
        // ...
    }
    return 0;
}
```

We're supposed to be allocating brand-new pages for this range of addresses, so
if a page has already been allocated, we'll just flip out in rage and panic.
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        // ...
        if (*pte & PTE_P) {
            panic("remap");
        }
        // ...
    }
    // ...
}
```

The last thing before checking the loop condition and updating `a` and `pa` is
to install the mapping to the right physical address with the right permissions
in the page table. Then we're done!
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        // ...
        *pte = pa | perm | PTE_P;
        // ...
    }
    // ...
}
```

Cool, now we have a way to map new pages into a page directory. We're well on
our way to leaving poor old `entrypgdir` behind for the shiny new `kpgdir`.

### kmap

Each process is gonna have its own page directory, so its mappings in the lower
half of the virtual address space might be totally different from those of
another process. But the mappings in the higher half (where the kernel lives)
will always be the same -- that way, the kernel can always use the existing page
directory for whatever process it happens to be running. We'll only use `kpgdir`
when the kernel isn't currently running a process, e.g. while it's running the
scheduler.

So when we create a new process, we'll need to copy in all the mappings that the
kernel expects to find into a fresh page directory for that process. Those are:
memory-mapped I/O device space from physical address 0 to 0x10_0000 (the boot
loader is also here, but we don't need it any more), kernel code and read-only
data from 0x10_0000 to the physical address of `data` (one of the symbols
defined in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld)), kernel data the rest of physical memory from there to
`PHYSTOP`, and more I/O devices from 0xFE00_0000 and up. Each of these ranges
needs its own permissions too.

We'll represent each of these mappings with a `struct kmap`, which has fields
for the starting virtual address, the starting and ending physical addresses,
and the permissions; then the mappings will get stored in a static global
variable `kmap`... oh come on, what fresh hell is THIS?
```c
static struct kmap {
    void *virt;
    uint phys_start;
    uint phys_end;
    int perm;
} kmap[] = {
    { (void *)KERNBASE, 0, EXTMEM, PTE_W },
    { (void *)KERNLINK, V2P(KERNLINK), V2P(data), 0 },
    { (void *)data, V2P(data), PHYSTOP, PTE_W },
    { (void *)DEVSPACE, DEVSPACE, 0, PTE_W }.
};
```

Okay, there are a few things going on here. First, the `static` keyword for a
variable means that variable has a single fixed location in memory that it's
never gonna move out of.

Then it does that thing again where we simultaneously define a `struct` type and
define a variable of that type. So the type is
```c
struct kmap {
    void *virt;
    uint phys_start;
    uint phys_end;
    int perm;
};
```

So then the static global variable `kmap` is an array of `struct kmap`s. I guess
we ran out of names or something. The array has four entries, and since each one
is a `struct`, it needs curly braces around it.

The first entry (for the lower of the two memory-mapped I/O device regions) has
a `virt` field of `KERNBASE`, a `phys_start` field of 0, a `phys_end` field of
`EXTMEM` (defined as 0x10_0000), and permission flag `PTE_W`. So it maps a
virtual address range starting at `KERNBASE` to the physical address range from
0x0 to 0x10_0000 and makes it writeable so we can communicate with the devices
there. The next two entries are similar, except that the kernel code isn't
writeable.

The last entry has `phys_start` of 0xFE00_0000 and a `phys_end` of 0. That's a
little strange, but it's because we want to map all the way up to the end of the
virtual address space at 0xFFFF_FFFF. The end should be one byte past that, but
it's impossible to represent 0x1_0000_0000 with 32 bits. Setting the end to 0
makes the size calculation (`phys_end - phys_start`) work out nicely: it'll just
overflow to the right number. This is okay since we're using unsigned integers,
but note that *signed* integer overflow is undefined behavior and thus VERY BAD
and the cause of many security vulnerabilities.

Okay, back to getting rid of `entrypgdir`!

### setupkvm

This function sets up a fresh new page directory with all the mappings in `kmap`
in order to please the kernel when it encounters the page directory. So needy,
right?

It takes no arguments and returns a pointer to the new page directory. First,
let's allocate a page of memory to hold the new directory. We'll be good and
remember to check for null (in which case we return null too) and clear the page
of the garbage values we wrote when we freed it.
```c
pde_t *setupkvm(void)
{
    pde_t *pgdir;
    if ((pgdir = (pde_t *) kalloc()) == 0) {
        return 0;
    }
    memset(pgdir, 0, PGSIZE);
    // ...
}
```

The upper end of virtual memory after `DEVSPACE` has I/O devices, so `PHYSTOP`
should be below that; this is as good a place as any to make sure.
```c
pde_t *setupkvm(void)
{
    // ...
    if (P2V(PHYSTOP) > (void *) DEVSPACE) {
        panic("PHYSTOP too high");
    }
    // ...
}
```

Finally, we'll add all the mappings in `kmap` above into this page directory so
the kernel is happy. We'll use `mappages()`, which returns -1 if it fails, so
we should check for that. The `freevm()` function is defined below, and we'll
get to it soon, but for now just know that it gets rid of all the mappings we
just made, in case any of them fails.
```c
pde_t *setupkvm(void)
{
    // ...
    struct kmap *k;
    for (k = kmap; k < &kmap[NELEM(kmap)]; k++) {
        if (mappages(pgdir,
                k->virt,
                k->phys_end - k->phys_start,
                (uint) k->phys_start,
                k->perm) < 0) {
            freevm(pgdir);
            return 0;
        }
    }
    return pgdir;
}
```
Let's check out that for loop: `k` is a pointer to a `struct kmap`, and `kmap`
is an array of `struct kmap`s; in C, arrays decay to pointers, so they have the
same type. `k` starts off pointing to the first (zero) entry of `kmap`. Then
incrementing it with `k++` shifts its value by the size of a `struct kmap`, so
it'll point to the next entry. The loop stops when `k` points beyond the last
entry of `kmap`, as determined by the `NELEM()` macro which counts the number of
entries in an array. Note that array element-counting only works in C if the
array is defined in the same function or as a global variable in the same file,
which is why it's so easy to do an out-of-bounds read or write in C (yet another
common security vulnerability).

Finally, if everything worked out okay, we return a pointer to the new page
directory.

### switchkvm

We said above that the kernel would usually just use the page directory of the
currently-running process, but it'll use `kpgdir` when no process is running,
i.e. during the kernel setup and while it's scheduling a new process. So we need
a way to tell the paging hardware to load `kpgdir` into register `%cr3`, which
holds a pointer to the page directory. That's this function.

It's a one-liner: get the physical address of `kpgdir` and stick it in `%cr3`
with the assembly instruction `lcr3`.
```c
void switchkvm(void)
{
    lcr3(V2P(kpgdir));
}
```

### kvmalloc

FINALLY, we're here! We're gonna get rid of `entrypgdir`! The kernel's `main()`
calls this function right after `kinit1()`.

We already did all the hard work, so this one's a breeze: we call `setupkvm()`
to allocate a new page directory and fill it with the kernel's mappings, then
call `switchkvm()` to load it into the paging hardware.
```c
void kvmalloc(void)
{
    kpgdir = setupkvm();
    switchkvm();
}
```

And we're DONE! Take that, `entrypgdir`, we don't need you anymore. We're big
kids now.

## Summary

So far, it's been a serious odyssey just to move from no paging in the boot
loader, to super basic paging with `entrypgdir` in [entry.S](https://github.com/mit-pdos/xv6-public/blob/master/entry.S), to `kpgdir` now.
Along the way, we've looked at code to allocate and free pages and install new
mappings in page directories and page tables. That'll come in handy when we look
at processes next; the virtual memory story still isn't over.

Also, note that `kpgdir` still isn't at the height of its powers: at the point
when `main()` calls `kvmalloc()`, the free list only contains pages for physical
memory between 0 and 4 MB. The rest will have to wait until `kinit2()` unleashes
its full potential. (Maybe some self-actualization seminars would help...)

# More Paging: The User Side

It's almost time to turn to interrupts and processes so we can figure out how to
work that sweet multiprocessing magic, but unfortunately we have some last
pieces of paging to wrap up before we can get there.

I know, we've been talking about virtual memory for what feels like a century
now, but so far everything we've done has been on the kernel side, allocating
pages and creating new page directories with the same kernel mapping. But what
about the lower half of the virtual address space, where user processes live?

This post will go through the rest of
[vm.c](https://github.com/mit-pdos/xv6-public/blob/master/vm.c)
and set up the paging-related machinery we'll need to run processes later on.

## Detour: Starting a New Process

When xv6 runs a new process, it will create a brand new virtual memory space for
it with a fresh page directory. We haven't talked about processes in xv6 yet, so
you might wonder how a process gets started up in the first place.

Let's forget all about xv6 for a second and think about another Unix-like OS:
Linux. How do we start a process there? Okay, we also have to forget about GUI
applications there. Let's just say you want to run some C code (xv6 maybe?) that
you've just compiled; what happens when you run it from the terminal?

Hopefully, you've done the OSTEP project called [processes-shell](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/processes-shell)
by now, so you know the answer; if you haven't, I recommend doing that one right
now before I give it away. (It's not strictly required, but are you really the
kind of person who loves getting movies spoiled for them?)

Okay, are you done?

The answer: it's just an `exec()` system call! The shell finds the executable
file in the file system, calls `fork()` to create a new child process, which
then calls `exec()` to transform itself into the program you want to run.

We'll get to these system calls later, so for now let's just go over the broad
strokes as they relate to virtual memory. `fork()` works by taking the parent
process's virtual memory space and making a copy of it for the child process.

`exec()` allocates a new page directory, figures out how much memory the new
program will need when it runs, then grows the virtual memory space allocated in
that new page directory to the required size. Then it loads the program into
memory in the new page directory.

Next, `exec()` skips a page, leaving it mapped but user-inaccessible; then the
next page becomes the process's stack. Why that empty page? It's an important
one for protection: that way, user programs that blow their stack will trigger a
page fault or a general protection fault instead of possibly overwriting random
code.

Then `exec()` copies some arguments into the stack before it switches to using
the new page directory and gets rid of the old one it had before.

Whew, okay, that's a lot of code to go over later, and that's only the virtual
memory part of the story. So let's just make it easier by doing all the work we
can right now. According to the above, we have to understand how xv6 does all of
the following:
* Makes a copy of a whole page directory,
* Creates a new page directory,
* Grows (or shrinks) the virtual memory space of a page directory,
* Loads program code into a page directory,
* Makes a page inaccessible to users,
* Copies stuff into a page in a page directory,
* Switches to a new process page directory, and
* Gets rid of an unused page directory.

Finally, there's one edge case to think about: running the very first process.
We obviously need to start running a shell at some point, so we need a special
way to get that started too, so it can in turn run other processes.

## vm.c, Again

We're gonna need some new functions! Actually, we already finished one of the
requirements -- `setupkvm()` can allocate a new page directory and set up the
kernel portion too. `switchkvm()` lets us switch to using `kpgdir` as a page
directory, but now we need to switch *away* from that to a page directory for a
process, so that'll be `switchuvm()`.

`copyuvm()` creates a copy of an entire page directory for a child process.
`allocuvm()` and `deallocuvm()` grow and shrink the virtual memory space that's
allocated in a page directory, and `freevm()` clears a page directory we no
longer need.

`loaduvm()` will load program code into a page directory; `clearpteu` makes a
page inaccessible to users, and `copyout()` copies data into a page in a page
directory. `inituvm()` handles the special case of setting up the page directory
for the very first process that xv6 will run.

The rest of this post will go over those functions one by one so we can be done
with virtual memory, but I know it's a little strange to go through a million
helper functions when we haven't seen the code that's gonna use them yet, so if
you'd prefer, you can come back to this after reading about processes and system
calls.

### deallocuvm

The arguments for this function are a page directory, the process's old size,
and the new size we want to shrink it down to; it'll return the process's new
size. By "shrinking" a virtual memory space, we really mean making sure that the
page directory only allocates up to `newsz` worth of pages. So if we think of
the sizes as virtual addresses, then the page directory currently maps the
virtual space from 0 to `oldsz`, so we should free everything between `newsz`
and `oldsz`, leaving behind the space from 0 to `newsz`.

First, we should make sure the new size is actually smaller than the old one;
otherwise trying to "shrink" down to the new size might cause integer overflow.
There; the sizes are both unsigned integers here, so at least it wouldn't be
that scary boogeyman of undefined behavior, but it could still be bad: 0 would
wrap around to 2^32 - 1, so "shrinking" to the new size would actually grow the
process way beyond what physical memory could handle.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    if (newsz >= oldsz) {
        return oldsz;
    }
    // ...
}
```

We're gonna shrink the physical memory allocated to this page directory by
freeing pages until we reach the new size. Let's start with the first page above
`newsz`; we can get its virtual address by rounding up `newsz` to a page
boundary.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    uint a = PGROUNDUP(newsz);
    // ...
}
```

Now we'll just iterate over the pages between `a` and `oldsz` one at a time and
free them. This is a little tricky: `kfree()` takes a virtual address (cast to a
`char *`), but it should be a *kernel* virtual address in the higher half, not a
user virtual address. Luckily, we already have `walkpgdir()`, which can take an
arbitrary virtual address and return its page table entry, so that's a good
start.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldz; a += PGSIZE) {
        pte_t *pte = walkpgdir(pgdir, (char *) a, 0);
        // ...
    }
    // ...
}
```

The page table entry contains the page's physical address, plus some flags to
determine whether it's mapped and what permissions are set for it.

Now, a virtual address space isn't laid out contiguously. Think about it: if you
sit back and imagine a user process hanging out in memory, what does that
address space look like? You're probably imagining the stack at one end of
memory and the heap at the other, with each growing toward the center, right?
so there will be some pages in the center that aren't mapped; some of the page
tables might not exist either, in which case `walkpgdir()` would return a null
pointer.

Remember we agreed to never dereference null pointers? Yeah, so we'll have to
skip all those unmapped pages. If we got a null pointer, then that means the
entire page table doesn't exist, so we need to skip forward to the next page
directory entry (and thus the next page table). We'll have to move `a` to the
virtual address that corresponds to that next page directory entry.

We can get the page directory index from `a` with the `PDX()` macro we've seen
seen before, and then just add 1 to get the next entry in the page directory.
Now we need to turn that back into a virtual address. We'll use a new macro,
`PGADDR()` (also from [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h)),
to do that. So then we'll continue to the next loop iteration, which will get
the page table entry for this new virtual address.

Wait wait wait, one last thing! After all that, `a` should now be the first
virtual address in the page table for the new page directory entry... except
it's get `PGSIZE` added to it because of the for loop's update statement.

Ugh, okay, fine, this is annoying. Let's just fix it with a hack: subtract
`PGSIZE` from it now, so that it gets incremented to the right value in the next
iteration. Okay, that's it, I swear!
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldsz; a += PGSIZE) {
        pte_t *pte = walkpgdir(pgdir, (char *) a, 0);

        if (!pte) {
            a = PGADDR(PDX(a) + 1, 0, 0) - PGSIZE;
        } else {
            // ...
        }
    }
    // ...
}
```

Okay, now the else branch: if we don't get a null pointer then at least the page
table exists, but that doesn't mean the page itself is mapped. If it's not, then
we don't need to do anything else, but if it is mapped, then we need to free it.
We can get the page's physical address out of the page table entry with the
`PTE_ADDR` macro then make sure it's not null.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldsz; a += PGSIZE) {
        // ...
        if (!pte) {
            // ...
        } else if ((*pte & PTE_P) != 0) {
            uint pa = PTE_ADDR(*pte);
            if (pa == 0) {
                panic("kfree");
            }
            // ...
        }
    }
    // ...
}
```

The whole point of this was to be able to call `kfree()`, remember? So let's
convert `pa` to a kernel virtual address as a `char *` and free it. Then after
the loop is done, we'll return the new size.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldsz; a += PGSIZE) {
        // ...
        if (!pte) {
            // ...
        } else if ((*pte & PTE_P) != 0) {
            // ...
            char *v = P2V(pa);
            kfree(v);
            *pte = 0;
        }
    }
    return newsz;
}
```

### allocuvm

This is the reverse of `deallocuvm()`: instead of freeing pages with `kfree()`,
we'll allocate them with `kalloc()`. Here too, we start by checking for integer
overflow by making sure `newsz` really is larger than `oldsz`. But now we also
have to check that we're not gonna grow the process's size into the region where
it could access kernel memory; otherwise it might read or modify arbitrary
physical memory.
```c
int allocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    if (newsz >= KERNBASE) {
        return 0;
    }
    if (newsz < oldsz) {
        return oldsz;
    }
    // ...
}
```

We're gonna start adding new pages right after `oldsz`, so we have to align that
to a page boundary:
```c
int allocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    uint a = PGROUNDUP(oldsz);
    // ...
}
```

The for loop is easier this time around because we already know that the pages
aren't mapped. First we allocate a new page. Any call to `kalloc()` needs two
things after, remember? We have to check for null, in which case we print an
error message to the console (that's `cprintf()`; we'll get to that in the
devices section), then undo any allocations we made and return 0. Then we have
to zero the page because we filled it with 1s when it was freed.
```c
int allocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < newsz; a += PGSIZE) {
        char *mem = kalloc();
        if (mem == 0) {
            cprintf("allocuvm out of memory\n");
            deallocuvm(pgdir, newsz, oldsz);
            return 0;
        }

        memset(mem, 0, PGSIZE);
        // ...
    }
    // ...
}
```

We have a page now, but it's not yet mapped in the page directory. We can do
that with `mappages()`; that might fail too (because it needs to allocate more
pages for the page tables), in which case we do the same as before. Then after
the for loop is done, we return the new size.
```c
int allocuvm(pde_t *pgdir), uint oldsz, uint newsz) {
    // ...
    for (; a < newsz; a += PGSIZE) {
        // ...

        if (mappages(pgdir, (char *) a, PGSIZE, V2P(mem), PTE_W | PTE_U) < 0) {
            cprintf("allocuvm out of memory (2)\n");
            deallocuvm(pgdir, newsz, oldsz);
            kfree(mem);
            return 0;
        }
    }
    return newsz;
}
```

### freevm

This function will get rid of a user page directory that we no longer need. Now
that we have `deallocuvm()`, it's easy: we just "shrink" the process to a size
of zero. Oh and we'll remember the lessons our ancestors have taught us and make
sure the pointer to the page directory isn't null before dereferencing it.
```c
void freevm(pde_t *pgdir)
{
    if (pgdir == 0) {
        panic("freevm: no pgdir");
    }
    deallocuvm(pgdir, KERNBASE, 0);
    // ...
}
```

Great, so all pages are freed, and we're done!

Now hang on a sec... The page directory itself resides in memory; so do the page
tables. We have to free those too. We'll start with the page tables; freeing the
page directory first would be a use-after-free vulnerability because we'd need
to use it to get to the page tables.

We'll iterate over the page directory's entries, checking whether each one has
the "present" flag set (`NPDENTRIES` is defined as 1024 in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/types.h)).
If it does, we'll get the page table's physical address from it with the
`PTE_ADDR()` macro, then convert that to a virtual address as a `char *` to make
`kfree()` happy. We don't have to worry about clearing the "present" flag in the
page directory because it's about to be freed anyway.
```c
void freevm(pde_t *pgdir)
{
    // ...
    for (uint i = 0; i < NPDENTRIES; i++) {
        if (pgdir[i] & PTE_P) {
            char *v = P2V(PTE_ADDR(pgdir[i]));
            kfree(v);
        }
    }
    // ...
}
```

We wrap up by freeing the page directory itself.
```c
void freevm(pde_t *pgdir)
{
    // ...
    kfree((char *) pgdir);
}
```

### copyuvm

The `fork()` system call will need to "clone" a process, which includes its
virtual address space. This function takes a pointer to the parent process's
page directory and the size of the parent process's address space and returns a
pointer to a fresh new page directory with everything set up exactly the same.

We start by creating a new page directory and taking care of the kernel's half
of the address space with `setupkvm()`. That might fail if it can't allocate a
new page, so we have to check for null. Sigh. C code is approximately 40%
checking for null return values.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    pde_t *d;
    if ((d = setupkvm()) == 0) {
        return 0;
    }
    // ...
}
```

Now we'll iterate over the user portion of the parent process's address space
from 0 to `sz`, copying everything over as we go. Say we want to copy a page
from the parent's virtual address `i` to the child's address `i` (note that
they'll map to different physical addresses). We'll have to figure out the
corresponding kernel virtual address for the parent's `i` in order to do that,
so we use `walkpgdir()` to get the page table entry, then get the page's
physical address.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        pte_t *pte;
        if ((pte = walkpgdir(pgdir, (void *) i, 0)) == 0) {
            panic("copyuvm: pte should exist");
        }
        if (!(*pte & PTE_P)) {
            panic("copyuvm: page not present");
        }

        uint pa = PTE_ADDR(*pte);
        uint flags = PTE_FLAGS(*pte);
        // ...
    }
    // ...
}
```
In this case we know the parent process is already set up, so we don't really
have to worry about `walkpgdir()` failing and returning null, but it's bad C
juju to ignore a possibly-null return value, so we just panic if it does fail or
if the page isn't present.

Next we allocate a page for the child process (checking for null again...) and
copy everything from the parent's page to the new child page.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        char *mem;
        if ((mem = kalloc()) == 0) {
            goto bad;
        }
        memmove(mem, (char *) P2V(pa), PGSIZE);
        // ...
    }
    // ...
}
```
You might recognize `memmove()` as a C standard library function that copies the
contents of one memory address into another, but we can't use those, remember?
So xv6 provides its own implementation of it in
[string.c](https://github.com/mit-pdos/xv6-public/blob/master/string.c).

If you haven't seen a `goto` statement before, it's basically a holdover from ye
olde days before Edsger Dijkstra preached the gospel of structured programming
to the world and invented the if statement. It does exactly what it sounds like:
you make a label somewhere in code and it takes you there.

Next we stick that new page into the child's page directory, checking for null
again. If `mappages()` fails, then the new page won't be in the page directory,
so we have to free it here or else we'll never be able to find it again: a
memory leak.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        if (mappages(d, (void *) i, PGSIZE, V2P(mem), flags) < 0) {
            kfree(mem);
            goto bad;
        }
    }
    // ...
}
```

If none of the allocations failed, we just return a pointer to the new page
directory. But if something went wrong, then one of those `goto` statements will
send us to the time out corner of `bad`, where we undo all our work by freeing
the page directory and returning a null pointer.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    return d;

bad:
    freevm(d);
    return 0;
}
```
Great, another function we'll have to check for null.

### switchuvm

Okay, we've got a way to create a new process page directory. We also have a way
to switch to using the kernel page directory `kpgdir` with `switchkvm()`. But we
need a way to switch to using the process page directory too. Enter `switchuvm()`.

I'll warn you -- `switchkvm()` was nice and short, but `switchuvm()` is an ugly
one for sure.

The argument to this function is a pointer to a `struct proc`, which represents
a process. We'll talk about that more when we get to processes; two fields are
important now: `p->kstack` which holds a pointer to the kernel stack for that
process, and `p->pgdir`, which points to that process's page directory.

Okay, well let's start with some sanity checks to make sure that the process `p`
actually exists (the pointer is non-null) and its kernel stack and page directory
pointers are non-null too.
```c
void switchuvm(struct proc *p)
{
    if (p == 0) {
        panic("switchuvm: no process");
    }
    if (p->kstack == 0) {
        panic("switchuvm: no kstack");
    }
    if (p->kstack == 0) {
        panic("switchuvm: no pgdir");
    }
    // ...
}
```

The main function of loading the process's page directory will be the same as in
`switchkvm()`: just an `lcr3` instruction. But the difference now is that the
x86 architecture requires some additional bookkeeping for processes.

See, when the kernel runs a new process, the CPU will start executing different
instructions. But it needs a way to keep track of where it left off in the
kernel code so that it can pick the thread back up after the process is done
executing. Similarly, interrupts and system calls might change the running
process, so the CPU needs to record some metadata about the process's state too
before switching to another one. x86 does that by means of a structure called a
*Task State Segment*, or TSS.

The TSS holds information like the current state of certain registers (e.g.,
`%esp`, `%eip`, `%cr3`, etc.), segment descriptors (`%cs`, `%ss`, `%ds`, etc.),
the current privilege leve, and I/O privilege levels -- in other words, the
process's *context*. It can be located anywhere in memory, but the processor
needs to find it, so it uses an entry in the GDT called the TSS segment
descriptor that points to the TSS. Remember the GDT from way back when we were
talking about segmentation? Good times. The CPU holds a pointer to the GDT's TSS
entry in a special register called the task register.

Back in the segmentation days of our youth, we stored the GDT in a `struct cpu`
that held information about the current processor. We got that `struct cpu` by
calling a `mycpu()` function. We're gonna do the same thing here in order to
update the GDT with a segment for the TSS. Getting interrupted in the middle of
this might be disastrous: the TSS would be half-updated, so who knows what would
happen when the CPU tried to resume execution where it last left off. So we'll
use the `pushcli()` and `popcli()` functions we saw with spin-locks to temporarily
disable interrupts.
```c
void switchuvm(struct proc *p)
{
    // ...
    pushcli();

    mycpu()->gdt[SEG_TSS] = SEG16(STS_T32A, &mycpu()->ts, sizeof(mycpu()->ts)-1, 0);
    mycpu()->gdt[SEG_TSS].s = 0;
    // ...

    popcli();
}
```
Whoa okay what is this?

We've seen the `SEG()` and `SEG_ASM()` macros before; they created GDT segments.
`SEG16()` does the same with 16 bits (it's defined in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h)). `STS_T32A`
is a flag that sets the segment's type as an available 32-bit TSS. Then we pass
in a pointer to the task state with `&mycpu()->ts`, its size, and a descriptor
privilege level of 0 (which means ring 0, the kernel level). The GDT's `.s`
field is a one-bit flag to determine whether this is a system or application
segment, so we set it to system.

Okay, so now the GDT points to the task state. Next we need to update the task
state, then load it into the CPU. We'll start by storing a segment selector and
the stack pointer in the task state; these should look familiar from the boot
loader and `seginit()`.
```c
void switchuvm(struct proc *p)
{
    // ...
    mycpu()->ts.ss0 = SEG_KDATA << 3;
    mycpu()->ts.esp0 = (uint) p->kstack + KSTACKSIZE;
    // ...
}
```

The TSS can also specify permissions for accessing I/O ports: for example,
setting the I/O privilege level to 0 in the `eflags` register *and* setting a
part of the TSS called the I/O map base address to an address beyond the TSS
segment forbids I/O instructions like `inb` and `outb` from user space. So we'll
set the I/O map base address next.
```c
void switchuvm(struct proc *p)
{
    // ...
    mycpu()->ts.iomb = (ushort) 0xFFFF;
    // ...
}
```

So now we have a GDT entry pointing to the TSS, which is now updated. Now we
just load it into the task register with the x86 instruction `ltr`; here we use
a C wrapper for that assembly instruction, defined in
[x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h).
```c
void switchuvm(struct proc *p)
{
    // ...
    ltr(SEG_TSS << 3);
    // ...
}
```

Finally, the last thing we do before re-enabling interrupts is to load the
process's page directory into the `%cr3` register so we can start using it.
```c
void switchuvm(struct proc *p)
{
    // ...
    lcr3(V2P(p->pgdir));
    // ...
}
```

### loaduvm

Okay, this is another function that's gonna require extra info we haven't seen
yet, but I'm gonna make it a bit easier by waving my hands around and glossing
over the details. It's gonna read a program from a file into memory at virtual
address `addr` using page directory `pgdir`. The part we want to read has size
`sz` and is located at position `offset` within the file.

Now, what about the file? We'll talk more when we get to the file system code,
but for now let's just say that files are represented in xv6 as `struct inode`s,
and we can read from them with the function `readi()`.

We're gonna run the program from this code, so the address it's stored in needs
to be page-aligned.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    if ((uint) addr % PGSIZE != 0) {
        panic("loaduvm: addr must be page aligned");
    }
    // ...
}
```

Next we're gonna iterate over pages starting from `addr`, reading from the file
in `ip` into that page. As usual, we'll need to get the kernel virtual address
from the user address `addr`, so we start by getting the page table entry via
`walkpgdir()`, checking for a null pointer if the corresponding page table
doesn't exist. Then we can turn that into a physical address.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // Get the page table entry
        pte_t *pte;
        if ((pte = walkpgdir(pgdir, addr + i, 0)) == 0) {
            panic("loaduvm: address should exist");
        }
        // Get the page's physical address
        uint pa = PTE_ADDR(*pte);

        // ...
    }
    // ...
}
```

Now we want to read from the file one page at a time using `readi()`, which
takes a pointer to an inode (here, `ip`), a kernel virtual address (`P2V(pa)`),
the location within the file of the segment we want to read (`offset + i`), and
the segment's size.

Now we want to read from the file one page at a time using `readi()`. We have to
specify a size in bytes to read; if the remaining unread part of the segment is
larger than a page, then the size we pass to `readi()` should be `PGSIZE`, but
otherwise it'll be less. So we'll compare `sz` to `i` and define define `n`
accordingly.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        uint n;
        if (sz - i < PGSIZE) {
            n = sz - i;
        } else {
            n = PGSIZE;
        }
        // ...
    }
    // ...
}
```

The other arguments to `readi()` are a pointer to an inode (`ip`), a kernel
virtual address (`P2V(pa)`), and the location within the file of the segment we
want to read (`offset + i`). It returns the number of bytes read, so if it's not
`n` we'll report an error by returning -1. Otherwise we return 0 after the for
loop is done.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        if (readi(ip, P2V(pa), offset + i, n) != n) {
            return -1;
        }
    }
    return 0;
}
```

### inituvm

Okay, the next three are nice and easy! This next one is pretty similar to
`loaduvm()`, except instead of loading program code from disk, it copies it in
from memory. We'll take `sz` bytes from a source address of `init` and stick it
in address 0 of the process's page directory `pgdir`.

This function is also easier because we're only gonna call it for programs that
are less than one page in size, so we don't have to worry about looping over
pages or anything like that. I like it when xv6 keeps things simple.
```c
void inituvm(pde_t *pgdir, char *init, uint sz)
{
    if (sz >= PGSIZE) {
        panic("inituvm: more than a page");
    }
    // ...
}
```

Next we allocate a fresh page of memory, zero it to clear the garbage values,
and stick it into `pgdir` at address 0.
```c
void inituvm(pde_t *pgdir, char *init, uint sz)
{
    // ...
    char *mem = kalloc();
    memset(mem, 0, PGSIZE);
    mappages(pgdir, 0, PGSIZE, V2P(mem), PTE_W | PTE_U);
    // ...
}
```

And we wrap up by actually loading the code from `init` into the new page.
```c
void inituvm(pde_t *pgdir, char *init, uint sz)
{
    // ...
    memmove(mem, init, sz);
}
```

### clearpteu

This function takes a page directory and a user virtual address and clears the
"user-accessible" flag so that the process can't touch it. It's used to create
an inaccessible page below a new process's stack to guard against stack
overflows; this way, a stack overflow will cause a page fault instead of
silently overwriting memory.

The `PTE_U` flag is in the page table entry, so we'll have to get that, then set
the flag.
```c
void clearpteu(pde_t *pgdir, char *uva)
{
    // Get the page table entry
    pte_t *pte = walkpgdir(pgdir, uva, 0);
    if (pte == 0) {
        panic("clearpteu");
    }
    // Clear the user permission flag
    *pte &= ~PTE_U;
}
```
Here `&` is a bitwise-AND and `~` is a bitwise-NOT; for reference, `|` is
bitwise-OR and `^` is bitwise-XOR. Contrast these with their logical versions,
`&&`, `!`, and `||` (XOR has no logical version). C also has corresponding
assignment operators (similar to `+=`, `-=`, `*=`, etc.) for each of them. So
the last line of code is equivalent to `*pte = *pte & (~PTE_U)`.

### uva2ka

We often need to convert user virtual addresses to kernel ones; `uva2ka()` is a
short helper function that does that while checking that the page is actually
present and has the user permission flag set.

We'll call walkpgdir to get the page table entry, then check both permission
bits before recovering the page address with `PTE_ADDR()` and converting it to a
kernel virtual address. We'll return the kernel virtual address as a `char *`,
or null if either flag is not set.
```c
char *uva2ka(pde_t *pgdir, char *uva)
{
    pte_t *pte = walkpgdir(pgdir, uva, 0);

    if ((*pte & PTE_P) == 0) {  // check that it's present
        return 0;
    }
    if ((*pte & PTE_U) == 0) {  // check that it's user-accessible
        return 0;
    }
    return (char *) P2V(PTE_ADDR(*pte));
}
```

Let me ask you a weird question: how are you feeling right now?

Okay, that was a test of your C coding practices, because if you took those null
checks to heart, you should be *really* uncomfortable right about now.

Check it out: `walkpgdir()` returns a pointer to the page table entry. *Any*
time a function returns a pointer, you should immediately ask yourself whether
that function can return a null pointer. Tons of C functions report an error by
returning null. In this case, we *know* `walkpgdir()` can fail and report null
if the page table doesn't exist, so we *know* we might get a null pointer out of
it -- it'll happen whenever a page table doesn't exist. So what do we do with
that knowledge?

Why, we go right ahead and dereference that pointer. WKBW;NQ39Q2A4T8YHMFGRW!!!

Dereferencing a null pointer is undefined behavior. There's literally no telling
what might happen. It can cause all kinds of bugs from segmentation faults to
security vulnerabilities.

All those null checks in the other functions serve a purpose: if something goes
wrong and a function returns a null pointer, they catch it before it gets
dereferenced, then either handle it gracefully or simply propagate the error by
returning null (or some other error code) and let the caller figure out what to
do with it.

Omitting a check for a null pointer like `uva2ka()` does is bad practice in C
because it means the programmer has to *guarantee* -- by manually checking --
that no call to this function could *ever possibly* cause a null return value.
Except humans are dumb, dumb creatures who make mistakes all the time, especially
in big projects: there's no way you'd be able to remember that tiny little
detail two years later when you decide to refactor your code or add a new
feature or something.

But maybe you can note that in the comments? Okay yeah, but think about it: how
often do you go and look up the source code for every single function you call?
Yeah, I thought so.

This is why C is so dangerous: there are hundreds of such problems that you need
to be aware of and remember to add stuff like null pointer checks to your code.
If you don't because you're a normal human who forgets things sometimes, then
you'll need to remember that you forgot to do it before and manually check every
single call to your code and think about every possible edge case that a
malicious adversary might exploit.

Good thing no one ever makes these mistakes in C, or we'd see enormous security
vulnerabilities being reported every single day in all kinds of critical
software. Oh wait...

So if you ever find yourself looking at C during code review and you come across
a function that returns a pointer, you should stop what you're doing and look up
the documentation for that function. If that function has any chance of
returning a null pointer, then you should yell and kick and scream until somebody
adds a null check and figures out how they want to handle it if it's null. Is
this annoying? Yes. Hard to remember? Yes. But that's C. *(cough cough use Rust
instead cough cough...)*

Now, the xv6 authors are so awesome that I'm gonna give them the benefit of the
doubt and assume they left it off because they hand-checked every call to make
sure it would never be an issue. But you and me? Nah.

The point of my rant is this: if you're reading this, then you're probably gonna
find yourself hacking away at xv6 for a project sooner or later. When you do
that, you should treat this function as VERBOTEN. You're not allowed to touch
it or call it, at least until you add a null check to it yourself.

The same goes for any functions that call this one, because maybe all the
existing calls to `uva2ka()` are fine right now, but then you make some tiny
change and now it's no longer guaranteed to never be null. For reference, this
function currently only gets called by `copyout()`, and that one only gets
called by `exec()`. `exec()` gets called by `sys_exec()`, the shell, and the
initial user-space program `init`. So be careful if you touch any of those.

Whew, okay, /rant.

### copyout

This function copies `len` bytes of data from a kernel virtual address `p` to a
user virtual address `va` using page directory `pgdir`. `exec()` will use this
to copy command-line arguments to the stack for a program it's about to run.

You might be wondering why it's needed -- doesn't `memmove()` do the same thing?
Almost, but the difficulty is that `pgdir` may not be the current page
directory, so we'll have to manually translate the virtual address `va`. That's
where `uva2ka()` comes in, plus it ensures that the page for `va` has the right
flags set. *Then* we can use `memmove()`.

First, `p` will be the source address, but `memmove()` requires a `char *` in
order to copy data byte-by-byte, so let's convert it now:
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    char *buf = (char *) p;
    // ...
}
```

Next we need to get the kernel virtual address corresponding to `va`, but
there's a challenge: what if the data crosses a page table boundary? It might be
spread across separate locations in physical memory (and thus in kernel virtual
memory too). So we'll need a loop in which each iteration gets the next kernel
virtual address and copies whatever part of the data is in this page.
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        // ...
        len -= n;
        buf += n;
        // ...
    }
    // ...
}
```

So we'll start each iteration by making `va0` the base address of the page `va`
is on and `pa0` the kernel address of `va0`, converted with `uva2ka()`. I...
honestly don't know why they used `pa0` as an identifier here. It makes it look
like it should be a physical address, but it's not; it's a kernel virtual
address. Sigh. Anyway, the call to `uva2ka()` might fail if the page isn't
present or it doesn't have a user permission bit, so we have to check for a null
pointer and return -1 if we find one.
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        uint va0 = (uint) PGROUNDDOWN(va);

        char *pa0 = uva2ka(pgdir, (char *) va0);
        if (pa0 == 0) {
            return -1;
        }

        // ...
        va = va0 + PGSIZE;
    }
    // ...
}
```

Now `va` is in between `va0` and the next page, so the length of the data within
this page is `PGSIZE - (va - va0)`, unless it's the last page, in which case we
should pick the lesser of this value and `len` (since `len` gets decremented on
each iteration through the loop).
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        // ...
        uint n = PGSIZE - (va - va0);
        if (n > len) {
            n = len;
        }
        // ...
    }
    // ...
}
```

Finally, we copy the data from `buf` into the target kernel virtual address for
`va`. Hmm, we don't have that yet. Oh wait, `pa0` is the kernel virtual address
for `va0`, and `va` is just `va-va0` bytes after that, so we'll use it.
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        // ...
        memmove(pa0 + (va - va0), buf, n);
        // ...
    }
    return 0;
}
```
We return 0 if everything went okay.

## Summary

Okay, that was a lot of helper functions, but we're ALL DONE with virtual
memory! From now on, we have all the tools we'll need to manage memory and set
up virtual address spaces for new processes.

# Processes

It's time to turn our attention to processes in xv6!
[proc.c](https://github.com/mit-pdos/xv6-public/blob/master/proc.c) is another
huge file, so I'm gonna split it up into a few posts. This one will focus on the
basic functions we'll need in order to create new processes; later posts will
go over scheduling and system calls.

## proc.h

I haven't spent much time on the header files in xv6, but
[proc.h](https://github.com/mit-pdos/xv6-public/blob/master/proc.h) defines some
important structures we're gonna be using often, so let's just get those out of
the way first.

Let's start off with the definition for `struct context`. The processor will
have to switch between different processes during interrupts, system calls,
exceptions, etc.; these *context switches* will require saving the contents of
some of the CPU registers so that it can reload them when it switches back and
resume execution where it left off. It'll save the process's context by pushing
those register contents on the stack; that way the stack pointer is effectively
a pointer to the context. So the fields of a `struct context` will just list all
the registers that were saved on the stack.

Now, which registers do we need to save? Let's look at the full list on the
[OSDev Wiki](https://wiki.osdev.org/CPU_Registers_x86). We've got some general-
purpose registers, the instruction pointer register `%eip`, segment registers,
a flags register, control registers, and the GDT and IDT registers (x86 doesn't
use the debug, test, or LDT registers).

The flags register, control registers, and GDT/IDT registers shouldn't change
between processes, so we don't need to save those. What about the segment
registers like `%cs`? Back when we set up segmentation, we made the segments be
identity maps that would always stay the same for all processes. There are
separate segments for user mode and kernel mode, but context switches will
always occur in kernel mode, so the segment registers shouldn't change, and we
don't need to save them either.

We should definitely save the program counter (AKA instruction pointer `%eip`),
since that will point to the place in the code where we should resume execution.

The only ones left now are the general-purpose registers: the stack base pointer
`%ebp` and stack pointer `%esp`, along with `%eax`, `%ebx`, `%ecx`, `%edx`,
`%esi`, and `%edi`. We said above that the stack pointer `%esp` would tell us
where to find the context, so that must mean we'll already have it through some
other means in order to find the rest of the context, so we don't need to save
it again (we'll see how we end up getting it later on). But we do need to save
`%ebp`.

There's an x86 convention that the caller of a function has to save `%eax`,
`%ecx`, and `%edx`, so those are already taken care of. So we'll just save the
others: `%edi`, `%esi`, and `%ebx`.

We end up with this list of saved registers as the fields for `struct context`:
```c
// ...
struct context {
    uint edi;
    uint esi;
    uint ebx;
    uint ebp;
    uip eip;
};
// ...
```

Next up: we might end up with a bunch of processes, some of which are currently
running while others aren't. Let's set up some labels to note that. We'll
definitely need a `RUNNING` label; we'll also use one called `RUNNABLE` for
processes that are ready to be run the next time there's a free CPU. We also
need a label for processes that are blocked waiting for something else to happen
(e.g., I/O); xv6 calls this `SLEEPING`. Processes that don't exist yet will be
called `UNUSED`.

There are two special moments in a process's lifecycle that we should be careful
with: birth and death. When we create a new process, we'll have to do a bunch of
setup before it's `RUNNABLE`; killing a process requires clean-up before it goes
back to `UNUSED`. We'll call those `EMBRYO` and `ZOMBIE`, respectively.

We could use bit flags for these states or just regular integers, except then
we'd have to do annoying bit arithmetic or keep track of which number represents
which state. And yes, we could use a bunch of `#define` directives for the
preprocessor for that, but there's a better way to do it. C lets us create data
types for labels using `enum`s. These don't have fields like `struct`s do;
they're basically just a mapping between integers and what the labels those
integers represent. So it's pretty similar to using a bunch of `#define`
directives, except that they're all defined neatly in a single place, so it
helps us remember they're all representing the same idea. So we'll use an `enum`
like this:
```c
// ...
enum procstate { UNUSED, EMBRYO, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
// ...
```

Now it's time to look at how we'll represent processes themselves together with
their metadata. Let's see... what kind of unique data does each process have?
We just talked about `struct context`s and `enum procstate`s; each process will
have both of those.

We also talked about virtual memory for processes in a previous post, so it
should also have its own page directory and stack for the kernel to use, plus a
way to track the size of its virtual address space. We said then that processes
are created using `fork()`, so let's add a field to point to the parent process.

We'll need a way for the kernel to refer to a process, so let's give it a unique
process ID. That's not super helpful when it comes to debugging, so let's also
add a name for it as a string.

The rest of the fields are for aspects we haven't seen yet but will talk about
soon: a *trap frame* for interrupts and system calls, a boolean to track whether
a process should be killed soon, a *channel* to be able to wake up a sleeping
process, an array of open files, and a current working directory.
```c
// ...
struct proc {
    uint sz;                    // size (in bytes) of virtual address space
    pde_t *pgdir;               // page directory
    char *kstack;               // kernel stack for this process
    enum procstate state;       // process state
    int pid;                    // process ID
    struct proc *parent;        // parent process
    struct trapframe *tf;       // trap frame for current system call
    struct context *context;    // saved register contents for context switches
    void *chan;                 // channel that process is sleeping on, if any
    int killed;                 // boolean: should process be killed soon?
    struct file *ofile[NOFILE]; // array of open files
    struct inode *cwd;          // current working directory
    char name[16];              // process name
}
// ...
```

Okay, next we'll add another structure for metadata representing each CPU.

If you read the previous post, then you know each CPU has its own local
interrupt controller with a unique ID, so we'll write that down. The post about
process paging talked about the TSS, so we'll need one of those per CPU, plus a
GDT too.

At any point in time, a processor will be running one of: its own initialization
routine (only once while the kernel is setting up), a user process (or any
interrupts or system calls that come up), or a scheduler routine to run the next
process. So let's add a pointer to a `struct proc`, which will be null if it's
not running a process; a boolean `started` will be false until the CPU finishes
its own set-up. The scheduler isn't itself a process; it uses the `kpgdir` page
directory and has its own context, so we'll store that context in a field here.

Finally: remember how the spin-lock post talked about nested calls to `pushcli()`
and `popcli()` tracking whether interrupts were enabled before the first call to
`pushcli()`, and only enabling interrupts after the last call to `popcli()` if
they were enabled before? Those were tracked with per-CPU fields `ncli` and
`intena`, so we need those too.
```c
struct cpu {
    uchar apicid;               // ID of this CPU's local interrupt controller
    struct context *scheduler;  // scheduler's context
    struct taskstate ts;        // task state segment
    struct segdesc gdt[NSEGS];  // global descriptor table
    volatile uint started;      // boolean: has this CPU been initialized yet?
    int ncli;                   // depth of pushcli() nesting
    int intena;                 // were interrupts enabled before pushcli()?
    struct proc *proc;          // currently running process
};
// ...
```

Last but not least, we'll add declarations for the global array of CPUs and the
number of CPUs actually present on this machine; these were defined in
[mp.c](https://github.com/mit-pdos/xv6-public/blob/master/mp.c)

Okay, on to the functions now!

## proc.c

xv6 uses a global process table with an array of processes to store all the
`struct proc`s in; this means we'll never be able to create more processes than
the number of entries in the array, `NPROC`, defined in
[param.h](https://github.com/mit-pdos/xv6-public/blob/master/param.h) as 64.
We'll need a lock too to prevent data races while accessing the process table.
The process table's definition does that thing again where you simultaneously
define a `struct type` and define a variable using that type in a single
statement.
```c
struct {
    struct spinlock lock;
    struct proc proc[NPROC];
} ptable;
// ...
```

Then we define a global static variable to point to the first process that gets
run on xv6, so that other files can set it up.
```c
// ...
static struct proc *initproc;
// ...
```

Finally, we're gonna need to assign unique process IDs, so we'll use a global
counter to know which one we should use next.
```c
// ...
int nextpid = 1;
// ...
```

### pinit

This function only does one thing: initializes the lock in the process table.
```c
void pinit(void)
{
    initlock(&ptable.lock, "ptable");
}
```

### mycpu

This function will return a pointer to the `struct cpu` for the current CPU.
There's a potential concurrency bug with this function: if it gets interrupted
before it returns, then it might get rescheduled on a different CPU, and end up
returning an incorrect `struct cpu`. So we need to make sure that interrupts are
disabled when we call it. Normally we'd do that with `pushcli()` and `popcli()`,
but those functions actually call this one, so we'd get an infinite recursion.
So instead we're just gonna have to remember to disable interrupts *before*
calling this function.

If you're reading this because you're gonna do some xv6 kernel hacking for an
OSTEP project or something, you should read that as "DANGER DANGER DANGER!". If
your code calls this function, or calls any other functions that in turn call
this one, you *have* to make sure you've disabled interrupts first.

Concurrency bugs are a nightmare because they're not deterministic: for example,
if you forget to disable interrupts before calling this function, it might work
just fine most of the time until the one unlucky moment when it gets interrupted
and rescheduled on a different CPU. So let's make this easier to debug by
starting off with a check that interrupts are disabled and panic if they're not.
We can check whether the interrupt flag `FL_IF` is set in the `eflags` register.
```c
struct cpu *mycpu(void)
{
    if (readeflags() & FL_IF) {
        panic("mycpu called with interrupts enabled\n");
    }
    // ...
}
```

Okay so how do we figure out which CPU we're on? Well, the previous post talked
about interrupt controllers; each CPU has a local interrupt controller with a
unique ID which we can get with `lapicid()`. Once we have that, we can iterate
over the CPU array `cpus` until we find an entry with a matching `apicid`; we'll
just panic if none of them match.
```c
struct cpu *mycpu(void)
{
    // ...
    int apicid = lapicid();

    for (int i = 0; i < ncpu; ++i) {
        if (cpus[i].apidid == apicid) {
            return &cpus[i];
        }
    }
    panic("unknown apicid\n");
}
```

### cpuid

Those local interrupt controller IDs aren't guaranteed to start from 0, so we'll
need another way to identify CPUs. We can just use its entry number in the
global `cpus` array for that; `cpus` is an array of `struct cpu`s, which in C
means it's really a pointer to the entry with index 0. `mycpu()` returns a
pointer to the entry for the current CPU, so we can just subtract those pointers
to get the index.
```c
int cpuid(void)
{
    return mycpu() - cpus;
}
```

### myproc

This function returns a pointer to the `struct proc` running on this CPU. We're
gonna call `mycpu()` here, so we'll be good and remember to dsable interrupts
first with `pushcli()` and reenable them at the end with `popcli()`. Then we'll
get the current process from the `struct cpu`'s field.
```c
struct proc *myproc(void)
{
    pushcli();

    struct cpu *c = mycpu();
    struct proc *p = c->proc;

    popcli();
    return p;
}
```

### allocproc

Okay, we're finally at the code to create a new process! Whew, it's been a long
journey.

This is a `static` function, which means it can only be called by functions
defined in this same file. Creating a new process will require modifying the
process table, so we need to grab the lock so that other threads can't mess with
it while we're using it.
```c
static struct proc *allocproc(void)
{
    acquire(&ptable.lock);
    // ...
}
```

Now we need to look through the table and find a slot that's `UNUSED`; if we
find on, then great, we'll assign that slot to the new process after the `found`
label below. But if none of them are free, we'll have to return a null pointer
to indicate that. You know what that means, right? Yup, we're gonna have to add
null checks every time we call this function! Wooooo!
```c
static struct proc *allocproc(void)
{
    // ...
    struct proc *p;

    // Look through process table looking for an UNUSED slot
    for (p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state == UNUSED) {
            goto found;
        }
    }
    // If none is found, return null pointer
    release(&ptable.lock);
    return 0;

    // ...
}
```
Check out that for loop too: `p` is a pointer to a `struct proc` that starts off
pointing to `ptable.proc`; that means it points to the entry and index 0. Then
it gets incremented by 1 each iteration; since it's a `struct proc`, the pointer
arithmetic will work out so that it points to the next entry in the process
table.

Okay now let's check out the `found` label and see what happens if we did find
an unused slot. First we set its state to `EMBRYO` (instead of `RUNNABLE`, since
we're not done setting it up) and give it a PID. That state means it's neither
`UNUSED` nor `RUNNABLE`, so we can be confident that any other threads wouldn't
try messing with it right now; they can't allocate the slot to another process,
and they can't try to run it yet. So we can stop hogging the process table now
and let other threads take a turn.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    p->state = EMBRYO;
    p->pid = nextpid++;

    release(&ptable.lock);

    // ...
}
```

Now we need to allocate a page for this process's kernel thread to use as a
stack. Remember, `kalloc()` can return null, so we need a null check here.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    if ((p->kstack = kalloc()) == 0) {
        p->state = UNUSED;
        return 0;
    }
    // ...
}
```

Now, we're not gonna set up its page directory yet; that'll happen in `fork()`,
which we'll see later on. But we do need to set up the process so that it'll
start executing code somewhere. It needs to start off in kernel mode, then it'll
context-switch back into user mode and start running its code.

We haven't looked at the mechanics of context switches yet, so I'll spoil it a
little now (I know, I'm sorry). When a process is already running, it can send a
system call to ask for the kernel's attention to do whatever it needs, like a
baby crying until it gets fed or changed or burped or whatever. Then it'll
switch into kernel mode to run the system call, then switch back to where it
left off and pick up from there.

Well, xv6 is all about simplicity, right? And what's more simple and elegant
than treating a special case (creating a new process and starting it off running
some code) the same as the general case (returning from a system call)? So xv6
will set up every new process to start off by "returning" from a (non-existent)
system call. That way the context switch code can be reused for new processes
too.

New processes are created via `fork()`, so we'll return into a function called
`forkret()`. Then that has to return into the function `trapret()`, which
closes out a *trap* (interrupt, system call, or exception) by restoring saved
registers and switching into user mode. We'll get to `forkret()` and `trapret()`
soon.

But first, the challenge: how do we "return" into a function that never called
us in the first place? We talked about function calls in x86 in the post on
spin-locks with the `getcallerpcs()` function, so make sure to read that now if
you need a refresher.

To summarize: when a function `f()` calls another function `g()`, it pushes the
arguments of `g()` on the top of its stack. Then it pushes a return address to
know where it should continue running the code of `f()` after `g()` returns;
that's just the `%eip` register. Then it pushes the base address of the stack
for `f()`, i.e. the current `%ebp` register. That's where `g()`'s stack will
start off.

When the scheduler first runs the new process, it'll check its context via
`p->context` to get its register contents, including the instruction pointer
`%eip`. So if we want it to start executing the code in `forkret()`, the `eip`
field of its context should point to the beginning of `forkret()`. Then we can
trick it into thinking that the previous caller was `trapret()` by setting up
arguments and a return address in its stack.

Let's start off by getting a pointer to the bottom of the stack. We had just
allocated a new stack page at `p->kstack`, but the stack grows from high to low
addresses, so the base of the stack is really at `p->kstack + KSTACKSIZE`. We'll
make it a `char *` so we can move around one byte at a time using pointer
arithmetic.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    char *sp = p->stack + KSTACKSIZE;
    // ...
}
```

Now we should push any arguments for `trapret()` on the stack; it takes a
`struct trapframe` (which we'll go over later), so we'll leave some room for it
and make the process point to it with `p->tf`.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    sp -= sizeof(*p->tf);
    p->tf = (struct trapframe *) sp;
    // ...
}
```

Then we add a "return address" to the beginning of `trapret()` after that.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    sp -= 4;
    *((uint *) sp) = (uint) trapret;
    // ...
}
```

The last thing we need is to save some space for the process's context on the
stack and point `p->context` to it. Then we'll zero it all out, except for the
`eip` field, which will point to the beginning of `forkret()`. And that's it!
We just return the pointer to the process now.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    sp -= sizeof(*p->context);
    p->context = (struct context *) sp;

    memset(p->context, 0, sizeof(*p->context));
    p->context->eip = (uint) forkret;

    return p;
}
```

We can create new processes now!

### growproc

What about growing or shrinking the size of a process's address space? We
already did most of the hard work with `allocuvm()` and `deallocuvm()` from the
post on process paging, so let's take a beat to thank past us for that.

Okay, so first we have to get the current process's size.
```c
int growproc(int n)
{
    struct proc *curproc = myproc();
    uint sz = curproc->sz;
    // ...
}
```

Depending on the size of `n`, we'll either grow the process or shrink it by `n`
bytes. Both `allocuvm()` and `deallocuvm()` can fail and return zero, so let's
add some checks for those and return -1 if they fail.
```c
int growproc(int n)
{
    // ...
    if (n > 0) {
        if ((sz = allocuvm(curproc->pgdir, sz, sz + n)) == 0) {
            return -1;
        }
    } else if (n < 0) {
        if ((sz = deallocuvm(curproc->pgdir, sz, sz + n)) == 0) {
            return -1;
        }
    }

    curproc->sz = sz;
    // ...
}
```

Finally, we need to tell the hardware that there's a new page directory in town
with a different size than the old one, so we'll use `switchuvm()` to update
the page directory and TSS stored by the hardware to reflect the changes. Then
we return 0 to indicate everything went okay.
```c
int growproc(int n)
{
    // ...
    switchuvm(curproc);
    return 0;
}
```

### procdump

This function is for debugging purposes: it'll print a complete listing of any
processes in the process table. Quick spoiler: the keyboard interrupt handler
function will set things up so that pressing `^P` runs this function. Go ahead,
load up xv6 and try it out!

We want to print out the state for each process, but the states in `enum
procstate` are just integers, which isn't very debug-friendly. So let's map them
all to strings first with a static array of strings.
```c
void procdump(void)
{
    static char *states[] = {
        [UNUSED]    "unused",
        [EMBRYO]    "embryo",
        [SLEEPING]  "sleep ",
        [RUNNABLE]  "runble",
        [RUNNING]   "run   ",
        [ZOMBIE]    "sombie",
    };
    // ...
}
```
This array notation might be a little unusual if you haven't seen it before: C
lets you initialize arrays by specifying the value of each entry. If you leave
any entries out, then they'll get initialized to zero. You can even write the
entries out of order by adding their index before them in square brackets. So
`{ [1] 5, [0] 2 }` is the same thing as `{2, 5}`. The `enum` turns the states
into integers, so they work as indices here.

Now we'll just iterate over the process table to get all the processes, skipping
over any `UNUSED` ones.
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state == UNUSED) {
            continue;
        }
        // ...
    }
}
```

Next we'll get the process's state (or just use `"???"` if something went wrong
and the state isn't recognized).
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        // ...
        char *state;
        if (p->state >= 0 && p->state < NELEM(states) && states[p->state]) {
            state = states[p->state];
        } else {
            state = "???";
        }
        // ...
    }
}
```

Then we can print out its PID, state, and name to the console.
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        // ...
        cprintf("%d %s %s", p->pid, state, p->name);
        // ...
    }
}
```

Finally, we'll see later on that the `sleep()` and `wakeup()` system calls
involve some lock trickery, so sleeping processes could be a common cause of
concurrency issues like deadlocks. So if a process is sleeping, we'll print out
its call stack using the `getcallerpcs()` function.
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        // ...
        if (p->state == SLEEPING) {
            uint pc[10];
            getcallerpcs((uint *) p->context->ebp + 2, pc);

            for (int i = 0; i < 10 && pc[i] != 0; i++) {
                cprintf(" %p", pc[i]);
            }
        }
        cprintf("\n");
    }
}
```

## Summary

Whew, we're making good progress. The most important part of this code was how
xv6 creates new processes and sets them up to start running: basically, it uses
some stack and function call trickery to make the scheduler start running a new
process with the code in `forkret()`, then `trapret()`, before switching context
into user mode.

We haven't talked about those two functions yet; we'll hold off on that until we
do traps and system calls. Next up is scheduling processes!

# Scheduling

We've done a lot of talking about context switching and scheduling, but we've
procrastinated looking at the code for those. It's time to fix that.

There are all kinds of advanced schedulers out there, but as we've said before,
the name of the game in xv6 is simplicity, so xv6 just uses a round-robin
scheduling algorithm in which in loops through the exisitng processes in order.
Each timer interrupt will force the current process to yield the processor and
perform a context switch back into the scheduler so it can run the next
available process.

## swtch.S

The `struct context` we talked about in the last post is gonna be key here, so
let's just look at its fields again:
```c
struct context {
    uint edi;
    uint esi;
    uint ebx;
    uint ebp;
    uint eip;
};
```

The context switch function is `swtch()`; it's gonna need to save and restore
processor registers, so that means it's gonna have to be written in assembly.
But let's pretend it's just a C function for a second and talk about what it's
going to do.

This function will save the contents of the registers on the stack as a `struct
context`, then save that location as the old context. Then it'll load a new
context, switch to the new stack, and restore the registers of the new context.
Its declaration would look like this in C:
```c
void swtch(struct context **old, struct context *new);
```
The first argument is a pointer *to a pointer* to a `struct context`. That
double indirection might be confusing, but there's a method to this madness: C
passes arguments by value, so if we used `struct context *old` and changed `old`
to point to the saved context, it would be lost as soon as we returned from this
function. So instead we have to use this kind of double pointer so we can set
`*old` to point to the saved context. This way `old` will be lost anyway, but
`*old` was changed and will persist beyond this function's return.

Note that, as we've said before, those arguments will be pushed on the stack
before `swtch()` is called. So at the beginning of `swtch()`, the stack pointer
`%esp` points to a return address; the argument `old` is one space (4 bytes)
above that in the stack, and `new` is one space higher than that.

Okay, let's check out the assembly code now. We're gonna start by saving those
arguments into registers. We can't just use any old registers here, or we might
overwrite some of the data we're trying to save. But in the last post, I said
x86 has a convention that the caller has to save the contents of the `%eax`,
`%ecx`, and `%edx` registers, so that means we're free to overwrite them all we
want since they've already been saved.
```asm
.globl swtch
swtch:
    movl    4(%esp), %eax
    movl    8(%esp), %edx
    # ...
```
We haven't seen this number-parenthesis notation in assembly yet, so in case
you're not familiar with x86 assembly, it's just a way to add a number to the
contents of a register, then treating it as a pointer and dereferencing it. So
`4(%esp)` in assembly is the same as `*(esp + 4)` in C. So at this point, `%eax`
holds the `struct context **old` pointer, and `%edx` holds the
`struct context *new` pointer.

Now it's time to save all the fields in a `struct context` on the stack. The
stack grows from high addresses to low ones, but C `structs` expect their fields
to be from low to high, so we'll save them in reverse order. Oh, and hang on --
remember what's at the bottom of the stack right now, after the arguments?
That's right, a return address. That's just a saved `%eip`, so that one's
already done for us! We just need to save the others.
```asm
swtch:
    # ...
    pushl   %ebp
    pushl   %ebx
    pushl   %esi
    pushl   %edi
    # ...
```

Next we have to save a pointer to this old `struct context` into `*old`. Well,
we pushed them on the stack in reverse order, right? So `%esp` already *is*
pointing to it, so that's our pointer; we'll just copy it into `*old` (remember
it's stored in `%eax`, and we dereference it in assembly with parentheses).
```asm
swtch:
    # ...
    movl    %esp, (%eax)
    # ...
```

Now it's time to switch stacks to the `new` context, which we saved in `%edx`.
That context must have been saved by a previous call to `swtch()`, so it also
happens to be a stack pointer as well.
```asm
swtch:
    # ...
    movl    %edx, %esp
    # ...
```

At this point, we're using the stack from `new`, which will already have its
saved context at the top. So we can load the new context by popping it off the
stack in reverse order into the corresponding registers. And again, just like
the `call` instruction had already saved `%eip` on the stack as the return
address, the `ret` (return) instruction will pop it off and restore it into
`%eip` for us.
```asm
swtch:
    # ...
    popl    %edi
    popl    %esi
    popl    %ebx
    popl    %ebp
    ret
```

And that's it! That's a context switch in xv6.

## proc.c

And now, finally, we can look at the scheduling code. Once the kernel is done
setting itself up, initializing all the devices and drivers, etc., the very last
function that `main()` calls is `scheduler()`. Interrupts were disabled in the
boot loader and haven't been enabled yet, so it's also the scheduler's job to
enable them for the first time in xv6.

`scheduler()` never returns; it's an infinite loop that just keeps searching
through the process table for a `RUNNABLE` process, then runs it. So from that
point on, with the exception of interrupts and system calls, the kernel will
only ever do one thing: schedule processes to run.

### scheduler

A CPU that's running the scheduler isn't running its own process. So we'll start
off by setting this CPU's process pointer to null. Note that `mycpu()` requires
interrupts to be disabled before it's called, but that's okay here because
interrupts were disabled in the boot loader and haven't been re-enabled before
the scheduler is called.
```c
void scheduler(void)
{
    struct cpu *c = mycpu();
    c->proc = 0;
    // ...
}
```

The order of the next few steps is tricky, and the authors of xv6 had to be
extremely careful to do them in the right order to avoid concurrency problems.
We need to (1) re-enable interrupts, (2) acquire the process table's lock, and
(3) create an infinite loop to iterate over the process table forever, scheduling
processes along the way. To see why this is nontrivial, let's check out some
different orders (with a `fake_scheduler()` function) and see what problems we
get.

ATTEMPT #1: interrupts -> lock -> loop. Let's try it out.
```c
void fake_scheduler1(void)
{
    // ...
    sti();                  // enable interrupts
    acquire(&ptable.lock);  // acquire lock
    for (;;) {              // infinite scheduling loop
        // ...
    }
}
```
Interrupts have been disabled since the boot loader used `cli`, so when we call
`sti()` here they'll be turned on for the first time in the kernel. At that
point we'll find out if there were any interrupts waiting to be acknowledged,
and possibly jump into some handler function to take care of it. Then when
that's done, we'll come back here and acquire the process table's lock. Acquiring
a lock disables interrupts, remember? So they're disabled again in the infinite
scheduling loop (but not forever; we'll release the lock before switching to a
user process). That sounds okay, right?

Not so fast! There's a hidden problem: suppose we had a situation in which none
of the current processes are `RUNNABLE` -- maybe they're all blocked (or
`SLEEPING`) waiting for I/O or something, which is not unlikely. In that case,
the scheduler would just keep idly looping through the process table until one
of them becomes `RUNNABLE` again. But if interrupts are always disabled in the
loop, then this processor will never find out about, e.g., a disk interrupt
saying it's done reading data which would allow a blocked process to become
`RUNNABLE`. That means the process will never find out the condition it's
waiting for has already happened, which means the scheduler will never find any
`RUNNABLE` processes. It'll just get stuck in an infinite loop, repeatedly and
desperately searching every entry of the process table. So basically, the
system would freeze while the CPU pointlessly spins at top speed.

Okay okay, so that doesn't work. We'll have to periodically re-enable interrupts
before disabling them again. So let's try moving the call to `sti()` inside the
infinite loop so interrupts get re-enabled every once in a while.

ATTEMPT #2: lock -> loop -> interrupts.
```c
void fake_scheduler2(void)
{
    // ...
    acquire(&ptable.lock);  // acquire lock
    for (;;) {              // infinite scheduling loop
        sti();              // temporarily enable interrupts
        // ...
    }
}
```
Problem solved, right? Actually... this one turns out to be just as bad. The
call to `acquire()` disables interrupts, only for `sti()` to enable them again.
There's a reason that locks disable interrupts, remember? If an interrupt occurs
that switches away from `scheduler()`, then it might call a handler function
that needs to access the process table lock, which is already held by
`scheduler()`, so that function would spin forever in a deadlock.

So now we arrive at the correct order: we'll call *both* `sti()` and `acquire()`
inside the loop, in that order. That means we'll also need a call to `release()`
at the end of the loop before we try to `acquire()` again in the next iteration.
We had already said we'd have to release the lock before running a process; now
we'll have to acquire it again before context-switching back into the loop.

ATTEMPT #3 (the right one): loop -> interrupts -> lock. This will give us a
chance to detect any outstanding interrupts in each iteration of the for loop,
but before we've acquired the lock again and thus, before doing so could cause a
deadlock.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        sti();
        acquire(&ptable.lock);
        // ... pick a process and run it ...
        release(&ptable.lock);
    }
}
```

Whew, okay. Basically, we've learned that concurrency bugs can be hard to predict
and can turn seemingly-fine code into impossible-to-diagnose system crashes or
freezes.

Okay, so now let's fill in the part of the loop where the scheduling algorithm
goes. We'll add an inner for loop to iterate over the process table entries
and stop when we find a `RUNNABLE` process.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            if (p->state != RUNNABLE) {
                continue;
            }
            // ... run that process ...
        }
        // ...
    }
}
```

Next, if we found a process, then we need to switch to that process's virtual
address space; that is, we need to start using its page directory.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            c->proc = p;
            switchuvm(p);
            // ...
        }
        // ...
    }
}
```
Now, if we just switched to an arbitrary page directory in the middle of running
other code, we might cause a bunch of problems: all the virtual addresses we're
currently using for variables, functions, instructions, etc. might suddenly
become invalid and point to random other places in memory. But this is where can
see some of the earlier design decisions in xv6 start to pay off: remember how
`setupkvm()` made sure every single process would have the exact same mappings
for the upper half of the address space, starting at `KERNBASE`? That means that
if we're running in kernel mode, we can arbitrarily switch to any process's page
directory and know that all of our mappings will be exactly the same. The user
mappings in the lower half might be different, but the kernel side will never
change. Nice!

Now we can run the process using `swtch()`.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            p->state = RUNNING;
            swtch(&(c->scheduler), p->context);
            // ...
        }
        // ...
    }
}
```
`swtch()` will *not* return here immediately; instead, it'll pick up execution
wherever the process last left off, which will be in kernel mode -- if it
stopped running before, it must have been due to a system call, interrupt, or
exception, which would have been handled in kernel mode before calling the
scheduler.

Note that this process will still be holding the process table lock when it
starts running again. For example, that's the main reason for the existence of
the `forkret()` function we mentioned before. This is another dangerous detail
we'll have to remember, so I'm just gonna go ahead and hope you remember THIS
BIG GIANT GLARING WARNING FLAG RIGHT HERE: if you do any xv6 kernel hacking, and
you want to add a new system call that will let go of the CPU, then your code
*must* release the process table lock at the point at which it starts executing
after switching to it from the scheduler.

This is pretty dangerous; if xv6 were a big project, it would be really easy to
forget that when adding more features later on. But in this case, there's no
easy way to get around it; for example, we can't just release the process table
lock before calling `swtch()` and reacquire it after. The problem becomes
apparent if you think of locks as protecting some invariant; that invariant
might be temporarily violated while you hold the lock, but it should be restored
before the lock is released.

The process table protects invariants related to the process's `p->state` and
`p->context` fields, e.g. that the CPU registers must hold the process's
register values, that a `RUNNABLE` process must be able to be run by any idle
CPU's scheduler, etc. These don't hold true while executing in `swtch()`, so we
need to hold the lock then; otherwise another CPU might decide to run the
process before `swtch()` is done executing.

Now, at some point that process will be done running and will give up the CPU
again. Before it switches back into the scheduler, it has to acquire the process
table lock again. So here's ONE MORE GIANT WARNING for good measure: you should
make sure to do that too if you add your own scheduling-related system call.

Eventually, it'll switch back here with a call with the arguments in reverse,
like `swtch(&(p->context), c->scheduler)`. At the point, execution of the
scheduler will resume right here, so we need to switch back to using the kernel
page directory `kpgdir`.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            switchkvm();
            c->proc = 0;
        }
        // ...
    }
}
```

After that, the outer for loop just releases the lock before looping to the top
again to temporarily re-enable interrupts, then acquire the lock again and check
for another process to run.

### forkret

Let's take a quick look at one example of where a process might start to execute
after being scheduled. All processes (whether the very first process, or any
others created later through calls to `fork()`) will start running code in
`forkret()`, then return from here into `trapret()`.

Most of the time, this function does just one thing: it releases the process
table lock. However, there are two kernel initialization functions that have to
be run from user mode, so we can't just call them from `main()` and be done with
it. We need a place for a process to call them, and `forkret()` is as good a
place as any. So the very first call to `forkret()` will run these two start-up
functions, and the rest will ignore them.

The two functions are `iinit()` and `initlog()`, which are part of xv6's file
system code; we'll get to them later on. For now, we'll just use a `static int`
as a boolean and set it to false after we've run those functions once on our
first pass through `forkret()`.
```c
void forkret(void)
{
    static int first = 1;

    release(&ptable.lock);

    // Only gets run once, on the first call
    if (first) {
        first = 0;
        iinit(ROOTDEV);
        initlog(ROOTDEV);
    }
    // Returns into `trapret()`
}
```
Any other kernel code that switches into the scheduler (e.g., `sleep()` and
`yield()`) will have a similar lock release right after returning from
the scheduler.

### sched

We saw one example of code that runs after switching *away* from the scheduler,
but what about code that runs before switching *to* the scheduler? Any functions
that need to call into the scheduler can't just call `scheduler()`, since the
scheduler probably left off last time halfway through the loop and should resume
in the same place. So `sched()` handles the task of picking up the scheduler
wherever it last left off.

`sched()` should be called *after* acquiring the process table lock and without
holding any other locks (lest we cause a deadlock somewhere). Also, the process
should not be in the `RUNNING` state anymore since we're about to stop running
it. So we'll start off by checking that those are all true and that interrupts
are disabled.
```c
void sched(void)
{
    struct proc *p = myproc();

    if (!holding(&ptable.lock)) {
        panic("sched ptable.lock");
    }
    if (mycpu()->ncli != 1) {
        panic("sched locks");
    }
    if (p->state == RUNNING) {
        panic("sched running");
    }
    if (readeflags() & FL_IF) {
        panic("sched interruptible");
    }
    // ...
}
```

Next, remember when the `pushcli()` and `popcli()` functions checked whether
interrupts were enabled before turning them off while holding a lock? That's
really a property of this kernel thread, not of this CPU, so we need to save
that now. Then we can call `swtch()` to pick up where the scheduler left off
(the line right after its own call to `swtch()`). This process will resume
executing after that line eventually, at which point we'll restore the data
about whether interrupts were enabled and let it run again.
```c
void sched(void)
{
    // ...

    // Save whether interrupts were enabled before acquiring the lock
    int intena = mycpu()->intena;

    // Perform context switch into the scheduler
    swtch(&p->context, mycpu()->scheduler);
    // Execution will eventually resume here

    // Restore whether interrupts were enabled before
    mycpu()->intena = intena;
}
```

### yield

Okay, let's see an example of how this all comes together now! The `yield()`
function forces a process to give up the CPU for one scheduling round. For
example, this will be used to handle timer interrupts later on. Now that we know
how scheduling works in xv6, `yield()` is easy. We just acquire the process
table lock, set the current process's state to `RUNNABLE` so it can get picked
up again in the next scheduling round, and call `sched()` to switch into the
scheduler. When we eventually return here, we'll just release the lock again.
```c
void yield(void)
{
    acquire(&ptable.lock);

    myproc()->state = RUNNABLE;
    sched();

    release(&ptable.lock);
}
```

## Summary

We've now seen how xv6 handles process scheduling with a super-simple round-
robin algorithm. The `scheduler()` function had plenty of concurrency pitfalls,
but luckily the xv6 authors took care of all the careful coding for us, so we
just get to sit back and admire their work.

We also saw how context switches occur in xv6, so now we can understand how, in
the previous post, `allocproc()` set up a new process with a context that would
result in it starting execution in `forkret()`.

Next up, we'll look at the way xv6 handles interrupts, system calls, and software
exceptions.

# It's a Trap!

The last post introduced the mechanisms that xv6 uses for scheduling and context
switches. User processes can transfer control to kernel code with system calls,
potentially switching into the scheduler with `sleep()` or `exit()` to find
another process to run. But there are many other system calls besides those two
Kernel code can also be invoked during hardware interrupts or software
exceptions; these three together are collectively referred to as traps.

We'll go over traps now to understand them more generally. First, about the
terminology: depending on the source, interrupts might mean hardware interrupts
specifically or any trap generally; similarly, exceptions might mean errors
arising from the code, or traps in general. It's super frustrating because it
makes it really hard to know what's meant by a word like "interrupt" or
"exception" in whatever specification or source you happen to be reading. So I'm
gonna try my best to save you that kind of pain in this post by sticking to
"interrupt" for the hardware interrupts only, "exception" for software errors,
and "trap" for those two combined with system calls.

## Interrupt Descriptor Table

Imagine if, after every single time some user code carried out a division, the
processor stopped, context switched into the kernel, and asked the kernel to
check if there was a division by zero and handle it if necessary. Or every time
a hardware interrupt happened, the kernel had to start polling all the devices
to figure out which one just yelled. No. Just no. Running kernel code for all
this would be way too slow.

So it's the processor that will have to detect traps and decide how to handle
them. But what exactly it should do for a specific trap depends on all kinds of
of particulars about that OS, e.g. a disk saying it's done reading from a file
might require updating some file system data or storing the disk data in a
specific buffer or something. That's too much responsibility for the processor.

Okay, so the kernel will set up a bunch of handler functions for every possible
type of trap. Then it tells the hardware, "Okay, so if you get a disk interrupt,
here are my instructions to handle that. For timer interrupts, use these
instructions. If a process tries to access an invalid page, do this..."
From then on, the processor can handle the traps without further input from the
kernel by looking up the interrupt number in a big table to get the trap handler
function that the kernel set up, then just running it.

In the x86 architecture, that table is called the *interrupt descriptor table*
or IDT. I know, I'm sorry, I promised I'd say "trap" for the general case, but
the x86 specs give it the official name of IDT even though it handles all the
traps. Sigh. It has 256 entries (so that's the maximum number of distinct traps
we can define); each one specifies a segment descriptor (ugh segmentation again,
you know what that means: opaque code) and an instruction pointer (`%eip`) that
tell the processor where it can find the corresponding trap handler
function.

xv6 won't use all 256 entries; it'll mostly use trap numbers 0-31 (software
exceptions), 32-63 (hardware interrupts), and 64 (system calls), all defined in
[traps.h](https://github.com/mit-pdos/xv6-public/blob/master/traps.h).
But we do have to stick all 256 in the IDT anway, so we're the unlucky fools who
get to write 256 functions' worth of assembly code by hand. Nah, just kidding:
xv6 uses a script in a high-level language to do that for us and spit out the
entries into an assembly file.

Unfortunately for us, that high-level language is Perl. Sigh. Perl is infamous
as a "write-only" language, so I guess instead we're just the unlucky fools who
get to try reading Perl.

## vectors.pl

Okay, I'm not gonna assume you know Perl, and either way I really don't wanna go
over every single line of this file. The syntax is similar enough to C's (except
that somehow they managed to make it even *worse* than C), so you can read it on
your own if you want.

Now, no script will be able to generate 256 completely unique assembly functions
with enough detail to handle each trap correctly, so each function in the script
has to be pretty generic. They're all gonna call the same assembly helper
function, which will call a C function where we can more comfortably code up
how to handle each interrupt.

The gist of this Perl script is that it prints a bunch of stuff using a for loop
with 256 iterations. The xv6
[Makefile](https://github.com/mit-pdos/xv6-public/blob/master/Makefile)
will run it from the command line with `./vectors.pl > vectors.S` so that the
output gets saved in an assembly file, which will then get assembled together
with all the other kernel code in `OBJS`.

The resulting assembly file will look like this:
```asm
.globl alltraps

.globl vector0
vector0:
    pushl   $0
    pushl   $0
    jmp     alltraps

.globl vector1
vector1:
    pushl   $0
    pushl   $1
    jmp     alltraps

.globl vector2
vector2:
    pushl   $0
    pushl   $2
    jmp     alltraps

# ...
```

Except that a handful of entries (8, 10 through 14, and 17) will skip one line
(I'll explain why below):
```asm
# ...

.globl vector8
vector8:
    pushl   $8
    jmp     alltraps

# ...
```

Then at the end, it defines an array `vectors` with each of those entries above:
```asm
# ...

.data
.globl vectors

vectors:
    .long vector0
    .long vector1
    .long vector2
    # ...
```

Okay, so those are all the handler functions; the `vectors` array holds a
pointer to each one. They're all more or less the same: most of them push zero
onto the stack, then all they push a *trap number* to indicate which trap
just happened, and then they jump to a point in the code called `alltraps`;
that's the assembly helper function I mentioned earlier.

A handful of the entries don't push zero on the stack: these are trap numbers
8 (a double fault, which happens when the processor encounters an error while
handling another trap), 10 (an invalid task state segment), 11 (segment
not present), 12 (a stack exception), 13 (a general protection fault), 14 (a
page fault), and 17 (an alignment check). These are special because the
processor will actually push an error code on the stack before calling into the
corresponding handler function in `vectors`. It doesn't push any error codes on
the stack for the others, so we just push 0 ourselves to make them all match up.

## trapasm.S

### alltraps

The processor needs to run the trap handler in kernel mode, which means we have
to save some state for the process that's currently running so we can return to
it later (similar to the `struct context` we saw before), then set things up to
run in kernel mode. The `alltraps` routine does just that.

Remember how we said the IDT holds segment selectors for `%cs` and `%ss`, plus
and instruction pointer `%eip`? (I know we haven't seen the code to create the
IDT and store the entries of `vectors` in it yet; we'll get to that below.) The
processor will start using those segments (and save the old ones) before running
the trap handler function. Each trap handler function in `vectors` above pushed
an error code (or 0) followed by a trap number. Now we have to push all the
other segment selectors on the stack one at a time, then push all the general-
purpose registers at once with the x86 instruction `pushal`.
```asm
.globl alltraps
alltraps:
    pushl   %ds
    pushl   %es
    pushl   %fs
    pushl   %gs
    pushal

    # ...
```

Cool, all the registers are saved now. So now we'll set up the `%ds` and `%es`
registers for kernel mode (`%cs` and `%ss` were already done by the processor).
```asm
    # ...
    movw    $(SEG_KDATA<<3), %ax
    movw    %ax, %ds
    movw    %ax, %es
    # ...
```

Now we're ready to call the C function `trap()` that's gonna do most of the
work. That function expects a single argument: a pointer to the process's saved
register contents. Well, we just pushed them all on the stack, so we can just
use `%esp` as that pointer.
```asm
    # ...
    pushl   %esp
    call    trap
    # ...
```

That function will return back here when it's done, so let's ignore the return
value by moving the stack pointer just above it (essentially popping it off the
stack).
```asm
    # ...
    addl    $4, %esp
    # ...
```

### trapret

We've talked about this function before; when we create a new process, it starts
executing in `forkret()`, which then returns into `trapret()`. More generally,
any call to `trap()` will return here as well.

This function just restores everything back to where it was before, popping
stored registers off the stack in reverse order. We can skip the trap number and
error code; we won't need them anymore. Then we use the `iret` or "interrupt
return" (though you should read that as "trap return") instruction to close out,
return to user mode, and start executing the process's instructions again.
```asm
.globl trapret
trapret:
    popal
    popl    %gs
    popl    %fs
    popl    %es
    popl    %ds
    addl    $0x8, %esp  # skip the trap number and error code
    iret
```

## trap.c

Okay, on to the main part of the code! We have to do two things here: stick the
trap handler functions in `vectors` into an IDT, and figure out what to do with
each interrupt type.

At the top, we've got four global variables. The IDT is represented as an array
of `struct gatedesc`s, defined in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h). It's worth
taking a look at because it uses an obscure C feature (bit fields); we'll do that
in the next section.

Then we declare the `vectors` array of trap handler (with an `extern` keyword,
since it's defined in an assembly file), a global counter `ticks` that tracks
the number of timer interrupts so far (basically a rough timer), and a lock to
use with `ticks`.
```c
struct gatedesc idt[256];
extern uint vectors[];
struct spinlock tickslock;
uint ticks;
// ...
```

### Bit Fields

This section will get *deep* into the weeds, so feel free to skip it if you're
having a nice day and don't want to spoil it by reading about a bunch of C
standards.

So far, we've used bit flags with regular integers by manually doing some bit
arithmetic to set one bit at a time. For example, the flags for page table and
page directory entries are defined as powers of 2 (e.g., `PTE_P` is 0x1, `PTE_W`
is 0x2, `PTE_U` is 0x4, etc.) so that we can set a specific bit using a bitwise-
OR like `pte |= PTE_U` or test whether it's set with a bitwise-AND like
`pte & PTE_P`.

But sometimes that can get annoying and hard to keep track of; wouldn't it be
nice if we could just have variables that represent a single bit? Or two bits,
or any number of bits we want?

The trouble is that most computer architectures don't work with a single bit at
a time; they operate on bytes, words (2 bytes), long/double words (4 bytes), or
quad words (8 bytes), so it would be nontrivial to compile a line of C like
`a = 1` if `a` is a nonstandard size.

In fact, accessing variables that aren't aligned to a standard size (4 bytes on
x86 or 8 bytes on x86_64) is much slower than when they are aligned. Compilers
often optimize code to correct for this by padding `struct`s so that they'll
line up along those standard sizes. For example, one like
```c
struct nopadding {
    int n;
};
```
is probably left the same on x86, but one like this:
```c
struct padding {
    char a;
    int n;
    char b;
};
```
is probably converted by the compiler into this:
```c
struct padding {
    char a;
    char pad0[3];
    int n;
    char b;
    char pad1[3];
};
```

WARNING: We're entering the dark arts of C's unspecified and implementation-
defined behavior here. Note that these are different from *undefined* behavior:
undefined behavior means you did something BAD BAD BAD like dereferencing a null
pointer, freeing a memory region twice, using a variable after freeing it,
accessing an out-of-bounds index in a buffer, or overflowing a signed data type.
Implementation-defined and unspecified behavior aren't as dangerous as undefined
behavior is, but they can cause portability issues.

The C standard is a huge document with a bunch of legalese rules about what
makes C, well, C. People who write C compilers need to know exactly how C code
should behave under all kinds of different circumstances, so the C standard
spells most of it out. But there are some parts it intentionally leaves out.

*Implementation-defined* behavior means the C standard doesn't set any fixed
requirements about how a compiler should handle some behavior or feature; the
developers of a C compiler get to decide how to write that part of the code with
total freedom. One example is the number of bits in a byte; we've been assuming
it's 8, but there are some (dumb) architectures where it's different.

*Unspecified behavior*, on the other hand, means that the C Standard provides
some specific options, and compiler developers have to choose from those options
for *each instance* of the behavior in the code they're compiling (that means,
don't assume it's always gonna be the same, even with the same compiler).

Structure padding is implementation-defined, and there are often implementation-
defined ways to modify it or disable it altogether (i.e., to *pack* the `struct`
instead of *padding* it), usually with stuff like `__attribute__`s or `#pragma`
directives for the preprocessor.

Wait weren't we gonna talk about bit manipulation? Why are we talking about
`struct`s? Well, C does have a workaround to make bit manipulation a little
easier by avoiding that slightly-annoying bit arithmetic you have to do to set
or clear flags in an `int` or `unsigned int`: it's called a *bit field*, and it
takes advantage of `struct` padding.

You can specify the number of bits that a field of a `struct` should occupy by
adding a colon and a size after the field name:
```c
struct bitfield_example {
    unsigned char a : 1;
    unsigned char b : 7;
};
```
This way, you can set the single-bit flag `a` with simple variable assignments
like `var.a = 1`, and the compiler will figure out any necessary magic similar
to structure padding to make that happen. Awesome, right? So why haven't we been
using it all the time instead of all that opaque bit arithmetic with arcane
operators like `<<`, `>>`, `|`, and `&`?

Well, there are some big downsides to bit fields. First, the C standard sets
some strict rules on their use to make sure that compilers can figure out how to
handle them. Bit fields are only allowed inside of structures. You're not
allowed to create arrays of bit fields or pointers to bit fields. Functions
aren't allowed to return a bit field. You're not allowed to get the address of a
bit field with the `&` operator. You can only operate on a single bit field at a
time in any statement; that means you can't set one bit field to equal another,
and you can't compare the values of two bit fields.

Second, they're *extremely* implementation-defined. Each implementation (read:
compiler + architecture combo) determines what data types and sizes are allowed
to be used in bit fields. The data types you *can* use might have different
signedness rules from the usual ones for signed and unsigned types. How they're
laid out, ordered, and padded in memory can differ. In short: the low-level
details are a total black box that you can probably only figure out by reading
*deep* into the compiler's specifications.

Now imagine trying to do something that requires specific protocols like sending
data over a network, and you come across a bit field. Lolwut. Who knows what
you'd have to do. Bit fields make it impossible to port your code.

BUT! Bit arithmetic is annoying, so let's use bit fields anyway!

Okay, so back to `struct gatedesc`. IDT entries have to contain a 16-bit code
segment selector (`%cs`), 16 low bits and 16 high bits for an offset in that
segment, the number of arguments for the handler function, a type, a system/
application flag, a descriptor privilege level (0 for kernel, 3 for user), and a
"present" flag. And x86 is very particular about how it's all laid out, so we
have to set up `struct gatedesc` in the exact right order.
```c
struct gatedesc {
    uint off_15_0 : 16;
    uint cs : 16;
    uint args : 5;
    uint rsv1 : 3;
    uint type : 4;
    uint s : 1;
    uint dpl : 2;
    uint p : 1;
    uint off_31_16 : 16;
};
```

Well, okay, that's it for now.

### tvinit

This function loads all the assembly trap handler functions in `vectors` into
the IDT. The `SETGATE()` macro in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) will organize
each entry correctly. We said before that the IDT needs a code segment selector,
an instruction pointer (from `vectors`), and a privilege level (0 for kernel
mode), so we'll stick those in.
```c
void tvinit(void)
{
    for (int i = 0; i < 256; i++) {
        SETGATE(idt[i], 0, SEG_KCODE << 3, vectors[i], 0);
    }
    // ...
}
```

We're basically done now, but there's one last hiccup: user code needs to be
able to generate system calls, but we just set all the privilege levels so only
the kernel and processor can generate traps. So we'll fix the entry for system
calls as a special case.
```c
void tvinit(void)
{
    // ...
    SETGATE(idt[T_SYSCALL], 1, SEG_KCODE << 3, vectors[T_SYSCALL], DPL_USER);
    // ...
}
```

Oh and while we're at it, let's just go ahead and initialize the lock for the
tick counter.
```c
void tvinit(void)
{
    // ...
    initlock(&tickslock, "time");
}
```

### idtinit

The last function stored all the trap vectors in the IDT, so now we need to tell
the processor where to find the IDT. There's a special assembly instruction for
that in x86 called `lidt`.
```c
void idtinit(void)
{
    lidt(idt, sizeof(idt));
}
```

### trap

This last function is the one that gets called by the assembly code in `alltraps`;
it's responsible for figuring out what to do based on the trap number we pushed
on the stack before. Heads up: it's gonna do that by calling a bunch of other
functions, many of which we haven't seen yet. I'll just give a quick summary
when we come across them, and we'll get to them later on.

The only argument is a pointer to a `struct trapframe`. Wait, hang on. Up above
in the assembly code, the argument we pushed on the stack was `%esp`, the stack
pointer, not a pointer to any `struct trapframe`. What's up with that? Did we
pass the wrong kind of argument in?

Let's check out the definition for `struct trapframe`, found in
[x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h). It's got a
bunch of fields, starting off with the general purpose registers (those are the
fields from `%edi` to `%eax`). Then it has four segment registers (fields `%gs`
through `%ds`), plus some unused padding bits in between them to round the 16-
bit segment registers up to 32 bits. The next two fields are a trap number and
an error code.

All that should sound familiar. Take another look at
[trapasm.S](https://github.com/mit-pdos/xv6-public/blob/master/trapasm.S): so
far, those are the exact same things we pushed on the stack! The other fields
are what the processor pushed on the stack before calling the handler function
in the IDT. So basically, we're never gonna construct a `struct trapframe` in C
code; we already constructed it manually in assembly. It just describes
everything that's already on the stack by the time this `trap()` function gets
called. In that sense, the `%esp` we pushed as an argument really *is* a pointer
to a `struct trapframe`. It's a clever way to read values off the stack.

So we said we're gonna check the trap number and decide which kernel function to
call based on that, right? Let's start by checking if the trap number indicates
this is a system call (trap number 64, or `T_SYSCALL`).
```c
void trap(struct trapframe *tf)
{
    if (tf->trapno == T_SYSCALL) {
        // ...
    }
    // ...
}
```
Well how should we handle system calls? xv6 will have several, and we don't even
know what they all are yet. So let's procrastinate again and just call some
other function `syscall()` to handle the work of figuring out which system call
to execute. Now we'll store the pointer to the `struct trapframe` in that
process's `struct proc`, obtained with a call to `myproc()`. Also, processes
need to be killed once they're done, or if they cause an exception; that happens
by setting a `killed` flag in the `struct proc`. So we'll check for that before
and after carrying out the system call and close the process out with `exit()`
if it's due to be killed.
```c
void trap(struct trapframe *tf)
{
    if (tf->trapno == T_SYSCALL) {
        if (myproc()->killed) {
            exit();
        }
        myproc()->tf = tf;
        syscall();
        if (myproc()->killed) {
            exit();
        }
        return;
    }
    // ...
}
```

Okay, now we have all the other trap numbers to think about. We could do them
with a ton of `if` statements, but that would be a pain; we'll use a `switch`
statement instead. If you haven't seen `switch` statements, they replace big
`if-else` blocks with cases instead. The cases can only be indexed by integers,
and you have to stick a `break` statement at the end or else you'll fall through
to the next case and execute the code found there as well. (To be honest, I
don't see a reason why the system call case wasn't just included in this same
switch statement; if you see a reason for that, let me know.)
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        // cases go here
    }
    // ...
}
```

First up is the trap number for timer interrupts; the main function of timer
interrupts is to schedule a new process, but that will come further down in this
function. For now, we'll just increment the `ticks` counter then call `wakeup()`,
which checks if any processes went to sleep until the next tick; it'll switch to
running any process it finds. There's one detail to deal with here: the system
may have multiple processors, each with their own timer and interrupts. We want
to use the `ticks` counter as a rough timer, but we don't know whether all the
CPU timers will be synchronized, so we'll only update `ticks` using the first
CPU to avoid those issues.

If you read the post on interrupt controllers then you'll be familiar with
`lapiceoi()`; if you didn't (or you forgot), it just tells the local interrupt
controller that we've read and acknowledged the current interrupt so it can
clear it and get ready for more interrupts.
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        case T_IRQ0 + IRQ_TIMER:
            if (cpuid() == 0) {
                acquire(&tickslock);
                ticks++;
                wakeup(&ticks);
                release(&tickslock);
            }
            lapiceoi();
            break;
        // ...
    }
    // ...
}
```

Later on, we'll see some interrupt handler functions for various devices:
`ideintr()` handles disk interrupts, `kbdintr()` for key presses and releases,
and `uartintr()` for serial port data. We'll direct the corresponding interrupts
to those functions, then acknowledge and clear them with `lapiceoi()`. Also,
devices occasionally generate spurious interrupts due to hardware malfunctions;
we'll either ignore them (if they're coming from the Bochs emulator) or print a
message about it to the console.
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        // ...
        case T_IRQ0 + IRQ_IDE:      // disk interrupt
            ideintr();
            lapiceoi();
            break;
        case T_IRQ0 + IRQ_IDE + 1:  // spurious Bochs disk interrupt
            break;
        case T_IRQ0 + IRQ_KBD:      // keyboard interrupt
            kbdintr();
            lapiceoi();
            break;
        case T_IRQ + 7:             // spurious interrupt-no break, FALL THROUGH
        case T_IRQ + IRQ_SPURIOUS:  // spurious interrupt
            cprintf("cpu%d: spurious interrupt at %x:%x\n",
                    cpuid(), tf->cs, tf->eip);
            lapiceoi();
            break;
        // ...
    }
    // ...
}
```

Okay, so now we've dealt with system calls and hardware interrupts, so any other
trap must be a software exception. `switch` statements allow a catch-all case
with `default`, so we'll use that to catch the rest of the trap numbers. Now,
this may have come from a kernel error or a misbehaving user process. We can
check with `myproc()`, which returns a null pointer if we were running kernel
code or a pointer to a `struct proc` if we were in user space, or by checking
the current privilege level in the code segment selector. Depending on the
source, we'll print out an appropriate error message and either panic (if in the
kernel) or mark the process so it gets killed soon.
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        // ...
        default:
            if (myproc() == 0 || (tf->cs & 3) == 0) {
                // Kernel code exception
                cprintf("unexpected trap %d from cpu %d eip %x (cr2=0x%x)\n",
                        tf->trapno, cpuid(), tf->eip, rcr2());
                panic("trap");
            }
            // User process exception
            cprintf("pid %d %s: trap %d err %d on cpu %d "
                    "eip 0x%x addr 0x%x--kill proc\n",
                    myproc()->pid, myproc()->name, tf->trapno,
                    tf->err, cpuid(), tf->eip, rcr2());
            myproc()->killed = 1;
    }
    // ...
}
```
The reason we don't kill it immediately is because the process might be executing
some kernel code right now; for example, system calls allow other interrupts and
exceptions to occur while they're being handled. Killing it now might corrupt
whatever it's doing. So instead we just give it the kiss of death for now and
come back to finish the job later.

So next up, we'll check if this trap was generated by a user process that's due
to be killed, and that process is running in ring 3. If so, we finally do
the deed with `exit()`; otherwise if it's running in ring 0, it'll live for now
and get killed the next time it generates a trap instead.
```c
void trap(struct trapframe *tf)
{
    // ...
    if (myproc() && myproc()->killed && (tf->cs & 3) == DPL_USER) {
        exit();
    }
    // ...
}
```

Up above, the only thing a timer interrupt did was increment `ticks`. But we
know a really important function of timer interrupts is to force a process to
let go of the CPU and let someone else run. It's time to do that. We'll check if
the process's state is `RUNNING` and the trap was a timer interrupt; if so, we
call `yield()` to let another process get scheduled on this CPU.
```c
void trap(struct trapframe *tf)
{
    // ...
    if (myproc() && myproc()->state == RUNNING &&
            tf->trapno == T_IRQ0 + IRQ_TIMER) {
        yield();
    }
    // ...
}
```

Now we have one last check: a process that yielded, then got picked up again
later might have been marked as killed in the meantime, so if it was, we need to
finish it off now. So we do the exact same check as above again, and then we're
done.
```c
void trap(struct trapframe *tf)
{
    // ...
    if (myproc() && myproc()->killed && (tf->cs & 3) == DPL_USER) {
        exit();
    }
}
```
Note that this function will return into `trapret` in the assembly code, which
will then send it back to user mode.

## Summary

Let's take a moment to assess how much of xv6 we've already covered. Remember,
the xv6 kernel has four main functions: (1) finishing the boot process that the
boot loader started, (2) virtualizing resources in order to isolate processes
from each other, (3) scheduling processes to run, and (4) interfacing
between user processes and hardware devices. Let's take that as a checklist and
go through those items now.

We've already seen some of the initialization routines that get run on boot in
`main()`; most of the code there sets up virtual memory and all the hardware
devices. We still have a few more devices to talk about: the keyboard, serial
port, console, and disk; each of those has its own boot function that we'll need
to go over in order to wrap up point (1).

On the other hand, we're already done with (2) and (3): we spent a lot of time
going over virtual memory and paging, and the last post on scheduling showed us
how xv6 virtualizes the CPU as well as it runs processes.

The code we saw in this post was our introduction to point (4). Traps are the
primary mechanism for user processes to communicate with the hardware; the
kernel coordinates that communication by setting up trap handler functions. The
code we've seen here basically acts like an usher, directing traps to the
right trap handler function depending on its type.

When a trap occurs (x86 instruction `int`), the processor will stop executing
code, find the IDT, and looks up the entry for that trap number. The script that
xv6 uses to generate the IDT entries just makes them all point to the same
function `alltraps()`, which saves all the process's registers, switches into
kernel mode, and calls `trap()`. Then that function uses the trap number to
figure out how the kernel wants it to respond to this particular trap. So any
hardware interrupt, software exception, or user system call will get funneled
into the functions here before getting dispatched to some other appropriate
kernel code that will know what to do with it.

We haven't finished point (4) yet, though: we have to actually see what each of
those trap handler functions does. But we did see some of them: for example, we
saw that a software exception either kills the process that caused it or panics
if it occurred in kernel code. That already takes care of one of the three types
of traps, so we're left with hardware interrupts and system calls. All the
system calls got redirected to a `syscall()` function which we haven't seen yet.

We have seen how some of the hardware interrupts are dealt with: a timer
interrupt increments a `ticks` counter (if it's on CPU 0), then calls `yield()`
to force a process to give up the CPU until the next scheduling round. Spurious
interrupts either get ignored or print a message to the console. But we've
procrastinated some of the others: disk interrupts call an `ideintr()` function
to handle them, keyboard interrupts call `kdbintr()`, and serial port interrupts
call `uartintr()`, none of which we've gone over.

So in order to wrap up the xv6 kernel, we still have to understand how system
calls are routed in general, as well as how devices are initialized at boot and
how the kernel responds to specific system calls that require use of those
devices. The general system call routing mechanism is up next.

# System Calls: Routing

We said in the last post that system calls are the primary means for user
processes to request some action by the kernel; system calls mediate processes'
access to hardware resources.

If a user process wants to generate a system call, it starts a trap with the
trap number for system calls. Then it identifies which of the various xv6 system
calls it wants to do and passes any required arguments. The processor will then
handle the trap instruction using the code we saw in the last post. Eventually,
it'll get to the `trap()` function, which will recognize the trap number as a
system call and pass it on to the `syscall()` function.

`syscall()` is itself a routing function like `trap()`; it'll figure out which
system call the process created and redirect it again to the appropriate kernel
code.

## syscall.c

All system calls use the same trap number: 64, or `T_SYSCALL`, but xv6 has
multiple system calls, so we need another number for a process to identify which
system call it wants to run. The convention on x86 is to use a system call
number which the calling process should put in the `%eax` register, which
usually holds return values. Then the kernel's handler function (here,
`syscall()`) can just check `%eax` to figure out which system call to run. The
system call numbers are defined in
[syscall.h](https://github.com/mit-pdos/xv6-public/blob/master/syscall.h). There
you can see that, e.g. `SYS_fork` is defined as 1, `SYS_exit` is 2, and so on.

All the system call functions are defined in other files, so we'll have to
import their declarations with the `extern` keyword:
```c
// ...
extern int sys_chdir(void);
extern int sys_close(void);
extern int sys_dup(void);
extern int sys_exec(void);
extern int sys_exit(void);
extern int sys_fork(void);
extern int sys_fstat(void);
extern int sys_getpid(void);
extern int sys_kill(void);
extern int sys_link(void);
extern int sys_mkdir(void);
extern int sys_mknod(void);
extern int sys_open(void);
extern int sys_pipe(void);
extern int sys_read(void);
extern int sys_sbrk(void);
extern int sys_sleep(void);
extern int sys_unlink(void);
extern int sys_wait(void);
extern int sys_write(void);
extern int sys_uptime(void);
// ...
```

Now we've got the numbers and the functions. Note that the numbers start with
uppercase `SYS_` and the functions start with lowercase `sys_`, so make sure
your kernel hacking adventures don't do anything like `SYS_fork()`; use
`sys_fork()` instead.

We'll also need a way to map the numbers to those system call functions so that
`syscall()` can call the right one depending on the number. We could use another
`switch` statement like we did in `trap()`, but there are 21 system calls here,
so that would get pretty long; also, each number will just call the specific
function, unlike the different trap numbers which required different responses
(e.g., the timer interrupt trap number didn't call any function at all). xv6
does something else this time that's much simpler and more elegant, but it uses
some slightly-obscure C features, so we'll go over it carefully.

Remember function pointers from way back in the boot loader? Functions are just
a set of instructions in order, loaded somewhere in the kernel's code segment,
so C lets us use the function's name as a pointer to the beginning of its code
in memory. So if we have a C function like `int func(char c)`, then `func` is
its function pointer. We could even assign it to a variable; that variable's
type would be a pointer to a function of argument type `char` and return type
`int`; then we could call the function using the new pointer too. Here's an
example that would print "Match!" to the screen:
```c
int m = func('a');

int (*func_ptr)(char) = &func;
int n = (*func_ptr)('a');

if (m == n) {
    printf("Match!\n");
}
```

So instead of a big old `switch` statement, the `syscall()` function will use a
static, global array of pointers to all the system call functions we just
imported above. (Remember that the `static` keyword in front of a variable means
it always occupies the same fixed place in memory.) It'll work because all the
functions have the same argument type (`void`) and return type (`int`), so their
pointers all have the same type and can fit inside a single array. Then we can
get the right function by just using the system call number to index into the
array of function pointers.

Now, we'd have to be super careful to add the function pointers into the array
in the right order so that the indices match up. Even worse, there is no system
call with number zero, so we'd have to skip that entry of the array. This could
get complicated. Luckily, even though humans are bad at this kind of thing,
computers are *really* good at it. So instead of trying to line them up by hand,
we can use the array notation from `procdump()` in the post on processes where
we specified the value of each entry of an array like this with the index in
square brackets, like this:
```c
int arr[] = { [2] 5, [0] 1, [4] -2 };
```
The C compiler will use the indices we wrote there to figure out that the array
needs 5 entries (indices 0 to 4), and entry 0 is 1, entry 2 is 5, and entry 4 is
-2. Entries 1 and 3 will just be initialized to zero.

So at the end of the day, our array of pointers to system call functions looks
like this:
```c
// ...
static int (*syscalls[])(void) = {
    [SYS_fork]      sys_fork,
    [SYS_exit]      sys_exit,
    [SYS_wait]      sys_wait,
    [SYS_pipe]      sys_pipe,
    [SYS_read]      sys_read,
    [SYS_kill]      sys_kill,
    [SYS_exec]      sys_exec,
    [SYS_fstat]     sys_fstat,
    [SYS_chdir]     sys_chdir,
    [SYS_dup]       sys_dup,
    [SYS_getpid]    sys_getpid,
    [SYS_sbrk]      sys_sbrk,
    [SYS_sleep]     sys_sleep,
    [SYS_uptime]    sys_uptime,
    [SYS_open]      sys_open,
    [SYS_write]     sys_write,
    [SYS_mknod]     sys_mknod,
    [SYS_unlink]    sys_unlink,
    [SYS_link]      sys_link,
    [SYS_mkdir]     sys_mkdir,
    [SYS_close]     sys_close,
};
// ...
```

Okay great, now we're ready to route system calls to the right function.

### syscall

The first thing we need to do is get the system call number so we can figure out
which function to call. We said above that the x86 convention is to store it in
the `%eax` register, but we might have a problem: by the time we get to
`syscall()`, the processor has already executed the code in the trap handler
function for trap number `T_SYSCALL`, which sent it to `alltraps()`, which
replaced all the register contents with those of `trap()`, so the system call
number is probably long gone from `%eax`.

But wait, all is not lost! `alltraps()` saved all the registers in a
`struct trapframe` for the current process. So we can just read the value of
`%eax` from there. Whew, that was some good forward-thinking.
```c
void syscall(void)
{
    struct proc *curproc = myproc();
    int num = curproc->tf->eax;
    // ...
}
```

Now we just need to do one more thing: call the function that corresponds to
that number. We're gonna use the array of function pointers above, but we have
to be careful: this number was given to us by a user process. A malicious user
process might pass in an invalid number in the hopes of getting the kernel to
carry out some undefined behavior which might lead to an easy exploit. So in
order to keep up good security practices, the kernel should *always* distrust
anything originating from user code and handle it carefully, preferably with
three-inch-thick lead-lined gloves. So let's think about it: what might go
wrong?

First of all, any entries that weren't explicitly initialized above (including
the 0 entry) will have been automatically initialized to zero, i.e. a null
pointer. Also, a number that's bigger than the highest system call number will
make us do an out-of-bounds read from the array, thus possibly executing some
arbitrary kernel code that's stored after the array in memory. So we should
check that (1) the number is greater than 0, (2) it's smaller than the number of
elements in the array, and (3) the entry it points to is not a null pointer.

Finally, the `%eax` register is usually used in x86 to store return values, so
we'll put the return value of the system call function there. If any of the
above checks failed, we'll just print a message to the console and return -1 to
indicate failure.
```c
void syscall(void)
{
    // ...
    if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        curproc->tf->eax = syscalls[num]();
    } else {
        // Invalid system call number
        cprintf("%d %s: unknown sys call %d\n", curproc->pid, curproc->name, num);
        curproc->tf->eax = -1;
    }
}
```

The system call handler function will store its return value in `%eax`; after
that, `syscall()` will return to the line below where it was called in `trap()`.
After executing the rest of the code there, `trap()` will return into `trapret()`,
which ends with an `iret` (interrupt return) instruction to tell the processor
to switch to user mode and resume executing the process's code.

### fetchint

Take a look at the `sys_` functions we imported above: they all have argument
type `void`. But if you think about it, many system calls need an argument: for
example, `open()` needs to know which file to open, `chdir()` needs to know
which directory to open, `kill()` needs a PID to know which process to kill,
etc. So why did we make them all have argument type `void`?

The trouble is that until we get the system call number in `syscall()` above, we
have no way of knowing which function we'll need. And each function takes
arguments with different types, e.g. `open()` might need a string for the file
to open but `kill()` might need an integer for the PID. So there's no way for
the kernel to know which arguments to expect in `syscall()`, even though the
arguments were already pushed on the stack. The task of recovering the arguments
from the stack will have to fall to each of the `sys_` functions. But let's go
ahead and make their lives a little easier by setting up some nice helper
functions now.

The system call functions might take integers, strings, or pointers, so we'll
need functions to fetch each of those types. `fetchint()` is one example; it
takes a user virtual address (an integer argument's location in memory) and a
pointer to an integer where we can store the integer we find. Then it returns 0
if it was able to find it, or -1 if it failed.

Just like `syscall()` above, we need to treat anything passed from user space
with extreme caution. A user process that tries to read or write memory outside
its address space will cause a segmentation fault or page fault and be killed,
but the kernel has free reign over memory, so a malicious process might try to
trick the kernel into doing that *for* it by putting its "argument" outside of
the user's address space. So we have to start by checking that the entire 4
bytes of the integer is inside the process's address space.
```c
int fetchint(uint addr, int *ip)
{
    struct proc *curproc = myproc();

    if (addr >= curproc->sz || addr + 4 > curproc->sz) {
        return -1;
    }
    // ...
}
```

Now we can just cast the address to a pointer, dereference it, and store the
value in `*ip`.
```c
int fetchint(uint addr, int *ip)
{
    // ...
    *ip = *(int *)(addr);
    return 0;
}
```

Note that we can use an address like `addr` which will be in the lower half of
memory because traps don't perform a full context switch, so we're still using
the process's page directory even though we're in kernel mode (ring 0). If we
had switched to a kernel page directory, we'd have to call `walkpgdir()` or
`uva2ka()` to figure out the corresponding kernel virtual address for `addr`.

Now hopefully, if you've taken anything away from my past rants about undefined
behavior in C, you noticed something wrong with this function. If you didn't,
take another look; I'll wait.

Did you see it? We're dereferencing `addr` without checking that it's not null,
so if the user passed in a null address, we'd dereference a null pointer! We
also dereference `ip` without a similar check, but at the very least `ip` is
passed in by the kernel.

This could be very dangerous -- in general, it's undefined behavior in C, but
now that we've seen the code for handling traps, we're actually at a point where
we can figure out what would happen in xv6 if a null pointer gets dereferenced,
so let's take the opportunity to think about it for a bit.

First, what would happen if the kernel dereferenced a null pointer? Well, if the
kernel is currently using `kpgdir` as a page directory, the address 0 isn't
mapped to anything, so when the paging hardware goes to figure out which physical
address corresponds to the kernel virtual address 0, it would fail and generate
a "General Protection Fault" (trap number 13, or `T_GPFLT`). That would start
running the trap handler code, which would eventually get to the `switch`
statement in `trap()` (see the last post). Trap number 13 would fall under the
`default` case, and the if statement there would recognize that it originated in
the kernel. So it would print an error message to the console, then panic.

Okay, what if we're using a process's page directory, e.g. during a system call?
Address 0 is in the lower half of memory, so it's a user virtual address. The
result will depend on whether that page and its page table are mapped in the
process's page directory. If they are, then dereferencing a null pointer might
be fine after all. But if they're not mapped, dereferencing a null pointer will
cause a General Protection Fault. This time, `trap()` would print an error
message to the console, then mark the process to be killed.

Now, killing a process or causing a kernel panic might not sound like a huge
deal. In fact, xv6 does a great job here by killing a process that might have
dereferenced a null pointer or caused the kernel to do so. A kernel panic would
be much worse -- think about how annoying it would be if that PDF you downloaded
from that one sketchy website installed some malware that made your kernel panic
all the time -- the OS would become unusable. In fact, this is an example of a
"denial of service" vulnerability -- a malicious process might not be able to
read or write arbitrary memory or execute arbitrary code, but it can still keep
you from using your machine the way you expect to.

Just like `uva2ka()`, this function will only get called by one other function
(we'll see it soon), so it just so happens that under the current xv6 code,
it'll all be okay because it should never get passed a null pointer. But
everything from my rant about `uva2ka()` applies here: if you add any kernel
code that calls this function, be *VERY* careful and add your own null checks.

Okay, deep breath now. /rant.

### fetchstr

Fetching a string argument is tricky too; strings in C are just pointers to an
array of characters that ends in nul, i.e. `'\0'`, so this time we have to make
sure that both the pointer *and* the entire string are in the user's address
space; otherwise, we could unwittingly read from some arbitrary memory location
and pass the data back to the user process.

So we'll start by making sure the pointer itself is in a valid address:
```c
int fetchstr(uint addr, char **pp)
{
    struct proc *curproc = myproc();

    if (addr >= curproc->sz) {
        return -1;
    }
    // ...
}
```

Now we'll store the string pointer in `*pp`. We'll also get a pointer to the end
of the process's virtual address space so we can make sure the entire string is
inside its bounds.
```c
int fetchstr(uint addr, char **pp)
{
    // ...
    *pp = (char *) addr;
    char *ep = (char *) curproc->sz;
    // ...
}
```

How can we check if the entire string is inside user memory? Well, a string ends
with a nul byte, `'\0'`, so we just have to start scanning the memory starting
from `*pp` up to `ep` until we find a zero byte. If we find one in that range,
then the entire string is in user memory and we can return its length to
indicate success; otherwise the string overflows past the end of the process's
virtual address space, so we should return -1 to indicate failure.
```c
int fetchstr(uint addr, char **pp)
{
    // ...

    // Scan for a nul byte inside process's address space
    for (char *s = *pp; s < ep; s++) {
        if (*s == 0) {
            // If nul byte found, return the length
            return s - *pp;
        }
    }
    // String is not nul-terminated inside process's memory, so report failure
    return -1;
}
```

Note that again, we're dereferencing `pp` and `addr` without any null checks
(where `addr` is definitely the bigger concern, since it's user-generated), and
again, it's gonna work out okay (a misbehaving process will just get killed),
but once more: be careful if you use this function for your own kernel hacks.

### argint

This is the main function that the `sys_` system call functions will use to
recover an integer argument; it's basically just a wrapper for `fetchint()`. The
arguments are an integer `n` to say we want the nth integer argument, and a
pointer `ip` to store the recovered argument in. We have to call `fetchint()`
with an address argument, so the main task now is to figure out where in memory
the nth integer argument should be.

We're gonna have to use the x86 function call conventions again. Remember how
whenever we call a function in x86, its arguments get pushed onto the stack in
reverse order (i.e., from right to left), so that the first argument is at the
top of the stack (i.e., lowest memory address)? Then we push a return address
(`%eip`) and the old stack base pointer `%ebp`. Normally, the stack pointer
would just keep going on to the next slot on the callee's stack, but in this
case the code in `alltraps()` saved all the registers (including the stack
pointer `%esp`) in a `struct trapframe` before calling `trap()` or `syscall()`.

That means we can recover the old value of `%esp` from the trap frame and look
one spots below that on the stack (i.e., 4 bytes higher in memory, since `int`s
are 4 bytes) to get the first (`n = 0`) argument. The second argument (`n = 1`)
would be 8 bytes higher than `%esp`, and so on. Pretty neat.

Okay, now that we've got that down, the code for this function is pretty
straightforward.
```c
int argint(int n, int *ip)
{
    return fetchint((myproc()->tf->esp) + 4 + 4*n, ip);
}
```

### argptr

Some of the system call functions will have pointer arguments, so this function
recovers them. Pointers are 4 bytes in x86, so we can use `argint()` to get the
pointer itself before performing some additional checks to make sure the pointer
and the address it points to are valid.

The arguments are `n` (to retrieve the nth function argument), a pointer `pp` to
an address where we can store the retrieved pointer, and the size of the block
of memory that the retrieved pointer points to.

Let's start off by just retrieving the value of the pointer as an integer using
`argint()`; that'll make sure that the number `n` is valid.
```c
int argptr(int n, char **pp, int size)
{
    struct proc *curproc = myproc();

    int i;
    if (argint(n, &i) < 0) {
        return -1;
    }
    // ...
}
```

Now we have to make sure that the pointer we just retrieved is itself valid,
i.e. that the size is nonnegative and the beginning and end of the memory block
it points to are both within the process's address space.
```c
int argptr(int n, char **pp, int size)
{
    // ...
    if (size < 0 || (uint) i >= curproc->sz || (uint) i + size > curproc->sz) {
        return -1;
    }
    // ...
}
```

Finally, we can store the pointer in `*pp` and return 0.
```c
int argptr(int n, char **pp, int size)
{
    // ...
    *pp = (char *) i;
    return 0;
}
```

### argstr

A string is just a pointer in C, so we can recover the pointer's value using
`argint()` again, then pass it to `fetchstr()`. The former will make sure `n` is
valid, and the latter will make sure the string is nul-terminated and resides
entirely in the process's address space.
```c
int argstr(int n, char **pp)
{
    int addr;
    if (argint(n, &addr) < 0) {
        return -1;
    }
    return fetchstr(addr, pp);
}
```

## sysproc.c

So now we know how `syscall()` will route a system call trap to the right `sys_`
function, and we've seen how those functions can recover arguments from the
process's stack. Let's see some examples in action; most of these will be simple
wrapper functions.

### sys_fork

All the hard work here is gonna be done by `fork()`, which will create a new
child process by cloning the parent process's virtual address space. We don't
need any arguments for this, so we'll just call `fork()`.
```c
int sys_fork(void)
{
    return fork();
}
```

### sys_exit

`exit()` closes out a process, but it puts it in the `ZOMBIE` state so that the
parent process can call `wait()` to find out it's done running. `exit()` should
never return, so we'll add a return value here to make the compiler happy, but
it should never get executed.
```c
int sys_exit(void)
{
    exit();
    return 0;
}
```

### sys_wait

This system call is the parent process's counterpart to `exit()`; it'll do as
its name says and wait until the child process exits.
```c
int sys_wait(void)
{
    return wait();
}
```

### sys_kill

The `kill()` system call sounds like a more aggressive version of `exit()`:
after all, we're killing another process against its will, right? But in reality
it would be way too complicated to do that: the process might be running on
another CPU, midway through updating some kernel data structure, or about to
wake up another process that's asleep. Killing it by force might screw up a lot
of other things.

So instead `kill()` just tags it with the `killed` field in its `struct proc`;
eventually either the process will call `exit()` on its own, or it'll generate
another trap, at which point the code in `trap()` will call `exit()` on it.

`kill()` needs an integer argument: the process ID for the process we wish to
kill. So now we can see the payoff of writing those functions above.
```c
int sys_kill(void)
{
    int pid;
    if (argint(0, &pid) < 0) {
        return -1;
    }
    return kill(pid);
}
```

### sys_getpid

The `getpid()` system call is so simple that it doesn't even have another
function for this `sys_getpid()` to call. We'll just return the PID for the
current process.
```c
int sys_getpid(void)
{
    return myproc()->pid;
}
```

### sys_sbrk

If you're not familiar with system calls like `brk()` and `sbrk()` on Unix
systems, here's what they do: they grow or shrink the virtual address space of a
process. `brk()` sets its new size to a specific maximum address; `sbrk()` grows
or shrinks the process by a certain size in bytes and returns its old size.
They're mostly used to implement higher-level memory management functions like
`malloc()`. Heh, "high-level" probably isn't high on your mind when you think of
adjectives for `malloc()`, right? Anyway, xv6 only has `sbrk()`, so let's check
out its `sys_` wrapper function.

We'll need an integer argument (the number of bytes to grow or shrink by), so
let's grab that.
```c
int sys_sbrk(void)
{
    int n;
    if (argint(0, &n) < 0) {
        return -1;
    }
    // ...
}
```

Now we can use `growproc()` from our posts on paging to grow the process by `n`
bytes. But we want to return the old size, so we'll have to grab that before we
change it with the call to `growproc()`.
```c
int sys_sbrk(void)
{
    // ...
    int addr = myproc()->sz;

    if (growproc(n) < 0) {
        return -1;
    }

    return addr;
}
```

### sys_sleep

The `sleep()` function is pretty interesting; we'll get to the implementation
details later, but let's talk about the broad strokes now. You might be familiar
with the `sleep()` system call in Unix systems; you pass it an integer (usually
in milliseconds) and it puts your process to sleep (i.e., leaves it inactive or
not running) for that amount of time.

However, `sleep()` plays a dual role in xv6: the kernel will call `sleep()` for
processes that need to wait while something else happens, e.g. waiting for a
disk to read or write data. That way the processes don't end up idly spinning in
a loop or something and wasting valuable CPU time.

Implementing that is tricky; there's no way to know how long it would take for
whatever condition the process is waiting on to be satisfied, so it's not like
we can just stick in a random amount of time in the call to `sleep()` and hope
the condition is satisfied by then. So instead the `sleep()` function will just
"put a process to sleep" (read: make its state `SLEEPING` so it can't be run by
the scheduler) on a *channel*, which is just an arbitrary integer. Then later on
the kernel can wake up any processes sleeping on that channel. So for example,
the kernel can put a process waiting on the disk to sleep using a specific
channel that's assigned to the disk; then when the next disk interrupt occurs it
can wake up any processes that might be sleeping on the disk channel.

Okay so that's all well and good for the kernel's use of `sleep()`. But what
about the regular old `sleep()` system call? The argument is an integer that
represents the number of ticks to sleep for; how are we gonna turn that into a
channel to sleep on?

The answer is pretty neat (at least I think so): we'll set the channel to the
address of the `ticks` counter. Remember, `ticks` is a global variable that gets
incremented with every timer interrupt. Go check out the code in `trap()` again:
each timer interrupt sends a wakeup call to any processes that might be sleeping
on the `&ticks` channel. That should wake the process at every timer interrupt.
Then we'll just stick that inside a for loop so it keeps sleeping forever until
the right amount of ticks have passed.

Let's start by retrieving the integer argument, which is the number of ticks to
sleep for.
```c
int sys_sleep(void)
{
    int n;
    if (argint(0, &n) < 0) {
        return -1;
    }
    // ...
}
```

That argument `n` is a relative count, since a user process won't necessarily
know how many ticks have already gone by. So let's get the current tick count
before we put the process to sleep.
```c
int sys_sleep(void)
{
    // ...
    acquire(&tickslock);
    uint ticks0 = ticks;
    // ...
}
```

Now we just have to write that while loop I mentioned above to put the process
to sleep until `n` ticks have passed. Since we started counting at `ticks0`, the
condition should be satisfied when `ticks - ticks0 == n`.

Two more details: first, we'll add a check inside the while loop to see if the
current process has been tagged to be killed; if so, we'll just return -1 so we
can hasten the process's actual death by letting it run more code so the kernel
will call `exit()` on it at the next trap. Second, the function `sleep()` takes
another argument in addition to the channel: a lock. It'll release the lock for
us and reacquire it before waking up so that a sleeping process doesn't hog a
lock when it doesn't need it.
```c
int sys_sleep(void)
{
    // ...
    while (ticks - ticks0 < n) {
        if (myproc()->killed) {
            release(&tickslock);
            return -1;
        }
        sleep(&ticks, &tickslock);
    }
    release(&tickslock);
    return 0;
}
```

### sys_uptime

The `uptime()` system call just returns the amount of ticks that have passed
since the system started. This is another one that's so simple it doesn't need
another function, so we'll take care of it all here.

We just acquire the lock for `ticks`, get its current value, release the lock,
and return the value we got.
```c
int sys_uptime(void)
{
    acquire(&tickslock);
    uint xticks = ticks;
    release(&tickslock);
    return xticks;
}
```

## Running System Calls from User Code

We have system calls now! Well, not quite -- we still have to check out the
actual functions like `exit()`, `sleep()`, `kill()`, etc. Plus, we only saw the
`sys_` wrapper functions for *some* of the system calls here; the rest are in
[sysfile.c](https://github.com/mit-pdos/xv6-public/blob/master/sysfile.c), which
we'll get to after we understand the xv6 file system.

But let's pause for a second and think about how a user process will send a
system call. Like let's say you're writing some C code for a user program that
will run on xv6 and you want to create a child process with `fork()`. What
should you do?

Well, if you were coding for a Unix system like Linux or macOS, you'd just write
a call to `fork()` in your code. But that can't be right in xv6, can it? After
all, `fork()` is a kernel function, to be run in kernel mode with a current
privilege level of 0. Plus, isn't it supposed to be called by `sys_fork()`? So
should we call that?

None of these options will work. Well, yes, you do end up just calling `fork()`,
but it's *not* the kernel function `fork()`, so if you're expecting that one,
you'll be surprised when it doesn't behave the way you want it to. You won't be
able to use any kernel code at all in your user program for xv6. This is a
mistake I've seen a *lot* of people make in their xv6 OSTEP projects, so bear
with me for a second while I explain why you can't do it; feel free to skip the
next section on the Makefile if you already know why.

### Makefile

To see why, let's check out the xv6
[Makefile](https://github.com/mit-pdos/xv6-public/blob/master/Makefile) to see
how xv6 is actually compiled, built, and run. There's a ton of stuff in there,
but take a second to think about this: how do you usually run xv6? I bet it's
a command like `make qemu` or `make qemu-nox`, right?

If you're not familiar with Makefiles, here's a quick primer: each command like
`make qemu`, `make clean`, etc. is specified in the Makefile with a rule that
looks like this:
```make
mycmd: dependency1 dependency2 ...
    build_cmd1
    build_cmd2
    # ...
```
So if I run `make mycmd`, the `make` program will check that `dependency1`,
`dependency2`, etc. are up to date; if they're not, it'll update them by looking
up *their* rules and executing those to update them. Then it'll execute
`build_cmd1` on the shell, followed by `build_cmd2`, etc.

Okay, I know that might be confusing, so let me simply the `make qemu` command a
bit to make it more readable (note that I cut a lot of stuff out here, so don't
try to run xv6 with what I wrote below).
```make
# ...
qemu: fs.img xv6.img
    qemu -drive file=fs.img,index=1 -drive file=xv6.img,index=0
# ...
```
This just says that in order to run `make qemu` when you type it on the
terminal, the `make` program first has to make sure that both `fs.img` and
`xv6.img` are fully up to date. Then once they are, it can just run the shell
command `qemu` with the options `-drive file=fs.img,index=1` and
`-drive file=xv6.img,index=0`. Those options are just regular flags like the
ones you're probably used to with stuff like `ls -a` or `rm -rf`. In this case,
they tell `qemu` to use the files `fs.img` and `xv6.img` as virtual hard drives,
with `xv6.img` as disk number 0 and `fs.img` as disk number 1.

Okay, let's check out the `make` command for `xv6.img` next.
```make
# ...
xv6.img: bootblock kernel
    # some dd commands here
# ...
```
Hey, that's interesting, we already saw `bootblock` in a prior post. That's the
one we get when we compile the boot loader. `kernel` is, well, all the kernel
code. The `dd` command is often used in Unix systems to format and set up disks;
the details aren't important here, so I left them out for now. The point is that
the boot loader got compiled separately from the kernel code, remember? But
their machine code files get smushed into the same (virtual) disk together as
`xv6.img`, which will be disk 0 when we run in `qemu`.

Not let's check out the (slightly simplified) `make` command for `fs.img`.
```make
# ...
UPROGS = cat echo forktest grep init kill ln ls # ...

fs.img: mkfs README $(UPROGS)
    ./mkfs fs.img README $(UPROGS)
# ...
```
Okay, so `UPROGS` is just a list of all the user programs. Each of those gets
compiled separately; e.g. if you look in their source code, you'll see each one
has its own `main()` function. Then the shell command says to run `mkfs` to
create a file system called `fs.img` with `README` and all the user programs as
files.

The point of this detour is this: the boot loader gets compiled as a single
unit, as does the entire kernel code. But the user programs are compiled one at
a time. So if you write a user program for xv6, you should add it to the list in
`UPROGS` (as well as in `EXTRA`) and expect it to get compiled individually and
stuck onto the `fs.img` disk.

That means there's no way for a user program to call into any kernel code; the
linker wouldn't even be able to match up the call to the right function. So no
user program will ever be able to call functions like (the kernel's) `fork()`.
Think about it: if you write a program in C and compile it to run on Linux, do
you expect to have to recompile the entire Linux kernel just to run your one
little program? No, right?

But certainly we can't just expect every single program ever to be totally self-
contained. You also don't have to rewrite and recompile all of `malloc()` every
time you write a C program. So operating systems provide libraries for users to
include and call in their programs. Aha! So all we need to do in order for
user processes to execute system calls is to provide a library. That library is
[usys.S](https://github.com/mit-pdos/xv6-public/blob/master/usys.S).

### usys.S

Let's trace back to the beginning of a trap. In order to execute a system call,
we're supposed to send the processor an `int` instruction with a specific trap
number; that would be `int 64` for system calls on xv6. We're also supposed to
stick the system call number in the `%eax` register. Let's say we want to call
`fork()`. According to
[syscall.h](https://github.com/mit-pdos/xv6-public/blob/master/syscall.h), the
system call number for fork is `SYS_fork`, or 1. In order to send a specific x86
instruction and manipulate individual registers, we'll have to write our system
call library in assembly. Here's what it would look like for the `fork()` system
call:
```asm
.globl fork
fork:
    movl    $1, %eax
    int     $64
    ret
```

Okay, that's easy enough, but we have 21 of these to write, and it would be
pretty easy to make a mistake or write the wrong system call number. Let's
automate it instead with a C preprocessor macro. We've seen plenty of examples
of defining simple constants with `#define` directives for the preprocessor, but
we haven't looked at them too closely until now.

The C preprocessor is a piece of software that edits C (or assembly) code before
it's compiled. Preprocessor directives like `#define A 5` create macros that are
expanded to replace every instance of `A` in the code with the number 5;
directives like `#include "header.h"` expand such that they essentially copy-
paste all the code in the file `header.h`. We can also create function-like
macros like the `P2V()` and `V2P()` macros we've used often by adding a
parameter inside parentheses; unlike functions, these will be expanded *before*
compilation to paste the code into every instance of its use, thus avoiding the
usual overhead associated with a function call. Function-like macros are also
generic, in a sense, since they don't require specifying parameter types or
return types (as long as it works within the places where the macro will be
used). Note that there are some drawbacks: macros aren't type-checked, they can
evaluate their arguments more than once, we can't use pointers to them like we
can with functions, and they can result in larger code.

We're gonna use a function-like macro here to create the assembly code for each
system call function so that it gets expanded before the code is assembled.
We'll use `T_SYSCALL` instead of 64 in the code above, and `SYS_fork` (or its
equivalent for each system call) for the system call number. We'll have to
replace the part after the underscore in `SYS_` with the name of the system call
function; we can do that with the token-pasting operator `##`, which glues two
tokens together to form a single token. Also, macros must be defined on a single
line, so we'll escape the newline characters with `\` and end each assembly line
with a semicolon.
```asm
#include "syscall.h"
#include "traps.h"

#define SYSCALL(name) \
    .globl name; \
    name: \
        movl    $SYS_##name, %eax; \
        int $T_SYSCALL; \
        ret
```

Now we can just invoke the macro on the name of each function we want to create:
```asm
# ...
SYSCALL(fork)
SYSCALL(exit)
SYSCALL(wait)
# and so on ...
```

After the preprocessor runs on the file, the result will look like this:
```asm
.globl fork
fork:
    movl    $1, %eax
    int     $64
    ret

.globl exit
exit:
    movl    $2, %eax
    int     $64
    ret

.globl wait
wait:
    movl    $3, %eax
    int     $64
    ret

# and so on...
```

Great! Now we have 21 functions for the system calls, all written in assembly.
All user programs for xv6 will be compiled together with the code for these
functions: see `ULIB` in the Makefile. So now, a user program can execute a
system call by calling these functions, e.g. `fork()`.

## Summary

After all the preparations are handled by the trap handler functions in the IDT,
`alltraps()`, and `trap()`, system calls get routed to the `syscall()` function,
which uses a system call number to pick the right function out of an array. That
function will have to recover any arguments to the system call before passing it
on to the real system call function later on.

Next up, we'll take a look at some of those system calls; we'll leave the rest
until after we go over xv6's file system.

# System Calls: Processes

In a previous post, I pointed out some of the most important functions a kernel
has to fulfill. System calls take care of two of these: virtualizing resources
via virtual memory and processes, and mediating communication between user-mode
processes and the hardware. We'll wrap up the former now by looking at the
system call functions relating to processes and scheduling.

## proc.c

### fork

Unlike some of the other functions we'll talk about in this post, `fork()` is
used almost exclusively by user code as a system call; the kernel never calls
it. That said, it has an extremely important role: after the first process has
started, it's the only way to create more processes. It does that by copying the
parent process's virtual address space into a new page directory. We haven't
talked about the file system yet, but hopefully you're familiar with file I/O in
Linux, so you know each process has its own list of open files and a current
working directory; `fork()` will clone those as well for the child process.

Let's start off by getting a pointer to the parent process and creating a slot
in the process table for the child process with `allocproc()`. Remember, that
function returns a pointer to the new process's `struct proc`, but it can fail
and return null (e.g., if there is no available slot in the process table, or if
its call to `kalloc()` fails), so we'll need to check for that.
```c
int fork(void)
{
    // Parent process
    struct proc *curproc = myproc();

    // Allocate process table slot for child process
    struct proc *np;
    if ((np = allocproc()) == 0) {
        return -1;
    }
    // ...
}
```
`allocproc()` also sets up the new process's stack so that it'll return into
`forkret()`, then `trapret()`, before context switching into user mode, and sets
the process's state to `EMBRYO`.

Next we need a page directory for the new child process; it should be a copy of
the parent process's page directory. Luckily, we already did the hard work for
this back in the virtual memory posts, so we can just use `copyuvm()` now. That
function can also fail, in which case we'll free the stack that `allocproc()`
created and set the child process's state back to `UNUSED`.
```c
int fork(void)
{
    // ...
    if ((np->pgdir = copyuvm(curproc->pgdir, curproc->sz)) == 0) {
        kfree(np->kstack);
        np->kstack = 0;
        np->state = UNUSED;
        return -1;
    }
    // ...
}
```

Next we'll copy the parent process's size and trap frame; the latter will make
sure the child starts executing after `trapret()` with the same register
contents as the parent. We'll set the child process's parent to, well, its
parent (the current process).
```c
int fork(void)
{
    // ...
    np->sz = curproc->sz;
    np->parent = curproc;
    *np->tf = *curproc->tf;
    // ...
}
```

The two processes will be nearly identical, so we need a way to distiguish them
from user space so that a user program can give different instructions to each.
xv6 follows the Unix convention that `fork()` should return the child process's
PID to the parent and return 0 for the child. The parent's return value is easy;
we'll just literally return the child's PID at the end. But the child didn't
actually call `fork()`, so how can we set a return value that it will see?

Well, the x86 convention is for return values to be passed in the `%eax`
register, right? And that register will be restored from the trap frame before
switching into user mode. So we'll just store the value 0 there.
```c
int fork(void)
{
    // ...
    np->tf->eax = 0;
    // ...
}
```

Next we'll copy all the parent process's open files and its current working
current working directory. The files are stored in a per-process file array
`curproc->ofile` of size `NOFILE`, so we can copy them over with the function
`filedup()` (which we'll see later). The current working directory is in
`curproc->cwd` and can be copied with `idup()`.
```c
int fork(void)
{
    // ...
    for (int i = 0; i < NOFILE; i++) {
        if curproc->ofile[i]) {
            np->ofile[i] = filedup(curproc->ofile[i]);
        }
    }
    np->cwd = idup(curproc->cwd);
    // ...
}
```

Then we'll copy the parent process's name with `safestrcpy()`, defined in
[string.c](https://github.com/mit-pdos/xv6-public/blob/master/string.c). You
might be familiar with the C standard library funtion `strncpy()`; this function
is almost identical, except that unlike `strncpy()` it's guaranteed to nul-
terminate the string it copies. If you haven't seen this kind of thing before,
it's a fairly common practice to write your own safe wrappers for some of the C
standard library functions, especially the ones in `string.h` which are so often
error-prone and dangerous.
```c
int fork(void)
{
    // ...
    safestrcpy(np->name, curproc->name, sizeof(curproc->name));
    // ...
}
```

Finally, we'll set the child process's state to `RUNNABLE` and return its PID
for the parent.
```c
int fork(void)
{
    // ...
    int pid = np->pid;

    acquire(&ptable.lock);
    np->state = RUNNABLE;
    release(&ptable.lock);

    return pid;
}
```

### kill

This is one of the functions that can get called both by the kernel and as a
system call. The kernel will use it to terminate malicious or buggy processes,
and user code can use it as a system call to kill another process too.

We said before that killing a process immediately would present all kinds of
risks (e.g. corrupting any kernel data structures it might be updating, etc.),
so all we're gonna do is give it the ominous mark of death with the `p->killed`
field. Then the code in `trap()` will handle the actual murder the next time the
process passes through there.

The argument is a process ID number, so let's just iterate over the process
table until we find a process with a matching PID; we'll return -1 if we don't
find any.
```c
int kill(int pid)
{
    acquire(&ptable.lock);

    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->pid == pid) {
            // ...
        }
    }

    release(&ptable.lock);
    return -1;
}
```

If we do find a matching process, then we'll set `p->killed`. Also, some of the
calls to `sleep()` will occur inside a while loop that checks if `p->killed` has
been set since the process started sleeping, so let's hasten the process's death
a little by setting its state to `RUNNABLE` so it'll wake up and encounter those
checks faster. There's no risk of screwing up by waking up a process too early,
since each call to `sleep()` should be in a loop that will just put it back to
sleep if it's not ready to wake up yet.
```c
int kill(int pid)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->pid == pid) {
            p->killed = 1;

            if (p->state == SLEEPING) {
                p->state = RUNNABLE;
            }

            release(&ptable.lock);
            return 0;
        }
    }
    // ...
}
```

### sleep

The last post went over the basics of `sleep()` and `wakeup()`; they act as
mechanisms for *sequence coordination* or *conditional synchronization*, which
allows processes to communicate with each other by sleeping while waiting for
conditions to be fulfilled and waking up other processes when those conditions
are satisfied.

Processes can go to sleep on a channel or wake up other processes sleeping on a
channel. In many operating systems, this is achieved via channel queues or even
more complex data structures, but xv6 makes it as simple as possible by simply
using pointers (or equivalently, integers) as channels; the kernel can just use
any convenient address as a pointer for one process to sleep on while other
processes send a wakeup call using the same pointer.

This does mean that multiple processes might be sleeping on the same channel,
either because they are waiting for the same condition before resuming execution
or because two different `sleep()`/`wakeup()` pairs accidentally used the same
channel. The result would be that a process might be woken up before the
condition it's waiting for has been fulfilled. We can solve that problem by
requiring every call to `sleep()` to occur inside a loop that checks the
condition; that way, if a process receives a spurious wakeup call before it
really should have been woken up, the loop will put it right back to sleep
anyway. We saw one example of this in the `sys_sleep()` function, in which the
while loop checked if the right number of ticks had passed.

A common concurrency danger with conditional synchronization in any operating
system is the problem of missed wakeup calls: if the process that's supposed to
send the wakeup call runs *before* the process that's supposed to sleep, it's
possible that the sleeping process will never be woken up again. The problem is
more general than just processes; it applies to devices too.

Imagine this scenario: a process tries to read from the disk; it'll check
whether the data is ready yet and go to sleep (inside a while loop) until it is.
If the disk gets to run first, then the process will just find the data ready
and waiting for it, so it can continue on to use the data. If the process runs
before the disk does, then it'll see the data isn't ready yet and sleep in a
loop until it is; the disk will wake the process up once the data is ready.

But suppose they run at the same time, or in between each other. The process
does its check and finds the data isn't ready, but before it can go to sleep, a
timer interrupt or some other trap goes off and the kernel switches processes.
*Then* the disk finishes reading and starts a disk interrupt that sends a wakeup
call to any sleeping processes, but the process isn't sleeping yet. When the
process starts running again later on, it'll go to sleep -- having already
missed its wakeup call.

The problem is that the process can get interrupted between checking the
condition and going to sleep, right? So why don't we just disable interrupts
there with `pushcli()` and `popcli()`? add a lock there? Ah, but there's another
problem: what if the disk driver is running simultaneously on another CPU?
Disabling interrupts on the process's CPU wouldn't stop the other CPU from
sending the disk's wakeup call too early.

Okay fine, so let's use a lock instead. The process will hold the lock while it
checks the condition and sleeps, and the disk driver will have to acquire the
lock before it can send its wakeup call... Can you see the problem here? If the
process holds the lock while it's sleeping, the disk driver will never be able
to acquire the lock in order to wake it up. That's a deadlock.

HEAD. DESK.

Ugh, okay, fine, you got me. So let's use a lock, but let's have `sleep()`
release it right away, then reacquire it before waking up; that way the lock
will be free while the process is sleeping so the disk driver can acquire it.
Done, right? Everybody's happy?

Nope. Now we're back to the original problem: if the lock gets released inside
`sleep()` before the process is actually sleeping, then the wakeup call might
happen in between those and get missed.

@*#&@#$**&@#%$!!!

So we need a lock. And we can't hold the lock while sleeping, or we'd get a
deadlock. But we also can't release it before sleeping, or we might miss a
wakeup call. So... ???

See, I told you: concurrency is your worst nightmare. Ever since we decided we'd
like our operating systems to do more than run a single basic process at a time,
we introduced all *kinds* of problems we have to reason through. Let's check out
how xv6 actually writes the `sleep()` function and think through it ourselves
and try to understand if it manages to solve this problem.

We'll start by making sure of two things: (1) this CPU is currently running a
process and not the scheduler (which can't ever go to sleep), and (2) the caller
passed in a lock (which can be any arbitrary lock).
```c
void sleep(void *chan, struct spinlock *lk)
{
    struct proc *p = myproc();
    if (p == 0) {
        panic("sleep");
    }
    if (lk == 0) {
        panic("sleep without lk");
    }
    // ...
}
```

Next we need to release the lock and put the process to sleep. That will require
modifying its state, so we should now acquire the lock for the process table.
But if the lock that the process is already holding *is* the process table lock,
then trying to acquire it again would cause a panic, so let's add a check for
that; if we're already holding it then we'll keep using it and we don't need to
release it.
```c
void sleep(void *chan, struct spinlock *lk)
{
    // ...
    if (lk != &ptable.lock) {
        acquire(&ptable.lock);
        release(lk);
    }
    // ...
}
```

Okay, now it's nap time for this process. We just update its channel to `chan`
and its state to `SLEEPING`, then call `sched()` to perform a context switch
into the scheduler so it can run a new process. We *have* to be holding the
process table lock before calling `sched()`, remember?
```c
void sleep(void *chan, struct spinlock *lk)
{
    // ...
    p->chan = chan;
    p->state = SLEEPING;
    sched();
    // ...
}
```

When the process wakes up later on (if indeed it turns out that the code here
works and doesn't miss any wakeup calls), it'll eventually be run by the
scheduler, at which point it will context switch back here. So at that point
we'll reset its channel and reacquire the original lock before returning.
```c
void sleep(void *chan, struct spinlock *lk)
{
    // ...
    p->chan = 0;
    if (lk != &ptable.lock) {
        release(&ptable.lock);
        acquire(lk);
    }
}
```

Okay, well I don't know about you, but I'm still not convinced that this
implementation won't miss any wakeup calls. After all, we release the original
lock before putting the process to sleep, right? We're holding the process table
lock at that point, which at least means that interrupts are disabled, but the
process that will wake this one up might already be running on another CPU and
might send the wakeup signal in between releasing the original lock and
updating this process's channel and state. Hmm... Well, as always, xv6 is
brilliant, so we'll see how this gets solved in the code for `wakeup()`.

But wait! Before we move on, I have a warning for you about using this function
in your own code when you start hacking away at xv6. Remember that when we first
talked about deadlocks, we saw we can cause a deadlock if two processes acquire
two locks in opposite orders? If process 1 tries to acquire lock A, then lock B,
and process 2 simultaneously tries to acquire lock B, then lock A, then the end
result is that process 1 will acquire lock A and process 2 will acquire lock B,
but neither will be able to acquire the other lock since it's already being held.

If you look at the code above, the process that called `sleep()` must have
already been holding a lock `lk`, then `sleep()` acquires `ptable.lock` before
releasing `lk`. You know what that means: there's potential for a deadlock. So
in order to avoid that, you should make sure that *any* lock you pass in to
`sleep()` must *always* get acquired before `ptable.lock`. If any other function
(or chain of function calls) could potentially acquire `ptable.lock` before `lk`,
then you might end up with a deadlock. As always, the xv6 authors have been
extremely careful to make sure that that never happens in the existing code, so
you'll have to do the same thing for any code you add.

### wakeup

This function is short and sweet because it procrastinates all the work it has
to do by pushing it off to a helper function, `wakeup1()`. It just acquires the
process table lock, calls `wakeup1()`, then releases the process table lock. It
has to grab that lock since it's gonna modify the process's state in the process
table.
```c
void wakeup(void *chan)
{
    acquire(&ptable.lock);
    wakeup1(chan);
    release(&ptable.lock);
}
```
xv6 has to use this kind of a wrapper function for the real wakeup function
`wakeup1()` in order to let processes that are already holding the process table
lock send wakeup calls too.

Okay, now before we go look at `wakeup1()`, let's get back to figuring out
whether xv6's implementation of `sleep()` and `wakeup()` can lead to missed
wakeup calls. Take a look at the code in `sleep()` again where the original lock
gets released -- we have to acquire the process table lock *before* we can
release the other lock. So now there are always two locks in play whenever we
use `sleep()` and `wakeup()`.

Let's go back to the example of a process waiting on a disk read. The process
acquires some disk-related lock first, then checks to see if the disk is done
reading; if not, it'll call `sleep()` inside a while loop. If the disk driver
runs now before the process gets to call `sleep()`, that's okay: the disk driver
also has to acquire the same lock before calling `wakeup()`, so the disk would
just end up spinning idly. Eventually, the process runs again and gets to
call `sleep()`; there, it will first acquire the process table lock before
releasing the original disk-related lock.

So what happens if the disk driver's code runs now? Now the disk would be able
to acquire the original lock, so there's nothing stopping it from calling
`wakeup()`. But the very first thing it has to do there is acquire the process
table lock, which the process is already holding, so it just spins idly again!
There's no way the disk driver could ever beat the process to acquiring this
second lock, because the process already held the first (disk-related) lock
before acquiring the second one (the process table lock). Now the process can
finish going to sleep and switch into the scheduler, which will eventually
release the process table lock. So then the disk driver can acquire it, release
the first lock, and finally send its wakeup call.

Moral of the story? There's no way for xv6 to ever have any missed wakeup calls!
The trick was to use two locks, and acquire the second before releasing the
first. But coming up with that solution isn't as easy as saying "oh, just use
two locks!" The solution only works because of the way the process table lock is
already being handled by so many other parts of the kernel code. For example, if
the context switch into the scheduler wasn't guaranteed to release the process
table lock, then the disk driver in the example would never be able to acquire
it after the process goes to sleep, resulting in a deadlock. The solution works
because of all the design decisions in xv6 up to this point.

### wakeup1

Okay, I'll stop fawning over the intricacies of xv6 concurrency management now
so we can look at how wakeup calls actually happen. Remember, this is a separate
function from `wakeup()` because sometimes the scheduler needs to send a wakeup
call while it's already holding the process table lock. So we're gonna assume
that every function that ever calls this is already holding it.

The implementation here is actually pretty simple now: we'll just iterate over
the process table and set every single process that's sleeping on channel
`chan` to `RUNNABLE`.
```c
static void wakeup1(void *chan)
{
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state == SLEEPING && p->chan == chan) {
            p->state = RUNNABLE;
        }
    }
}
```

Now, there might be multiple processes sleeping on this channel, so this will
wake them all up. For some of those processes, this might be a spurious wakeup,
so again, we should always make sure to call `sleep()` in a loop that checks for
some condition to be satisfied. Even if multiple processes do have their
sleep conditions satisfied, they'll have to reacquire their original lock before
returning out of `sleep()`, so only one of them will do so and the others will
spin until the first one is done.

Why not just wake up the first process we find that's sleeping on `chan`? Then
we could avoid the extra overhead of a bunch of processes waking up, checking a
condition, and going back to sleep, or even spinning idly waiting to reacquire
the lock before returning. The issue is that the channels may not be unique, so
there's no way to know which of all the sleeping processes is the one whose
sleep condition has just been fulfilled. If we wake up the wrong process, it'll
just go back to sleep, but the right process didn't wake up, so that means we've
lost a wakeup call.

### exit

Okay, so we saw above that `kill()` doesn't really kill a process immediately;
it shirks that responsibility and lets `exit()` handle it instead... except
even `exit()` won't really fully kill a process. Whew, a process's death just
keeps getting dragged out forever, doesn't it? It's starting to feel like a
cheesy death scene in a tragedy; I bet the process is tired of suffering the
slings and arrows of outrageous fortune by now.

But it does make sense. Think about what we have to do in order to wrap up a
process and recycle its slot in the process table: we have to close out any open
files and reset its current working directory, free its kernel stack and its
entire page directory, then notify the parent that it's done running.

The trouble comes with freeing the kernel stack and process page directory. This
function runs in kernel mode, so while the user stack in the lower half of
memory will be unused now, the kernel stack is still needed in order to keep
executing the instructions for `exit()`. Also, with the exception of the times
when it's running the scheduling algorithm, the kernel uses the page directory
of the current process. The moment we free that page directory, the very next
memory access will be to an invalid page; the CPU would trigger an exception
then. That exception would eventually get routed to `exit()` again, except, oh
wait, we can't even run any instructions without generating another exception,
because the entire page directory and stack have been freed; that's a double
fault. So then the CPU would try to handle *that* exception, which would cause
the dreaded boogeyman of OS devs around the world: a triple fault. After a fault
triggers a second exception, which itself triggers a third exception, the CPU
just decides that the kernel in its current state doesn't have its shit together
enough to keep running, so it takes over and reboots the whole system. Oops.

Okay, so let's not do that. That means we can't free the kernel stack nor the
page directory until we're running on a different stack/page directory combo.
That could happen in `scheduler()` while we're using the page directory `kpgdir`,
or it could happen while we're running another process. xv6 does it while it's
running the parent process, in the `wait()` system call. If you haven't used
that in Linux before, `wait()` lets a parent process sleep until a child process
is done running. xv6 will use `wait()` to finish cleaning up after an exited
child process too.

Now, the very first process that starts running in xv6 (`initproc`, which loads
and runs the shell) obviously has no parent process, but that's okay because
that one should never exit as long as the system is up. So let's start this
function off by making sure that the process that's exiting isn't the initial
process.
```c
void exit(void)
{
    struct proc *curproc = myproc();
    if (curproc == initproc) {
        panic("init exiting");
    }
    // ...
}
```

Next we'll close all open files and clear the current working directory; again,
we haven't seen the file system functions used here, but we'll get to them soon.
```c
void exit(void)
{
    // ...

    // Close all open files
    for (int fd = 0; fd < NOFILE; fd++) {
        if (curproc->ofile[fd]) {
            fileclose(curproc->ofile[fd]);
            curproc->ofile[fd] = 0;
        }
    }

    // Clear the current working directory
    begin_op();
    iput(curproc->cwd);
    end_op();
    curproc->cwd = 0;

    // ...
}
```

Now we only have one thing left to do: notify the parent process that this
process has exited. If the parent process is currently sleeping in `wait()`,
then we'll need to wake it up. But maybe the parent process is currently in the
middle of executing other code before it gets to `wait()`; we don't want it to
miss the wakeup call... oh wait, but that's okay, remember? The implementations
of `sleep()` and `wakeup()`/`wakeup1()` guarantee that we can't miss a wakeup
call as long as we're holding the right lock; `wait()` will use the process
table lock for that. So let's acquire it now and send a wakeup call.
```c
void exit(void)
{
    // ...
    acquire(&ptable.lock);
    wakeup1(curproc->parent);
    // ...
}
```

Now, remember that a sleeping process needs to check some condition in a loop;
how can the parent process know that the child has exited? Hmm, okay, let's set
the child's state to `ZOMBIE`. That'll also prevent the scheduler from trying to
run it again.

Ah, but hang on a sec... what if the parent process has itself been killed, i.e.
the current process has been orphaned? (Again with the melodrama...) A process
can't run any more user code after `exit()`, so an undead parent process would
never get to call `wait()` to clean up after its children. In that case, we'd
have to find another process that could adopt a child.

So let's just solve that problem now: this process is about to shuffle off its
mortal coil, so let's figure out if it has any children and pass them off to
another process that can keep raising them as its own. But which process is
guaranteed to live long enough to clean up after those children once they die?
Ah, `initproc`, of course! That first process is immortal, so it should be able
to look after any children that this process might leave behind after it makes
its quietus with a bare bodkin.

So we'll iterate over the process table, looking for any processes with parent
process equal to `curproc`; if we find any, we'll have `initproc` adopt them.
If any of our now-abandoned children has already exited before we did, we'll
send a wakeup signal to `initproc` too in case it's sleeping in `wait()`.
```c
void exit(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->parent == curproc) {
            p->parent = initproc;
            if (p->state == ZOMBIE) {
                wakeup1(initproc);
            }
        }
    }
    // ...
}
```

Okay, now it's finally time for this process to find out what dreams may come in
that sleep of death. We'll set its state to `ZOMBIE` and context-switch into the
scheduler, never to return; if something goes wrong and the scheduler *does*
return, we'll panic in order to keep this function from returning into user code
again.
```c
void exit(void)
{
    // ...
    curproc->state = ZOMBIE;
    sched();

    panic("zombie exit");
}
```

### wait

Like we said above, this system call lets a parent process wait for a child
process to exit; it also cleans up after the child process has exited.

First, we don't even know if this process has any children, so we'll have to
check by iterating through the process table and checking each process's parent
to see if it matches the current process. If it does, then we'll check if it's a
zombie, in which case we can clean it up and return its process ID.

We should also deal with two edge cases: first, if the process has no children
at all, and second, if the process does have children but none of them are dead
yet. In the first case, we'll just return -1 to report failure; in the second
case we'll put the current process to sleep until one of its children exits. The
`sleep()` call means we'll have to do these checks inside an infinite loop.

Alright, let's get started by getting the current process and acquiring the
process table lock, then starting an infinite loop.
```c
int wait(void)
{
    struct proc *curproc = myproc();

    acquire(&ptable.lock);

    for (;;) {
        // ...
    }
}
```

Inside the loop, we'll use a variable `havekids` as a boolean to track whether
we've found any child processes. Then we can iterate over the process table,
skipping any processes for which the current process is not the parent. If we
find any children, we'll set `havekids` to 1.
```c
int wait(void)
{
    // ...
    for (;;) {
        int havekids = 0;

        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            if (p->parent != curproc) {
                continue;
            }
            havekids = 1;
            // ...
        }
        // ...
    }
}
```

If we did find a child process, we should check if it's a zombie, in which case
it's time to finish its clean-up. That means freeing its kernel stack and its
page directory and recycling its `struct proc` so that it can be reallocated to
another process later on.
```c
int wait(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            if (p->state == ZOMBIE) {
                int pid = p->pid;

                // Free child's kernel stack
                kfree(p->kstack);
                p->kstack = 0;

                // Free child's page directory
                freevm(p->pgdir);

                // Recycle child's struct proc
                p->pid = 0;
                p->parent = 0;
                p->name[0] = 0;
                p->killed = 0;
                p->state = UNUSED;

                release(&ptable.lock);
                return pid;
            }
        }
        // ...
    }
}
```

Now, if `havekids` is still zero by the time we finish the for loop, that means
the process doesn't have any children, so we should report failure. We'll also
check if the process has been marked as killed in the meantime.
```c
int wait(void)
{
    // ...
    for (;;) {
        // ...
        if (!havekids || curproc->killed) {
            release(&ptable.lock);
            return -1;
        }
        // ...
    }
}
```

Finally, if it *does* have children, but none of them have exited yet, we'll put
the process to sleep. It'll get woken up when a child exits, at which point
it'll restart the outer for loop at the top and start looking through the
process table again.
```c
int wait(void)
{
    // ...
    for (;;) {
        // ...
        sleep(curproc, &ptable.lock);
    }
}
```

## Summary

By now, we've looked at a good chunk of the system calls available in xv6. These
system calls wrap up the mechanisms that xv6 uses to create and exit processes
with `fork()`, `kill()`, `exit()`, and `wait()`, and introduced `sleep()` and
`wakeup()` as a means for (limited) inter-process communication.

So what's left now? The rest of the kernel code we're gonna look at will just
focus on communicating with various hardware devices like the serial port,
console, and keyboard. Those drivers are relatively short, but there's one
device that will require a lot more work: the disk. Storing files on disk and
making sure they persist across reboots require careful planning, and making
files conveniently accessible to users requires an entire system of abstractions
layered on top of each other, along with a whole host of file-related system
calls.

# Sleep Locks

We've used plenty of spin-locks, and a previous post looked at their
implementation in xv6. Spin-locks have pretty harsh performance costs: a process
that's waiting to acquire a lock will just spin idly in a while loop, wasting
valuable CPU time that could be used to run other processes. So far, we've only
seen locks for kernel resources like the process table, page allocator, and
console, for which all operations should be relatively fast, on the order of a
few dozen CPU cycles at most.

Now it's time to look at the disk driver and file system implementation, and
we'll need some locks there too. But disk operations are *slow* -- reading from
and writing to disk might take milliseconds, which is a literal eternity for a
CPU. Imagine a process hogging a spin-lock for the disk while other processes
spin around and around waiting *forever* for the disk to finish writing. It
would be an enormous waste!

Spin-locks were the best we could do at the time, since we didn't have any
infrastructure to support more complex locks, but now we really do need a better
alternative. We also have some more kernel building blocks in place relating to
processes, including a bunch of system calls.

For example, we've seen the `sleep()` and `wakeup()` system calls, which let a
process give up the CPU until some condition is met. Well, hang on a second --
what if that condition is that a lock is free to acquire? Then a process could
sleep while another process holds the lock, and wake up when it's ready to be
acquired; that would let other processes run instead of forcing a process to
spin and spin. xv6 calls these *sleep-locks*, and it's time to find out how they
work.

## sleeplock.h

If we want a process holding a sleep-lock to give up the processor in the middle
of a critical section, then sleep-locks have to work well when held across
context switches. They also have to leave interrupts enabled. This couldn't
happen with spin-locks: it was important that they disable interrupts to prevent
deadlocks and ensure a kernel thread can't get rescheduled in the middle of
updating some important data structure.

Leaving interrupts on adds some extra challenges. First, we have to make sure
the lock can still be acquired atomically; second, we have to make sure that any
operations in the critical section can safely resume after being interrupted.

Let's solve the first problem: how can we make sure a sleep-lock will always be
acquired atomically? Well, if we want to do something atomically, we already
have a solution: spin-locks! So rather than reinventing the wheel, we'll just
make each sleep-lock a two-tiered deal with a spin-lock to protect its
acquisition.

We'll use a `locked` field just like the one all spin-locks have, but then we'll
add a spin-lock to protect it. We'll also make debugging a little easier by
adding a name for the lock and a field for a PID to identify which process is
holding it.
```c
struct sleeplock {
    uint locked;
    struct spinlock lk;
    char *name;
    int pid;
};
```

## sleeplock.c

### initsleeplock

We can initialize a sleep-lock by initializing its guard spin-lock, then adding
a name for it, setting `locked` to false, and the `pid` field to zero.
```c
void initsleeplock(struct sleeplock *lk, char *name)
{
    initlock(&lk->lk, "sleep lock");
    lk->name = name;
    lk->locked = 0;
    lk->pid = 0;
}
```

### acquiresleep

In order to make sure sleep-lock acquisition is atomic, we'll bookend this
function by acquiring and releasing a spin-lock. This will also make sure that
interrupts are disabled during this function but re-enabled when it's done. It
does add some overheard in the form of spinning until this lock is free, but the
code here should be relatively short and fast to execute. What we really want is
to avoid spinning once the sleep-lock is acquired, i.e. spinning *after* this
function is done. So we'll tolerate a little waste here.
```c
void acquiresleep(struct sleeplock *lk)
{
    acquire(&lk->lk);
    // ...
    release(&lk->lk);
}
```

Okay, now we have to do the actual acquisition. We said above that we'd use the
`sleep()` function to avoid wasting processor time. Hopefully you remember one
important detail about `sleep()`: it must always be called inside a while loop
in order to make sure that we don't miss any wakeup calls. So let's check if the
sleep-lock is already being held and go to sleep if it is. We'll need a channel
and a lock for `sleep()` to release, so let's use the pointer to this lock `lk`
as the channel, and the outer spin-lock `lk->lk` as the lock to be released.
```c
void acquiresleep(struct sleeplock *lk)
{
    // ...
    while (lk->locked) {
        sleep(lk, &lk->lk);
    }
    // ...
}
```
It's important to keep the two locks separate in your head right now: `lk` is
the sleep-lock, and `lk->lk` is the spin-lock it uses to protect the sleep-lock's
acquisition. Note that we're checking `lk->locked` here, *not* the spin-lock
`lk->lk` -- this process is already holding `lk->lk`, but we need to acquire
`lk` itself by updating `lk->locked`. Phew, try saying that ten times fast.

Now the process will go to sleep and yield the CPU until the sleep-lock is free.
If multiple processes are sleeping waiting on the same sleep-lock, they will all
wake up at the same time, but all of them have to reacquire `lk->lk` before
returning from sleep, so only one will get to return here and complete the
sleep-lock acquisition. The others will spin a bit longer, then return here only
to find that `lk->locked` is already being held by another process, so the while
loop will put them to sleep again.

Once the sleep-lock is free, the process can exit the while loop and claim the
sleep-lock for itself.
```c
void acquiresleep(struct sleeplock *lk)
{
    // ...
    lk->locked = 1;
    lk->pid = myproc()->pid;
    // ...
}
```
We don't need fancy atomic operations like `xchg` anymore, since the guarding
spin-lock has already made sure that interrupts are disabled and all operations
are effectively atomic. So that's all we need! Now we just release the spin-lock
and return.

### releasesleep

Now that we've seen how a process acquires a sleep-lock, releasing it is easy,
we just do the opposite. We'll set `lk->locked` to zero and clear the `lk->pid`
field. And what's the opposite of `sleep()`? Well, `wakeup()`, of course! That
will check whether there are any processes sleeping on this channel and let them
know they can attempt to acquire the sleep-lock now.
```c
void releasesleep(struct sleeplock *lk)
{
    acquire(&lk->lk);

    lk->locked = 0;
    lk->pid = 0;

    wakeup(lk);

    release(&lk->lk);
}
```

### holdingsleep

This function is even more simple: it just checks whether a sleep-lock is being
held, and if so, whether it's being held by the current process. The first is
done by just checking `lk->locked`; the second is done by checking that `lk->pid`
matches the current process's PID. The result is a boolean stored in a temporary
variable so we can release the guarding spin-lock before returning the result.
```c
int holdingsleep(struct sleeplock *lk)
{
    acquire(&lk->lk);

    int r = lk->locked && (lk->pid == myproc()->pid);

    release(&lk->lk);
    return r;
}
```

## Summary

Okay, that wasn't too bad! It makes sense why we couldn't use sleep-locks in a
kernel without system calls like `sleep()` and `wakeup()`. But xv6 already has
those, so why not use them everywhere? If sleep-locks really do cut down on the
wasted CPU time, can we just go back and replace all the spin-locks with
sleep-locks? Then the only use for spin-locks would be as a guard for the more-
sophisticated sleep-locks.

Hold your horses! It's not that easy. Sleep-locks leave interrupts enabled, so
they can't be used in interrupt handler functions, or inside a critical section
where a spin-lock is being used, since interrupts will be disabled (though spin-
locks can be used inside sleep-lock critical sections). They also can't be used
by kernel threads like the scheduler, since those aren't processes and thus
can't be put to sleep.

Finally, there are some situations in which a sleep-lock might actually add
*more* overhead than a spin-lock: it takes some time to put a process to sleep,
schedule another process, send a wakeup call, schedule the first process again,
and so on, and the process will hold the sleep-lock the entire time. If another
process is waiting on the sleep-lock, it might actually end up waiting longer
than with a spin-lock, although it'll wait in a sleeping state instead of a
running state where it just spins in a loop.

Additionally, sleep-locks can only be used when it's safe to interrupt a process
in the middle of a critical section and wake it up later. Sure, no other process
can acquire the sleep-lock in the meantime, but it's still not great for time-
sensitive operations like getting the current number of `ticks`.

So sleep-locks are great, but their applications are more limited than spin-
locks. The perfect use for them is when a process needs to complete an operation
atomically, but that operation itself might take a very long time. A great
example of that is disk I/O, and we'll see next how xv6 puts them to use in its
file system implementation.

# Devices: Disk Driver

At this point, we've seen how xv6 virtualizes memory and the processor to give
each user process the illusion of a contiguous, near-infinite memory space and a
dedicated CPU to run it; we've also seen how xv6 mediates interactions between
most of a computer's hardware components and user processes via system calls.
But there's one more piece of hardware that's critically important for an OS
that we haven't looked at yet: the disk. All that's left in the kernel code for
us to look at is how xv6 manages data storage on the disk and how it presents
that data to users in a simplified way.

The function of a disk is to provide *persistence* for an operating system. RAM
is volatile memory: it gets erased when the machine is turned off, so any data
stored there is fleeting. A disk allows an OS to store and retrieve data across
shut-offs. The disk driver we'll go over in this post allows the xv6 kernel
direct access to that device so it can read and write data to it.

But unlike other devices, a simple driver isn't enough here. We don't just need
to be able to read and write data; we'd like to present users with a simplified,
accessible framework to navigate that data. Imagine using a computer where you
had to specify which byte of the disk to read or write, then remember that
yourself in order to access it again later. It's madness! Enter file systems;
"files" don't really exist in any real sense on a disk, but the OS can provide
the illusion of discrete, individual files in order to simplify access to data.

We also need to make sure concurrent accesses of the same file don't risk
corrupting the file (or even the entire file system). We need to separate out
kernel data (like the kernel code itself) from user data on the disk, so that a
malicious user process can't just overwrite arbitrary kernel code. Finally,
there's that oh-so-famous line about Unix systems, "everything is a file". We'll
need a way to present "everything" in the elegant abstraction of a file.

All of these abstractions and security checks will require far more code than a
simple driver to implement them, so before we go on to the driver, let's check
out how xv6 will organize its file system to get a preview of what's ahead.

## File System Organization

Laying the abstraction of a complete file system on top of a physical disk will
require several steps. xv6 does this using seven layers. From bottom (direct
hardware interaction) to top (user-facing code), they are:
* Disk driver: reads and writes blocks on an IDE hard drive.
* Buffer cache: caches disk blocks in memory and synchronizes access to them.
* Logging: provides atomic disk writes to mitigate the risk of a crash.
* Inodes: turns disk blocks into individual files that the OS can manipulate.
* Directories: creates a tree of named directories that contain other files.
* Path names: provides hierarchical, human-readable path names in the directory tree structure.
* File descriptors: abstracts OS resources like pipes and devices as files to provide a unified API for user programs.

That's a lot of work to do now, but it'll pay off! The kernel will do all this
labor so that users are free to be lazy later on and can live in blissful
ignorance of the fact that their precious little files actually exist as nothing
but ones and zeroes in totally arbitrary locations on the disk.

Note that hard drives are usually divided into *sectors*, which are physical
divisions (originally referring to literal geometric sectors), traditionally of
512 bytes. Operating systems can then collect these into larger *blocks* which
are multiples of the sector size. xv6 uses 512-byte blocks for simplicity so
that the sector and block sizes match up; I'll use the two terms interchangeably.

On the disk, block 0 usually contains the boot sector, so it's not used by xv6
(but remember the Makefile -- xv6 actually stores the boot loader and kernel
code on an entirely separate physical disk). Block 1 is called the *superblock*
because it contains metadata about the file system like its total size, the size
of the log, the number of files, and their location on the disk. Then the log
starts at block 2 and on.

## buf.h

If you've read any of the previous optional posts on device drivers, you know
that interacting directly with the hardware means all kinds of opaque code with
seemingly-arbitrary port I/O and cryptic magic numbers. Drivers are also specific
to the actual (or virtual) hardware in the machine that xv6 will run on, so it
tends to be less useful for showing general OS concepts -- hence why all the
other device driver posts were optional. That being said, the disk driver nicely
rounds out the rest of the file system code, so I recommend checking it out, but
if you're short on time or bored with all the talk about hardware specs, feel
free to skip to the summary section below.

Reading and writing disk data is super slow, so the second layer in the file
system is the buffer cache, which will store copies of disk blocks in memory for
faster access. But we still have to read from the disk to create that buffer,
and we still have to write any modified data to the disk once we're done, so
we still need a layer below the buffer cache to do that. That layer is the disk
driver; its purpose is to copy data from the disk to the in-memory cache and
vice versa. A single block is represented in the cache as a `struct buf`, defined
in [buf.h](https://github.com/mit-pdos/xv6-public/blob/master/buf.h).
```c
struct buf {
    int flags;
    uint dev;               // device number
    uint blockno;           // block number (same as sector number)
    struct sleeplock lock;  // sleep-lock to protect buffer reads and writes
    uint refcnt;            // how many processes are using this buffer
    struct buf *prev;       // for use with buffer cache doubly-linked list
    struct buf *next;       // for use with buffer cache doubly-linked list
    struct buf *qnext;      // for use with disk driver queue
    uchar data[BSIZE];      // data stored in the buffer
};

#define B_VALID 0x2
#define B_DIRTY 0x4
```

The two constants defined at the bottom are used in the `flags` field; `B_VALID`
indicates that a buffer has been read from disk and should accurately reflect
the sector's contents on the disk, and `B_DIRTY` says we've modified the buffer
but haven't yet updated the on-disk version of a file, so we need to write the
buffer to disk soon.

We'll see later on that the buffer cache uses a doubly-linked list of buffers;
the `prev` and `next` fields are used there. However, the disk driver also
maintains its own queue of buffers that are waiting to be read from or written
to the disk; that's implemented as a singly-linked list using the `qnext` field.

## ide.c

We've already seen some code to read and write disk data in the [boot loader](boot.md);
I know it's been a while, so you can check that out again if you want. We can't
reuse the code there for a few reasons, though: (1) the boot loader has to be
compiled separately from the kernel, so we can't access any of the functions
there, and (2) we need to store data in the buffer cache, so we can't even copy-
paste the code we used before since the boot loader barely even knows what
memory is, let alone a buffer cache.

### ATA Programmed I/O Mode

Modern disk drivers usually talk to the disk via direct memory access (DMA), but
to keep things simple xv6 is just gonna talk to it with port I/O. That's much,
much slower, and it requires active participation by the CPU (which means it
can't do anything else at the same time), but hey, xv6 thinks it's 1995,
remember? So PIO mode is still (relatively) cutting edge. Either way, extreme
performance isn't the goal here, so we'll just have to suck it up.

Okay, let's do a super-quick summary. `inb` is a C wrapper for an x86 assembly
instruction that reads a single byte of data from a port; `outb` writes a byte
to a port. The disk controller chip has primary and secondary buses; the primary
bus sends data on port 0x1F0 and has control registers on ports 0x1F1 through
0x1F7. Port 0x1F7 doubles as a command register and a status port with some
useful flags we can check in order to know what the disk is up to; we saw some
of those before, but I'll give you the full list now.
* Bit 0 (0x01) - ERR (indicates an error occurred)
* Bit 1 (0x02) - IDX (index; always set to zero)
* Bit 2 (0x04) - CORR (corrected data; always set to zero)
* Bit 3 (0x08) - DRQ (drive has data to transfer or is ready to receive data)
* Bit 4 (0x10) - SRV (service request)
* Bit 5 (0x20) - DF (drive fault error)
* Bit 6 (0x40) - RDY (ready; clear when drive isn't running or after an error and set otherwise)
* Bit 7 (0x80) - BSY (busy; drive is in the middle of sending/receiving data)

The disk driver defines some of these with preprocessor macros at the top of the
file.
```c
#define SECTOR_SIZE 512
#define IDE_BSY     0x80
#define IDE_DRDY    0x40
#define IDE_DF      0x20
#define IDE_ERR     0x01
// ...
```

We also saw one command example in the boot loader: sending 0x20 to port 0x1F7
tells the disk to read a sector and send it to us through data port 0x1F0. Now
we'll also use commands to write a sector, as well as to read or write multiple
sectors at once.
```c
// ...
#define IDE_CMD_READ    0x20
#define IDE_CMD_WRITE   0x30
#define IDE_CMD_RDMUL   0xc4
#define IDE_CMD_WRMUL   0xc5
// ...
```

If, for some reason beyond mortal comprehension, you decide you want to know
more about the eldritch secrets of ancient hard drives, you can read [this
resource on ATA disks](https://pdos.csail.mit.edu/6.828/2018/readings/hardware/ATA-d1410r3a.pdf).

After those constants, we find three static global variables: a spin-lock for
accessing the disk, the queue of buffers waiting to be synchronized with their
on-disk counterparts, and a boolean to track whether xv6 is running with only
disk 0 (boot loader and kernel) or with disk 1 (user file system) as well.
```c
// ...
static struct spinlock idelock;
static struct buf *idequeue;
static int havedisk1;
// ...
```

### idewait

This function takes an integer `checkerr` argument that should be a boolean and
waits for the disk to be ready to receive more commands. If `checkerr` is true,
it'll also check whether the status port includes any error flags.

It starts by reading from the disk's status port and looping until the busy
flag is not set but the ready flag is. The bitwise-OR `IDE_BSY | IDE_DRDY`
combines both flags, and the bitwise-AND tests whether either one is set in `r`.
```c
static int idewait(int checkerr)
{
    int r;
    while (((r = inb(0x1f7)) & (IDE_BSY | IDE_DRDY)) != IDE_DRDY)
        ;
    // ...
}
```

Now if `checkerr` is nonzero we have to check that neither the error nor the
drive failure flag is set in the status port. If either one is set, we'll return
-1; we'll return 0 otherwise.
```c
static int idewait(int checkerr)
{
    // ...
    if (checkerr && (r & (IDE_DF | IDE_ERR)) != 0) {
        return -1;
    }
    return 0;
}
```

### ideinit

This function is called by the kernel's `main()` during set-up to initialize the
disk. We start by initializing the disk lock, then tell the I/O interrupt
controller to forward all disk interrupts to the last CPU. We talked about the
`ioapicenable()` function in detail in the post on interrupt controllers.
```c
void ideinit(void)
{
    initlock(&idelock, "ide");
    ioapicenable(IRQ_IDE, ncpu - 1);
    // ...
}
```

Then we wait for the disk to be ready to accept commands (ignoring any error
flags that may be present).
```c
void ideinit(void)
{
    // ...
    idewait(0);
    // ...
}
```

We said above that disk 0 should contain the boot loader and kernel, so we can
assume any machine running xv6 should have that present. However, we need to
make sure disk 1 is present; the
[Makefile](https://github.com/mit-pdos/xv6-public/blob/master/Makefile) includes
some configurations like `make qemu-memfs` under which xv6 can run without a
dedicated disk for the file system, storing files in memory instead.

Port 0x1F6 is used to select a drive. Bits 5 and 7 should always be set, and bit
6 picks the right mode we need to indicate a disk. Bit 4 determines whether we
want to select disk 0 or disk 1. So we can select drive 1 by setting bits 5-7
(0xE0 when combined), then bit 4 (`1 << 4`).
```c
void ideinit(void)
{
    // ...
    outb(0x1f6, 0xe0 | (1 << 4));
    // ...
}
```

Now we need to wait for disk 1 to be ready; we need to handle this as a special
case since `waitdisk()` can't check a specific disk for us, and because an
absent disk 1 would make the while loop there continue forever. So we'll check
the status register 1000 times; if it ever reports that it's ready, we'll set
`havedisk1` to true and break, but otherwise we'll assume disk 1 isn't present
and leave `havedisk1` as zero (i.e., false).
```c
void ideinit(void)
{
    // ...
    for (int i = 0; i < 1000; i++) {
        if (inb(0x1f7) != 0) {
            havedisk1 = 1;
            break;
        }
    }
    // ...
}
```

Finally, we'll switch back to using disk 0 by changing the fourth bit of the
register at port 0x1F6.
```c
void ideinit(void)
{
    // ...
    outb(0x1f6, 0xe0 | (0 << 4));
}
```

### idestart

This is the core function that will read or write a buffer to or from the disk.
It's a `static` function, so it can only be called by other functions in this
file; `ideintr()` and `iderw()` will both use it as a helper function. It takes
a pointer to a buffer, so the first thing to do is make sure that pointer isn't
null. We'll also make sure the buffer's block number is within the maximum limit
set by `FSSIZE`, defined in
[param.h](https://github.com/mit-pdos/xv6-public/blob/master/param.h) as 1000.
```c
static void idestart(struct buf *b)
{
    if (b == 0) {
        panic("idestart");
    }
    if (b->blockno >= FSSIZE) {
        panic("incorrect blockno");
    }
    // ...
}
```

Next we need to figure out which disk sector to read from or write to. Since xv6
uses blocks that are the same size as a sector, this should just be `b->blockno`,
but we'll add a conversion here in case that gets changed later on (especially
if we want higher disk throughput).
```c
static void idestart(struct buf *b)
{
    // ...
    int sector_per_block = BSIZE / SECTOR_SIZE;
    int sector = b->blockno * sector_per_block;
    // ...
}
```

If each block fits exactly one sector, then we'll need to use the single-sector
read and write commands; otherwise we should use the multi-sector versions of
those commands. We'll set `read_cmd` and `write_cmd` to the right versions.
We'll also make sure that there are no more than 7 sectors per block.
```c
static void idestart(struct buf *b)
{
    // ...
    int read_cmd = (sector_per_block == 1) ? IDE_CMD_READ : IDE_CMD_RDMUL;
    int write_cmd = (sector_per_block == 1) ? IDE_CMD_WRITE : IDE_CMD_WRMUL;
    if (sector_per_block > 7) {
        panic("idestart");
    }
    // ...
}
```

Now let's wait for the disk to be ready, ignoring any error flags.
```c
static void idestart(struct buf *b)
{
    // ...
    idewait(0);
    // ...
}
```

Okay, now it's time to brace yourself, because this next part is a hot mess of
port I/O operations with lots of magic numbers. First we'll tell the disk
controller to generate an interrupt once it's done reading or writing by setting
the device control register at 0x3F6 to zero. Then we'll tell it how many total
sectors we want to read or write by writing that number (AKA `sector_per_block`)
to port 0x1F2.
```c
static void idestart(struct buf *b)
{
    // ...
    outb(0x3f6, 0);                 // generate interrupt when done
    outb(0x1f2, sector_per_block);  // number of sectors to read/write
    // ...
}
```

Before sending the read or write command, we have to tell the disk which sector
to read from, using our `sector` variable from above. Let's take a second to
talk about hard drive geometry. A hard drive consists of a bunch of stacked
circular surfaces, where each surface has a corresponding *head* that changes
its position to read or write from the right place on the disk. Each surface has
a number of *tracks*: concentric circles that contain data. If you pick a track
number (i.e. pick a distance from the center of the surfaces) and collect all
those tracks from all the surfaces, you get a *cylinder*.

A sector number acts as a kind of address with each part specifying a different
geometric component, similar to how linear addresses contain a page directory
index, page table index, and offset. The eight most significant bits (24 through
31) identify the drive and/or head that the sector is located on (plus some
flags); bits 8 through 23 identify the cylinder, and bits 0 through 7 pick a
sector within that cylinder. Altogether, these define a 3D coordinate system
that uniquely identifies all sectors on a machine's disks.

Port 0x1F3 is the sector number register, ports 0x1F4 and 0x1F5 are the cylinder
low and high registers, and port 0x1F6 is the drive/head register. We can write
the sector number as `sector & 0xFF`; the cylinder low and high numbers can be
recovered by bitshifting `sector` down by 8 and 16, respectively.
```c
static void idestart(struct buf *b)
{
    // ...
    outb(0x1f3, sector & 0xff);             // sector number
    outb(0x1f4, (sector >> 8) & 0xff);      // cylinder low
    outb(0x1f5, (sector >> 16) & 0xff);     // cylinder high
    // ...
}
```

Now for the drive/head register, we'll use `b->dev` to get the block's device
and `(sector >> 24)` to get the head it's on. Finally, we'll set bits 5-7 as
required (and as mentioned above in `ideinit()`) with 0xE0. Then we can
bitwise-OR all of these together and write them to port 0x1F6.
```c
static void idestart(struct buf *b)
{
    // ...
    outb(0x1f6, 0xe0 | ((b->dev & 1) << 4) | ((sector >> 24) & 0x0f));
    // ...
}
```

Okay, that was the worst of it! Deep breath now. The last part is just sending
the actual read or write command. But how do we know which one we're supposed to
do? The only argument is a pointer to a buffer `b`, not any sort of boolean that
might tell us which to carry out. Well, remember the buffer flag `B_DIRTY`? That
one indicates that a buffer has been modified and needs to be written to disk.
If that flag is set, reading from the disk would overwrite any changes, which
probably isn't what we want. So let's just assume that the `B_DIRTY` flag means
we should write to disk, and the absence of that flag means we should read from
disk.
```c
static void idestart(struct buf *b)
{
    // ...
    if (b->flags & B_DIRTY) {
        outb(0x1f7, write_cmd);
        outsl(0x1f0, b->data, BSIZE / 4);
    } else {
        outb(0x1f7, read_cmd);
    }
}
```
Here `outsl()` is another C wrapper for an x86 instruction; this one writes data
from a string, four bytes at a time.

That's it! This is by far the most cryptic function in the disk driver; the last
two are relatively easy now.

### ideintr

We saw in `idestart()` that we set up the disk to send an interrupt whenever
it's done reading or writing data. Back when we looked at
[trap.c](https://github.com/mit-pdos/xv6-public/blob/master/trap.c), we saw that
the `trap()` function directs all disk interrupts to the handler function
`ideintr()`. It's time to check that one out now.

We'll start by acquiring the disk's spin-lock; note that we don't use a sleep-
lock because this is an interrupt handler function, so interrupts should be
disabled while it runs.
```c
void ideintr(void)
{
    acquire(&idelock);
    // ...
    release(&idelock);
}
```

If we got an interrupt, then it usually means the disk is done with the most
recent request. Those requests are stored in the global `idequeue` linked list,
with the current request at the front of the queue. So we'll get the head of the
queue as `b`, then set `idequeue` to point to the next buffer in the queue. If
the head is null, then we'll just return early.
```c
void ideintr(void)
{
    // ...
    struct buf *b;
    if ((b = idequeue) == 0) {
        release(&idelock);
        return;
    }
    idequeue = b->next;
    // ...
}
```

The read command in `idestart()` didn't specify where to read the data to, so we
do that now. We'll check if the `B_DIRTY` flag was set; if it wasn't (i.e. the
operation was a disk read), then we'll wait for the disk to be ready (without
any errors, using `idewait(1)` instead of `idewait(0)` as we have before) and
read the data into `b->data`.
```c
void ideintr(void)
{
    // ...
    if (!(b->flags & B_DIRTY) && idewait(1) >= 0) {
        insl(0x1f0, b->data, BSIZE / 4);
    }
    // ...
}
```

Next, we set the `B_VALID` flag with a bitwise-OR and clear any `B_DIRTY` flag
with a bitwise-AND and a bitwise-NOT. Then we'll wake up any user process that
went to sleep on a channel for this buffer after requesting a disk I/O operation.
```c
void ideintr(void)
{
    // ...
    b->flags |= B_VALID;
    b->flags &= ~B_DIRTY;
    wakeup(b);
    // ...
}
```

Finally, we'll get the disk started on the next operation, for the next buffer
in the queue.
```c
void ideintr(void)
{
    // ...
    if (idequeue != 0) {
        idestart(idequeue);
    }
    // ...
}
```

### iderw

The `idestart()` function is `static`, so it can't be called by anything outside
of this file; we need to provide a mechanism for both kernel and user threads to
read and write disk data. That's what `iderw()` does. Note that processes should
never call this function directly; it only gets called by the code for the
buffer cache layer of the file system. In other words, processes will use system
calls like `open()`, `read()`, `write()`, `close()`, etc., which in turn will
use functions from higher layers of abstraction, which in turn call functions
from lower layers, and so on, until they reach the buffer cache, which calls
`iderw()` to finally read/write directly from/to the disk.

By the time a process gets to `iderw()`, it should already be holding a sleep-
lock `b->lock` for the buffer `b` it wants to read or write, and either the
`B_DIRTY` flag should be set (to write to disk) or the `B_VALID` flag should be
absent (to read from disk). We'll start off with some sanity checks for those,
and make sure that we're not trying to read from disk 1 if it's not present on
this machine. Then we'll acquire the disk's spin-lock.
```c
void iderw(struct buf *b)
{
    if (!holdingsleep(&b->lock)) {
        panic("iderw: buf not locked");
    }
    if ((b->flags & (B_VALID | B_DIRTY)) == B_VALID) {
        // B_VALID is set, so we don't need to read it; B_DIRTY is not set, so
        // we don't need to write it
        panic("iderw: nothing to do");
    }
    if (b->dev != 0 && !havedisk1) {
        panic("iderw: ide disk 1 not present");
    }

    acquire(&idelock);
    // ...
    release(&idelock);
}
```

There may be other buffers waiting in line in the disk queue, so we have to
append this buffer `b` to the end of `idequeue`. We can do that by setting
`b->qnext` to null, then creating a variable `pp` to traverse the entire queue.
When `pp` points to the last element, we'll set its `qnext` field to point to
`b`.
```c
void iderw(struct buf *b)
{
    // ...
    b->qnext = 0;

    // Traverse the queue
    struct buf **pp;
    for (pp = &idequeue; *pp; pp = &(*pp)->qnext)
        ;

    // Append b to end of queue
    *pp = b;

    // ...
}
```
That traversal might look confusing as all hell, so let's take a closer look.
It defines `pp` as a double pointer: a pointer to a pointer to a `struct buf`.
(If you've seen the interview of Linus Torvalds where he talks about good style
with linked lists, it's similar to the code there; there's a nice summary
[here](https://github.com/mkirchner/linked-list-good-taste).) `pp` starts off
equal pointing to `idequeue`, i.e. the head of the linked list. Each iteration
checks that `pp` points to a valid (non-null) pointer, i.e. the loop will end
when we reach the end of the list. The body of the loop is empty, so none of the
iterations actually do anything; the purpose of the for loop is just to update
`pp` several times. At the end of each iteration, `pp` is updated to point to a
pointer to the next buffer in the queue.

Suppose the last buffer in the queue is `end`. At the end of the for loop, `pp`
will hold the address of `end->qnext`, so `*pp = b` sets `end->qnext = b`. The
double indirection makes it easy to update the last buffer in the queue; without
it, we would have to stop the loop one step earlier when `pp` points to `end`
instead of `end->qnext` then be careful to update the actual buffer at the end
of the queue instead of just updating the local variable `pp`. All in all, it's
just an elegant way to write a linked list traversal in a single line.

Okay, so now our buffer `b` is at the end of the queue. If there are others in
front of it, then `ideintr()` will make sure that each disk interrupt starts the
disk on the next operation. But what if `b` is actually the only buffer in the
queue? In that case, the disk isn't running yet, so we need to get it started
ourselves.
```c
void iderw(struct buf *b)
{
    // ...
    if (idequeue == b) {
        idestart(b);
    }
    // ...
}
```

At this point, we can be confident that the disk will either start our request
now or get to it eventually (if there are other requests in the queue). This
process just has to wait for the disk to finish, so we'll put it to sleep until
the buffer has been synchronized with the disk. We'll check that by making sure
the `B_VALID` flag is present but `B_DIRTY` is not set. The call to `sleep()`
will release `idelock` and reacquire it before returning.
```c
void iderw(struct buf *b)
{
    // ...
    while ((b->flags & (B_VALID | B_DIRTY)) != B_VALID) {
        sleep(b, &idelock);
    }
    // ...
}
```

## Summary

The disk driver handles direct communication with the hard drive, issuing orders
to read or write sectors. It exposes two API functions, `ideintr()` and
`iderw()`. The former is called by `trap()` to handle disk interrupts, while the
latter is called by the code for the buffer cache layer of the file system to
update blocks in the buffer cache with their corresponding sectors on disk. Next
up we'll look at the buffer cache itself, as well as the logging layer, which
provides crash recovery.
