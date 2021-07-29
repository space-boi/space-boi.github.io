---
layout: post
released: false
---

Writing a minimal operating system from scratch. If not interested in theory skip to [fun](#implementation) part.

# TOC

- [Theory](#Theory)
	- [Compilation overview](#compilation-overview)
	- [Object files](#object-files)
	- [Executables and ELF](#executables-and-elf)
	- [System V ABI](#system-v-abi)
	- [ASLR, GOT and PLT](#aslr-got-and-plt)
	- [Absolute code, PIC and PIE](#absolute-code-pic-and-pie)
	- [Paging and Segmentation](#paging-and-segmentation)
- [Implementation](#Theory)

# Theory

Some quickly summarized theory.

## Compilation overview

As is known CPU doesn't understand more or less anything so in order to run our C code (or some different programming language) we need to convert it to machine code first. Generally this is done by few stages (Compile -> Assemble -> Link -> Load)

First there is **preprocessor** stage which optimize code, remove comments, expands includes, macros and handle directives that with # (`gcc -E skynet.c`). Second stage is **compiling** stage, which
takes source code on input and returns assembly file (`gcc -S skynet.c`). Third stage is called **assembling**, which takes generated assembly and convert it to object files (`gcc -c skynet.s`). These
object files are further merged into a single binary in **linking** stage (`gcc skynet.o`). Even though these three part are seperated and can be called independently, when most people say compilation
they mean entire process of compilation, assembling and linking. Anyway, we have executable program, so now comes **loader** into a scene. Loader is part of operation system kernel and its purpose is to
take executable and puts it into a memory, then jumps to first instruction and start executing machine code.

> TODO: static vs dynamic linking

At this point CPU can execute instructions in machine code previously loaded into a memory. Process of executing program is described by CPU instruction pipeline, classic one is 5 stage pipeline (Fetch
-> Decode -> Execute -> Memory access -> Writeback) used by RISC CPUs but there are also 2 stage pipelines for microcontrollers and 10+ stage pipelines for CISC CPUs. Instruction pipeline is than
important for **pipelinig** technique which allow us to process instructions parallely (fetch next instruction while executing previous one).

More info at: [1](#compilation-1), [2](#compilation-1)

## Object files

Object files are input to linker, they are not directly executable, they contains compiled code, symbol names, constants, imports and exports. There are several types of object files.
[3](#object-files-1), [4](#object-files-1),

`.o` files are basic object files containing metadata for one source code file. We can create object file with `gcc -c` command. Notice that you can pass multiple source code files to gcc which then
create object file for each source file i.e. `gcc -c a.c d.c` generates `a.o` and `d.o` object files.

We can also pack object files into a libraries. Either static or dynamic.

`.a` files are static libraries ('a' in suffix stands for archive) that have bundled multiple `.o` files together. They are tied up with binary in linking stage. Problem is that binaries with
static libraries can easily get very large and you need to recompile binary on any library change.

`.so` on the other hand are dynamic or shared libraries. Binaries never include shared libraries in itself, it just keeps the reference and everything is fetched at runtime. For shared libraries should
be used position independent code so that the library can be mapped on the different location in each program without any overlaping. Shared libraries are equivalent to `.dll` in Windows.

There are also some other object files like library object (`.lo`) or kernel object (`.ko`) but their are not that important.

More info about object file types at: [5](#object-files-1), [6](#object-files-1)

## Executables and ELF

Executable format is the format of executables which can be loaded into memory and executed. Executables are binary files that along data contain instructions for CPU. Some famous executable formats are: ELF (Unix), PE (Windows 95/NT) and MACH-O (MacOS).

### ELF

ELF (Executable and Linkable Format) is file format used for executables and object files in Unix.

#### File Header

Each ELF file starts with file header containing magic "ELF" constant (`0x7F`) and file metadata with sizes, offsets, versions and also type of elf like ET_REL for relocatable files (those with position
independent code), ET_EXEC for statically linked executables and ET_DYN for shared libraries (and also dynamically linked executables which are special kind of shared libraries). There also some other
types of ELF but they are not that important.

#### Program Header

Next part of ELF is Program Header Table is array of structures describing segments. Each **segment** contains 0 or more sections and information about where it should be loaded in virtual memory and
what permission it has. Segments are not needed on link-time but they are used at runtime. What happens with them at runtime are decided by their type, most common are:

- `PT_LOAD` -- segments that are loaded into a memory
- `PT_DYNAMIC` contains list of required libraries
- `PT_INTERP` which contains path to dynamic interpreter, which is responsible for linking at runtime.

#### Section Header

**Sections** are stored in Section Header Table and contains information needed during linking such as code, data, debug info etc. Common sections are:

- `.text` containing code
- `.data` initialised code
- `.rodata` initialised read-only code
- `.bss` uninitialized code

A picture is worth a thousand words:
![ELF](elf.png)

## System V ABI

ABI stands for Application Binary Interface and defines CPU instruction set, file formats, calling conventions, system calls etc. **System V ABI** is ABI used in most Unix operating systems
and some more. Specifically for x86_64 architecture it tell us that data are padded (mostly) to 4-byte alignemt. First 6 arguments are passed via `%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9` and the rest
through stack. Returning value goes though `%rax`. push/pop for stack operations use `%rbp` as base pointer and so on. [11](#abi), [12](#abi)

## ASLR, GOT and PLT

**Address Space Layout Randomization** (ASLR) is technique used for prevention of some types of attacks and exploits by randomizing code, stack, heap and libraries location in a memory. This sounds cool but
there is a little catch, because the libraries will be always loaded in the different location linker cannot provide their addresses. These adressess must be provided at runtime that is task for dynamic
linker/loader which found the symbol we need (i.e. absolute address of printf in libc) and write the address into **Global Offset Table** (GOT). Process of finding those addresses are called lazy linking
or lazy binding because symbol resolution is done at first use of symbol not before. For implementation of lazy binding is used **Procedure Linkage Table** (PLT) which is part of text section and each entry
is small piece of executable code. 

Now let's put it all together with basic hello world, where we call `puts` from libc.
```c
#include <stdio.h>

int main() {
  puts("hello world");
  return 0;
}
```

Now let's took at assembler code for main function:

```asm
0x0000000000001139 <+0>:    push   %rbp
0x000000000000113a <+1>:    mov    %rsp,%rbp
0x000000000000113d <+4>:    lea    0xec0(%rip),%rdi
0x0000000000001144 <+11>:   call   0x1030 <puts@plt>
0x0000000000001149 <+16>:   mov    $0x0,%eax
0x000000000000114e <+21>:   pop    %rbp
0x000000000000114f <+22>:   ret
```

Because we don't know address of `puts`, we need to load it first by calling a `puts@plt`. 

```asm
0x0000000000001030 <+0>:    jmp    *0x2fe2(%rip)        # 0x4018 <puts@got.plt>
0x0000000000001036 <+6>:    push   $0x0
0x000000000000103b <+11>:   jmp    0x1020
```

This small chunk of code in plt first jump to `<puts@got.plt>`. `got.plt` is auxiliary section between `got` and `plt` sections specially for resolving function adressess, but as we stated before
addresses for functions are resolved lazily and this is our first `puts` call, so there is no entry for `puts` in `got`. This cause that `jmp` on `0x1030` ends up on `0x1036` where we push 0 on stack
where 0 is just identificator for `puts` function and jmp on `0x1020` which takes us to dynamic linker. Dynamic linker takes identificator from stack (for us 0 for `puts`) and find demanded address and
puts it into a GOT. On the next call of `puts` the jump on `0x1030` didn't leads us to linker but instead use found address of `puts` in GOT and go directly there.

![PLT and GOT](pltgot.png)

More info at: [12](#aslr-plt-got), [13](#aslr-plt-got), [14](#aslr-plt-got), [15](#aslr-plt-got)


## Absolute code, PIC and PIE

From the point of view of placement in memory there are two types of code: absolute code which and position independent code. 

**Position Independent Code** (PIC) is machine code that use relative references instead of absolute adresses and thus can be run without any changes anywhere in the memory while on the other hand
**absolute code** is code that need to be loaded into specific location in memory otherwise it will not work. **Position Independent Executable** (PIE) is the same concept as PIC used specifically for
executables. If we take look back on ASLR, PIC is needed for ASLR to be able to randomize code placement. 

> **random fact:** both `-fPIC` and `-fPIE` flags are enabled by default in modern gcc

More info at: [16](#pic-pie), [17](#pic-pie)

## Paging and Segmentation

**Paging** is memory managment concept which allows process to work with logical adresses which MMU (Memory Managment Unit) translate into physical adresses. Paging allow process to see a virtual
address space from 0 so there is no need to realocation of machine code. Paging works by dividing virtual adress space to fixed sized block called pages typically 4KiB long and using page table for address
translation. Page is minimal block of memory systel will offer to a process. Paging has multiple benefits like page-level protections that allow processes to access only data paged in their address
space.

**Segmentation** is another concept, it differs from paging by dividing adress space into a variable sized blocks. The translation is done by **Global Descriptor Table** (GDT) which hold information
about segments. Each segment has base (address) a and limit (lenght). In modern operating systems segmentation isn't really used that much and there are only 4 segments: Kernel Code Segment, Kernel Data
Segment, User Code segment and User Data Segment.

![Segmentation and Paging](segmentation-paging.jpg)

## Interrupts



## Syscalls





# Implementation

> Make GDT and interrupts

# References

### General:

[OSDev wiki](https://wiki.osdev.org)  
[Little OS book](http://littleosbook.github.io)  
[James Molloy](http://www.jamesmolloy.co.uk/tutorial_html)  
[Linux insides](https://0xax.gitbooks.io/linux-insides/content/index.html)

#### Compilation

[1] [Compilation Process Cornell](https://www.cs.cornell.edu/courses/cs3410/2019sp/schedule/slides/11-linkload-notes-bw.pdf)  
[2] [Compilation, Assembly, Linking and Loading](https://www.cs.fsu.edu/~baker/opsys/notes/linking.html)

#### Object files

[3] [Object Files](https://wiki.osdev.org/Object_Files)  
[4] [Object File Format](https://docs.oracle.com/cd/E19683-01/817-3677/chapter6-46512/index.html)  
[5] [Static vs Shared libs](https://stackoverflow.com/questions/2649334/difference-between-static-and-shared-libraries)  
[6] [Object File Types](https://stackoverflow.com/questions/30186256/what-is-the-difference-between-o-a-and-so-files/30186852)

#### ELF

[7] [ELF OSDev Wiki](https://wiki.osdev.org/ELF)  
[8] [ELF Format](https://refspecs.linuxbase.org/elf/gabi4+/ch4.intro.html)  
[9] [ELF Wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)

#### ABI

[10] - [System V ABI](https://wiki.osdev.org/System_V_ABI)  
[11] - [System V ABI document](http://www.sco.com/developers/gabi/latest/contents.html)

#### ASLR, GOT, PLT
[12] - [PIC in shared libraries](https://eli.thegreenplace.net/2011/11/03/position-independent-code-pic-in-shared-libraries/)  
[13] - [Dynamic Linker](https://www.bottomupcs.com/dynamic_linker.xhtml)  
[14] - [GOT and PLT for pwning](https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html)  
[15] - [GOT and PLT](https://reverseengineering.stackexchange.com/questions/1992/what-is-plt-got)


#### PIC, PIE

[16] - [PIC and PIE](https://leimao.github.io/blog/PIC-PIE/)  
[17] - [PIC vs PIE](https://stackoverflow.com/questions/28119365/what-are-the-differences-comparing-pie-pic-code-and-executable-on-64-bit-x86-pl)
