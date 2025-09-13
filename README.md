My Tiny OS
A simple 32-bit "Hello World" kernel for the x86 architecture, built from scratch. This project demonstrates the creation of a minimal operating system that boots using QEMU, featuring a basic bootloader, a C-based kernel, and VGA text output. It follows tutorials from the OSDev.org Wiki and showcases low-level programming, cross-compiler setup, and system boot processes.
Project Overview
This repository implements a minimal operating system with the following components:

Bootloader: A Multiboot-compliant assembly stub (boot.s) that sets up a stack and hands control to the C kernel.
Kernel: A C-based kernel (kernel.c) that initializes a VGA text buffer and prints "Hello, kernel World!" to the screen.
Linker Script: A linker script (linker.ld) to organize the kernel's memory layout for proper loading at boot.
Cross-Compiler: A custom-built i686-elf toolchain to compile and link the kernel for the x86 architecture.

The OS is booted using QEMU, demonstrating fundamental concepts of OS development, including memory management, assembly programming, and hardware interaction.
Architecture
The system follows a simple boot process:

Bootloader (boot.s): Initializes the stack and calls the C kernel's kernel_main function.
Kernel (kernel.c): Clears the VGA buffer and writes text to the screen using a basic terminal driver.
Linker (linker.ld): Ensures the kernel is loaded at the 2MB memory address with proper section alignment.
QEMU: Emulates an x86 environment to run the kernel without physical hardware.

graph TD
    A[BIOS/GRUB] -->|Loads| B[Bootloader: boot.s]
    B -->|Sets up stack| C[Kernel: kernel.c]
    C -->|Initializes| D[VGA Text Buffer]
    D -->|Outputs| E[Hello, kernel World!]
    A -->|Runs in| F[QEMU Emulator]

Prerequisites
This project requires a Linux-based environment (tested on Debian-based distributions like Ubuntu or Linux Mint). The following tools are needed:

Build Tools:
sudo apt update
sudo apt install build-essential bison flex libgmp-dev libmpc-dev libmpfr-dev texinfo


QEMU Emulator:
sudo apt install qemu-system-x86



Setup Instructions
Part 1: Building the Cross-Compiler Toolchain
A cross-compiler (i686-elf) is required to compile code for the x86 target architecture.
1.1 Environment Setup
Set environment variables for the toolchain:
export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"

Add the PATH export to ~/.bashrc or ~/.profile for persistence.
1.2 Building Binutils

Download Binutils (e.g., binutils-2.42.tar.gz) from GNU Binutils.
Build and install:mkdir $HOME/src
cd $HOME/src
tar -xzf binutils-2.42.tar.gz
mkdir build-binutils
cd build-binutils
../binutils-2.42/configure --target=$TARGET --prefix="$PREFIX" --with-sysroot --disable-nls --disable-werror
make
make install



Verify with: which i686-elf-as.
1.3 Building GCC

Download GCC (e.g., gcc-14.1.0.tar.gz) from GCC website.
Build and install:cd $HOME/src
tar -xzf gcc-14.1.0.tar.gz
mkdir build-gcc
cd build-gcc
../gcc-14.1.0/configure --target=$TARGET --prefix="$PREFIX" --disable-nls --with-newlib --enable-languages=c,c++ --without-headers
make all-gcc
make install-gcc
make CFLAGS_FOR_TARGET="-mno-red-zone" all-target-libgcc
make CFLAGS_FOR_TARGET="-mno-red-zone" install-target-libgcc



Part 2: Creating the Kernel
The kernel consists of three files: boot.s (bootloader), kernel.c (main kernel), and linker.ld (linker script).
Repository Structure
├── README.md
├── boot.s
├── kernel.c
├── linker.ld
├── Makefile

2.1 Bootloader (boot.s)
Sets up a Multiboot-compliant stack and calls the kernel:
.set ALIGN,    1<<0             # align loaded modules on page boundaries
.set MEMINFO,  1<<1             # provide memory map
.set FLAGS,    ALIGN | MEMINFO  # this is the Multiboot 'flag' field
.set MAGIC,    0x1BADB002       # 'magic number' lets bootloader find the header
.set CHECKSUM, -(MAGIC + FLAGS) # checksum required

.section .multiboot
.align 4
.long MAGIC
.long FLAGS
.long CHECKSUM

.section .bss
.align 16
stack_bottom:
.skip 16384 # 16 KiB
stack_top:

.section .text
.global _start
.type _start, @function
_start:
    movl $stack_top, %esp
    call kernel_main

    cli
1:  hlt
    jmp 1b

.size _start, . - _start

2.2 Linker Script (linker.ld)
Organizes the kernel's memory layout:
ENTRY(_start)

SECTIONS
{
    . = 2M; /* Load address */

    .text BLOCK(4K) : ALIGN(4K)
    {
        *(.multiboot)
        *(.text)
    }

    .rodata BLOCK(4K) : ALIGN(4K)
    {
        *(.rodata)
    }

    .data BLOCK(4K) : ALIGN(4K)
    {
        *(.data)
    }

    .bss BLOCK(4K) : ALIGN(4K)
    {
        *(COMMON)
        *(.bss)
    }
}

2.3 Kernel (kernel.c)
Implements a simple VGA terminal driver to print text:
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

#if defined(__linux__)
#error "You are not using a cross-compiler!"
#endif

#if !defined(__i386__)
#error "This needs to be compiled with a ix86-elf compiler"
#endif

enum vga_color {
    VGA_COLOR_BLACK = 0, VGA_COLOR_BLUE = 1, VGA_COLOR_GREEN = 2, VGA_COLOR_CYAN = 3,
    VGA_COLOR_RED = 4, VGA_COLOR_MAGENTA = 5, VGA_COLOR_BROWN = 6, VGA_COLOR_LIGHT_GREY = 7,
    VGA_COLOR_DARK_GREY = 8, VGA_COLOR_LIGHT_BLUE = 9, VGA_COLOR_LIGHT_GREEN = 10, VGA_COLOR_LIGHT_CYAN = 11,
    VGA_COLOR_LIGHT_RED = 12, VGA_COLOR_LIGHT_MAGENTA = 13, VGA_COLOR_LIGHT_BROWN = 14, VGA_COLOR_WHITE = 15,
};

static inline uint8_t vga_entry_color(enum vga_color fg, enum vga_color bg) {
    return fg | bg << 4;
}

static inline uint16_t vga_entry(unsigned char uc, uint8_t color) {
    return (uint16_t)uc | (uint16_t)color << 8;
}

size_t strlen(const char* str) {
    size_t len = 0;
    while (str[len])
        len++;
    return len;
}

#define VGA_WIDTH   80
#define VGA_HEIGHT  25
#define VGA_MEMORY  0xB8000

size_t terminal_row;
size_t terminal_column;
uint8_t terminal_color;
uint16_t* terminal_buffer = (uint16_t*)VGA_MEMORY;

void terminal_initialize(void) {
    terminal_row = 0;
    terminal_column = 0;
    terminal_color = vga_entry_color(VGA_COLOR_LIGHT_GREY, VGA_COLOR_BLACK);
    for (size_t y = 0; y < VGA_HEIGHT; y++) {
        for (size_t x = 0; x < VGA_WIDTH; x++) {
            const size_t index = y * VGA_WIDTH + x;
            terminal_buffer[index] = vga_entry(' ', terminal_color);
        }
    }
}

void terminal_putchar(char c) {
    if (c == '\n') {
        terminal_column = 0;
        if (++terminal_row == VGA_HEIGHT)
            terminal_row = 0; // Simple wrap-around, no scrolling
        return;
    }
    const size_t index = terminal_row * VGA_WIDTH + terminal_column;
    terminal_buffer[index] = vga_entry(c, terminal_color);
    if (++terminal_column == VGA_WIDTH) {
        terminal_column = 0;
        if (++terminal_row == VGA_HEIGHT)
            terminal_row = 0;
    }
}

void terminal_writestring(const char* data) {
    size_t len = strlen(data);
    for (size_t i = 0; i < len; i++)
        terminal_putchar(data[i]);
}

void kernel_main(void) {
    terminal_initialize();
    terminal_writestring("Hello, kernel World!\n");
    terminal_writestring("This is My Tiny OS.");
}

Part 3: Compiling and Linking
Compile and link the kernel using the cross-compiler:

Assemble the bootloader:i686-elf-as boot.s -o boot.o


Compile the kernel:i686-elf-gcc -c kernel.c -o kernel.o -std=gnu99 -ffreestanding -O2 -Wall -Wextra


Link the kernel:i686-elf-gcc -T linker.ld -o myos.bin -ffreestanding -O2 -nostdlib boot.o kernel.o -lgcc



The output file myos.bin is your bootable kernel.
Part 4: Running the OS
Boot the kernel using QEMU:
qemu-system-i386 -kernel myos.bin

A QEMU window will display "Hello, kernel World!" and "This is My Tiny OS.".
Challenges & Solutions

Challenge: The cross-compiler setup failed due to missing dependencies (libmpc-dev, libmpfr-dev).
Solution: Installed required libraries using apt install and verified with dpkg -l.


Challenge: Linker errors occurred due to incorrect GCC configuration.
Solution: Added --with-newlib and -mno-red-zone flags to GCC build to ensure compatibility with a freestanding environment.


Challenge: QEMU failed to boot with a "Multiboot header not found" error.
Solution: Verified the Multiboot magic number and flags in boot.s and ensured proper section alignment in linker.ld.



Lessons Learned

Gained deep understanding of low-level programming, including assembly, memory layout, and VGA hardware interaction.
Mastered cross-compiler setup for i686-elf, critical for OS development.
Learned to debug boot issues using QEMU’s error messages and OSDev.org resources.

Future Improvements

Add a basic keyboard driver to accept user input.
Implement a simple filesystem (e.g., FAT32) for persistent storage.
Extend the kernel to support multitasking or interrupts.
Automate the build process with a Makefile for easier compilation.

Resources

OSDev.org Wiki: Primary guide for this project.
GNU Binutils: Source for Binutils.
GCC Website: Source for GCC.
QEMU Documentation: Emulator setup and usage.

License
This project is licensed under the MIT License. See the LICENSE file for details.
