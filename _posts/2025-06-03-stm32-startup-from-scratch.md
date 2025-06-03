---
layout: post
title: "STM32 Startup From Scratch"
author: "Allex Franc"
date: 2025-06-03
tags: [stm32, embedded, bare-metal, arm]
excerpt: "Building a minimal boot sequence with less than 50 lines of code"
---

Have you ever wondered what's the minimal code you need 
to boot an STM32F407? Well, I have...

I skipped the traditional path of prototyping something fast.
You know the drill: download the IDE, use their startup code, 
and boom - start building. But no, that would be too easy.

I wanted to have full control of the boot process, not just use it. 
Along the way, we'll decode what those mysterious startup 
files actually do - knowledge that transfers to any embedded system.
So here's how (and why) to build your own from absolute zero:

## Setup - The Boring part:

I won't go into too much details here. Here's what I'm using(no fancy IDE, remember?):
[Arm GNU Toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads) (compilation and tools)
Notepad++ (code editor)
MSYS2 (make and linux like environment)
Some flashing tool at your disposal (command line or STM32CubeProgrammer)

## Understanding the Boot Process

Computers typically start reading from address 0x0, and this ARM processor 
is no different - with a twist. To support different boot modes, the STM32 
can map different memory regions to 0x0. 

For our minimal boot, the standard behavior maps Flash memory (starting at 
0x08000000) to 0x0. Remember this address - you'll see it everywhere!

So when the processor boots, it reads from 0x0 (which is actually 0x08000000 
in Flash), and what needs to be there? Enter the vector table!

## The Vector Table

The vector table is where we store all the relevant configurations for the
booting process. In our case, for a minimal boot, we need only two things:

1. **Initial Stack Pointer**
2. **Reset Handler Address** (pointer to the function where our code starts)

So, first thing, we'll need the Stack Pointer address. The stack lives in RAM
and grows downward, therefore we point at the top of RAM. The STM32F407 has 
128KB of RAM starting at 0x20000000, so the top is at 0x20020000 
(0x20000000 + 0x20000).

Why does the stack pointer come first? ARM's clever design: before running
ANY code, the processor needs a stack. So position 0 isn't code - it's the
stack location. The processor reads it, sets up the stack, THEN jumps to
position 1 to start executing.

Now we're set to start executing code. And you guessed it - main will
be called from inside Reset_Handler.

```c
// startup.c - The absolute minimum to boot

// Function prototypes
void Reset_Handler(void);

// The vector table - MUST be at exactly 0x08000000
__attribute__((section(".vectors")))
const unsigned int vector_table[] = {
    0x20020000,                 // Stack pointer
    (unsigned int)Reset_Handler // Reset handler
};

// This is where we start!
void Reset_Handler(void) {
    // Jump to main
    extern int main(void);
    main();
    
    // If main returns, hang here
    while(1);
}
```

## Main.c
Since we see already main being called from the Reset_Handler 
let me address the elefant in the room, main.c is 
definitely not our main concenr, pun intended.
Its just a simple 'Hey I'm alive' led blinking function to
see that out boot process has worked.

```c
#define RCC_AHB1ENR (*((volatile unsigned int *)0x40023830))
#define GPIOD_MODER (*((volatile unsigned int *)0x40020C00))  
#define GPIOD_ODR   (*((volatile unsigned int *)0x40020C14))

void delay(volatile int count) {
    while(count--);
}

int main(void) {
    // Enable GPIOD clock
    RCC_AHB1ENR |= (1 << 3);
    
    // Set PD12 as output
    GPIOD_MODER |= (1 << 24);
    
    while(1) {
        GPIOD_ODR ^= (1 << 12);
        delay(1000000);
    }
}
```

## From Source to Binary - Compilation and Linking

The build process involves distinct phases:

**Compilation Phase:**
The compiler generates relocatable object files (.o) containing:
- Machine code with symbolic references
- Symbol table (function/variable names and their offsets)
- Relocation entries (locations that need address fixup)
- Section headers (.text, .data, .bss, etc.)

At this stage, all addresses are relative to section start (offset 0). The compiler 
doesn't know the final memory layout - it just generates position-independent code.

**Linking Phase:**
The linker performs several critical tasks:
1. **Symbol Resolution** - Matches undefined symbols with their definitions across 
   all object files
2. **Section Merging** - Combines all .text sections, all .data sections, etc.
3. **Address Assignment** - Assigns absolute addresses based on the linker script
4. **Relocation** - Patches all address references with their final values

Our linker script link.ld (not the one from Zelda), provides the memory layout constraints:
- Memory regions (FLASH at 0x08000000, RAM at 0x20000000)
- Section placement rules (.vectors must be at FLASH origin)
- Output section definitions and their load/runtime addresses

The resulting ELF file contains absolute addresses, ready for execution.

As you can see, the linker is tightly coupled with the target
you are programming. You have to know the memory map in order to
reference the addresses in the linker script.

```c
/* Memory layout for STM32F407 */
MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 1024K
    RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 128K
}

/* Where to put our code */
SECTIONS
{
    /* The vector table MUST go first */
    .vectors : {
        *(.vectors)
    } > FLASH
    
    /* Code goes after vectors */
    .text : {
        *(.text)
        *(.text*)
		*(.rodata)
    } > FLASH
}
```

## Making It All Come Together

Now we just need some minimal Makefile configuration
to make it all come together.

We could do all these steps manually from the 
command line - compilation, linking, extracting the binary, etc. 
But life is hard as it is, so it's better to automate the process, right?
That's what the Makefile does. I won't go into much detail about
the contents of the file itself since it's almost self-explanatory, 
and this post is already getting too long. (Congrats if you've read this far btw!)

But essentially what we're doing is:
- Compiling the C code with our chosen flags
- Assigning addresses using the link.ld file we wrote  
- Generating the final firmware.bin to flash to our microcontroller

## Memory Analysis - Proof It Actually Works

Let's peek under the hood and see what actually ended up in the STM32's memory.
Using the programmer to read address 0x08000000, we see:

0x8000000: 20020000 08000009 AF00B580 F801F000...

Breaking this down:
- `20020000` - Our stack pointer (little-endian for 0x20020000)
- `08000009` - Reset_Handler address (0x08000008 + Thumb bit)
- `AF00B580` - The actual ARM Thumb instructions!

We can verify this with the generated firmware.map or just type:
```bash
arm-none-eabi-objdump -d firmware.elf
```

Which shows our Reset_Handler at exactly 0x08000008, containing push {r7, lr}
(encoded as B580). We're literally seeing our C code as raw bytes in memory!

And that's it, with less than 50 lines of code and zero dependencies, we've built a complete 
boot sequence from scratch. No magic, no hidden complexity - just you, 
the metal, and a blinking LED. Next time, we'll add a task scheduler and 
start building a tiny OS.

Ah, almost forgot, heres the [repo](https://github.com/allexfranc/stm32-startup-from-scratch)