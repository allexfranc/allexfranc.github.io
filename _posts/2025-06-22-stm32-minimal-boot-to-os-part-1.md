---
layout: post
title: "STM32 Scheduler Part 1"
author: "Allex Franc"
date: 2025-06-22
tags: [stm32, embedded, bare-metal, arm, os, scheduler, round robin]
excerpt: "From minimal boot to Operating System - Part 1: The minimal scheduler"
---

Your STM32 microcontroller has just one CPU core, executing only one 
instruction at a time. So how can it appear to handle multiple 
tasks simultaneously? The answer is a **scheduler** — the beating 
heart of every operating system. In this post we'll build
our very own scheduler from scratch!

This article follows our previous post on creating a minimal boot sequence, 
which is available [here.](https://allexfranc.github.io/2025/06/03/stm32-startup-from-scratch.html)
We'll use that project as a basis for building our Round-Robin scheduler.
Just a heads up: I promise this is the last article I'll go through every single piece of code,
for the next posts I'll be more strightforward and concetrate on the scheduler work.

By the end of this post we should have - wait for it - 2 incredibly shining bright
like diamonds blinking LEDs! - **WOW, grape**!
Yeah, I know, the blinking LEDs are not that exciting, but they perfectly demonstrate
preemptive multitasking: independent tasks executing in time-sliced intervals, with 
hardware-triggered context switches providing the illusion of parallel execution on 
a single-core processor.

# Round-Robin Overview
Round-robin is a preemptive, time-sliced scheduling algorithm where each task receives a fixed time quantum,
these are it's definitive characteristics:

- **Time Quantum**: Fixed execution period (e.g., 10ms via SysTick)
- **Context Switch**: Save current state, restore next state
- **Fairness**: Each task gets equal CPU time regardless of behavior
- **Deterministic**: Task execution order is predictable

The implementation requires:
1. Hardware timer interrupt (SysTick -> PendSV)
2. Context save/restore mechanism (stack manipulation and assembly)
3. Task state storage (TCB - Task Control Block)
4. Schedule selection logic (simple increment with wraparound)

# Implementation Overview
Since we'll implement the world's simplest scheduler, we'll be
manipulating only a few files. These are the ideas behind each implementation:
- **scheduler.c/.h**: Core scheduler implementation
- **startup.c**: Extended vector table
- **main.c**: Task implementation and system initialization
- **link.ld**: Memory layout modifications
- **Makefile**: Build configuration (won't talk about it in this post)

# main.c
Main is very minimal. We start with basic GPIO initialization, 
enabling the GPIOD clock and configuring LED pins as outputs. 
After hardware setup, we initialize the scheduler and start it running.
We also have the implementation of the two tasks that are going to run
in our scheduler - simple infinite loops that toggle LEDs at different rates.
Each task is a simple function that never returns, relying on the scheduler
to share CPU time between them.

```c
void task_green(void) {
    uint32_t counter = 0;
    while(1) {
        if(++counter > 500000) {
            GPIOD_ODR ^= (1 << 12);
            counter = 0;
        }
    }
}

void task_blue(void) {
    uint32_t counter = 0;
    while(1) {
        if(++counter > 1000000) {
            GPIOD_ODR ^= (1 << 15);
            counter = 0;
        }
    }
}

int main(void) {
    // Enable GPIOD
    RCC_AHB1ENR |= (1 << 3);
    
    // Configure LED pins as outputs
    GPIOD_MODER |= (1 << 24) | (1 << 30);  // PD12, PD15
    
    // Initialize scheduler
    scheduler_init();
    
    // Start scheduler
    scheduler_start();
    
    // Never reached
    while(1);
}
```

# startup.c
Startup.c has received a considerable upgrade from our last post. All Cortex-M processors 
share the same system exception layout in positions 0-15 of the vector table.

When an exception occurs, the hardware automatically looks up the corresponding position, 
reads the function address stored there, and jumps to execute that code. 

There's no software lookup or decision-making - it's pure hardware behavior baked into the CPU design.
Understanding this hardware-defined behavior is a powerful thing. 

The CPU doesn't care what you name your functions or where you put them in memory - it only
cares that valid function addresses exist at the exact positions it expects. 

For our minimal scheduler, we've populated two critical entries:
PendSV_Handler and SysTick_Handler, both implemented inside scheduler.c. The architectural
decision to implement them there is simple: these handlers are the scheduler's core mechanisms.
They're not general-purpose exception handlers - they exist solely to make scheduling work, 
so they belong with the scheduler code.

```c
// The vector table - MUST be at exactly 0x08000000
__attribute__((section(".vectors")))
const unsigned int vector_table[] = {
	(unsigned int)&_estack,          // 0: Stack pointer, defined in linker!
    (unsigned int)Reset_Handler,     // 1: Reset
    0,                               // 2: NMI
    (unsigned int)HardFault_Handler, // 3: HardFault
    0,                               // 4: MemManage
    0,                               // 5: BusFault
    0,                               // 6: UsageFault
    0, 0, 0, 0,                      // 7-10: Reserved
    0,                               // 11: SVCall
    0,                               // 12: Debug Monitor
    0,                               // 13: Reserved
    (unsigned int)PendSV_Handler,    // 14: PendSV
    (unsigned int)SysTick_Handler    // 15: SysTick
};
```

The second upgrade addresses a fundamental challenge in embedded systems: initialized variables.
Consider this declaration: uint32_t counter = 100;. This creates two requirements that seem contradictory:

The value 100 must persist through power cycles (needs non-volatile storage)
The variable must be modifiable at runtime (needs RAM)

To solve this we need to update our ResetHandler (the second instruction executed at startup).
 ```c 
 void Reset_Handler(void) {
    // Step 1: Copy initialized data from FLASH to RAM
    extern uint32_t _sdata, _edata, _sidata;
    uint32_t *src = &_sidata;  // Points to initial values in FLASH
    uint32_t *dst = &_sdata;   // Points to where variables live in RAM
    
    while (dst < &_edata) {
        *dst++ = *src++;       // Copy word by word
    }
 ```
- _sidata (maybe 0x08001234) points to where the linker stored initial values in FLASH
- _sdata (maybe 0x20000000) points to where these variables should live in RAM
- _edata (maybe 0x20000100) marks the end of initialized variables

```c
   // Step 2: Zero out uninitialized variables (.bss section)
    extern uint32_t _sbss, _ebss;
    dst = &_sbss;
    
    while (dst < &_ebss) {
        *dst++ = 0;           // Clear word by word
    }
```
C standard requires uninitialized globals start at zero, for example:
- static int flag; must be 0, not random
- uint32_t buffer[1000]; must be all zeros

Without this loop, uninitialized variables would contain whatever random values were in RAM at power-on.
And if you ever wondered why this has to be done at runtime in the Reset_Handler, well,
only code running on the STM32 can write to its RAM. We cannot program the RAM from outside. Besides,
RAM is volatile - every power cycle erases its contents, so during initialization we have to prepare the RAM 
and fulfill the expectations of the C compiled code regarding the memory state.

These files are mostly copy-and-paste setups, but I think it's really neat to know these details of things
we take for granted and understand why we do certain things.

# link.ld
I promisse this is the last time we'll talk about the linker, so bare with me!
In order to understand all the sections that are present in our updated link.ld, lets consider
the following example of a example.c file:
```c
#include <stdint.h>

// This will go into the .data section because it's a global
// variable with an initial non-zero value.
uint32_t initialized_global = 42;

// This will go into the .bss section because it's a global
// variable that is uninitialized (it will be zero-filled at startup).
uint32_t uninitialized_global;

// This will go into the .rodata (Read-Only Data) section
// because it is marked as a constant.
const char my_string[] = "Hello World";

// This function's machine code will go into the .text section.
void my_function(void) {
    initialized_global++;
}
```

Let's compile it with the -c flag, this tells the compiler to compile but
not link!
```bash
arm-none-eabi-gcc -c -mcpu=cortex-m4 -o example.o example.c
```

And let's take a look at the object file with the following command:
```bash
arm-none-eabi-objdump -h example.o
```

result:

example.o:     file format elf32-littlearm

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000001c  00000000  00000000  00000034  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000004  00000000  00000000  00000050  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000004  00000000  00000000  00000054  2**2
                  ALLOC
  3 .rodata       0000000c  00000000  00000000  00000054  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      00000046  00000000  00000000  00000060  2**0
                  CONTENTS, READONLY
  5 .ARM.attributes 0000002e  00000000  00000000  000000a6  2**0
                  CONTENTS, READONLY



> What do we see here? The compiler has read our example.c file and automatically
sorted everything into the stadard sections before the linker even runs.
- **text**: Has a size of 0x1c (28 bytes), which is the machine code for my_function.
- **data**: Has a size of 0x4 (4 bytes), holding the initial value for initialized_global.
- **bss**: Has a size of 0x4 (4 bytes), reserving space for uninitialized_global. Notice it has no CONTENTS flag because there's nothing to store in the file itself.
- **rodata**: Has a size of 0xc (12 bytes), which stores the string "Hello World" plus its null terminator.

It's also relevant to notice the `VMA` and `LMA` columns in the output.

* **VMA** (*Virtual Memory Address*): This is the **run-time address**.
    * It's the address where a section will be located in memory when your program is actually executing on the STM32.
    * When your code reads a variable or calls a function, it uses the VMA.

* **LMA** (*Load Memory Address*): This is the **load-time address**.
    * It's the address where the section is stored in the final executable file that gets programmed into the microcontroller's FLASH memory.

Now we have to match those sections generated by the compiler with our link.ld file implementation:
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
		*(.rodata*)
    } > FLASH
	
	/* Mark where .data initial values start in flash */
    _sidata = LOADADDR(.data);
	
	.data : {
        _sdata = .;
        *(.data)
        *(.data*)
        _edata = .;
    } > RAM AT > FLASH
    
    .bss : {
        _sbss = .;
        *(.bss)
        *(.bss*)
        *(COMMON)
        _ebss = .;
    } > RAM
    
    /* Stack at top of RAM */
    _estack = 0x20020000;
}
```

And that's exactly what the link.ld is doing. I'll like to end this discussion with some comments about the
.data and .bss sections. That's the link (pun intended) with the code we added at our Reset_Handler!

- _sidata: Where initial values are stored in FLASH (source for copying, initialized_global in our example.c)
- _sdata: Where variables should live in RAM (destination)
- _edata: End of initialized variables in RAM (stop copying here)

The >RAM AT > FLASH syntax tells the linker to give these variables RAM addesses for execution,
but store their initial values in FLASH

- _sbss: Start zeroing here
- _ebss: Stop zeroing here

Without the linker script the Reset_Handler would not know where to find initial values in flash,
where to put them in ram, how much to copy and what to zero. Phew, that's about it for the linker!

# scheduler.c/.h
Yep, we finally got to talk about the scheduler! Glad you are still here s2!
Let's talk about how our Cortex-M4 handles context switching and which resources 
are made availble to us.

## The Two Stacks Model: MSP vs PSP
Arm Cortex-M has two execution modes:
- Handler Mode: Running exeption/interrupt handlers always uses MSP (Main Stack Pointer)
- Thread Mode: Normal code execution uses MSP by defaults, can switch to PSP (Process Stack Pointer)

At reset, the processor starts in Thread Mode using MSP. The initial stack pointer 
value comes from vector[0] - this sets up MSP pointing to the top of RAM (0x20020000 
in our case). All code, including Reset_Handler and early main(), runs using this 
main stack.

The power of dual stacks becomes clear when we configure Thread Mode to use PSP. 
Each task gets its own stack region carved from RAM - we define arrays of fixed 
size (like uint32_t green_stack[128]), and the linker places them somewhere in RAM, 
well separated from the MSP region starting at 0x20020000 (Remember this value?). 

This creates a natural protection boundary: tasks can overflow their PSP stacks 
without corrupting the kernel's MSP stack. Meanwhile, MSP remains reserved exclusively 
for exception handlers, ensuring interrupts always have a clean, predictable stack 
to work with.

Memory Layout Example:
0x20020000: [MSP starts here, grows down]
0x2001F000: [Task 1 PSP region]  
0x2001E000: [Task 2 PSP region]
0x2001D000: [Task 3 PSP region]
... rest of RAM for variables, heap, etc.

With this model, if a task has a bug, like deep recursion, large arrays, etc, it 
overflows PSP and crashes only that task, leaving the kernel MSP safe. (Notice the 
memory placement and that stack grows downward).
This lays a fundation for **Memory Protection** which will talk about in another post, 
stay tuned ;)!

## Systick: The Heart Beat
SysTick provides the rhythm for our scheduler's time slices. With our board running 
at 168MHz, we configure:
```c
    SYST_RVR = 1680000 - 1;  // 10ms at 168MHz

```
This generates interrupts every 10 ms which are handled by the SysTick_Handler defined 
at the interrupt table.
But here's the clever part: the Systick does not perform context switch itself. Instead it triggers 
a very special exception, PendSV:
```c
void SysTick_Handler(void) {
    SCB_ICSR = (1 << 28);  // Set PendSV pending
}
```
Why this indirection? PendSV runs at the lowest priority, ensuring we only switch
contexts when it's safe - after all higher priority interrupts have finished.

## PendSV: The Switching Engine
Well, here's where the magic happens.
PendSV is where the context switch happens. When PendSV triggers, the hardware automatically
saves 8 registers onto the current task's stack, i.e., where PSP is pointing to.
Saved registers: R0, R1, R2, R4, R12, LR, PC, xPSR.
This automatic stacking is crucial - the hardware saves exactly what it needs to 
resume the task later. But notice what's NOT saved: R4-R11. That's our job!

## Into the Code
Now that the concepts are clear, let's dive into the code!

### Variables
```c
typedef struct {
    uint32_t* stack_pointer;
} tcb_t;

#define STACK_SIZE 128
#define NUM_TASKS 3

// Separate stack for each task
static uint32_t idle_stack[STACK_SIZE];
static uint32_t green_stack[STACK_SIZE];
static uint32_t blue_stack[STACK_SIZE];
// Task control blocks
static tcb_t tasks[NUM_TASKS];
static uint8_t current_task = 0;
```
- One critical piece of state: Each task only needs its stack pointer saved. Why? Because everything else
lives on that stack, i.e., registers, return address, local variables.
- Fixed arrays: No dynamic allocation needed. The linker places these stacks somewhere in RAM.
- Stack size: 512 (128 times 4) bytes per task, plenty of space for our simple tasks.
- Current task: Scheduler's only runtime state - which task is running now.

This Scheduler is absolutely minimal: no priority, no state, no timers. Those will be implemented for the next posts!

### Stack Init
```c
// Initialize a task stack
static uint32_t* init_stack(uint32_t* stack_top, void (*handler)(void)) {
    uint32_t* sp = &stack_top[STACK_SIZE - 1];
    
    // ARM Cortex-M stack frame (hardware pushed)
    *(sp) = 0x01000000;             // xPSR (thumb bit set)
    *(--sp) = (uint32_t)handler;    // PC
    *(--sp) = 0xFFFFFFFD;           // LR (return to thread mode, use PSP)
    *(--sp) = 0;                    // R12
    *(--sp) = 0;                    // R3
    *(--sp) = 0;                    // R2
    *(--sp) = 0;                    // R1
    *(--sp) = 0;                    // R0
    
    // Software saved registers
    *(--sp) = 0;                    // R11
    *(--sp) = 0;                    // R10
    *(--sp) = 0;                    // R9
    *(--sp) = 0;                    // R8
    *(--sp) = 0;                    // R7
    *(--sp) = 0;                    // R6
    *(--sp) = 0;                    // R5
    *(--sp) = 0;                    // R4
    
    return sp;
}
```

WWhen initializing a task's stack we face an interesting problem: the task hasn't run yet.
There's no context to save. But PendSV_Handler expects to find a specific stack frame, which 
is why we pre-populate the stack with initial values in this exact order, simulating what a 
context switch would create.

When PendSV_Handler first "restores" this task, it will:
1. Pop R4-R11 (finding our zeros)
2. Return to exception, hardware pops R0-PC-xPSR
3. PC points to task function - task starts running!

This clever trick makes task startup identical to task resumption - PendSV doesn't know or care that this is the first time.

Now let's decode 3 critical values:

**xPSR = 0x01000000**
The Program Status Register needs the Thumb bit (bit 24) set. ARM Cortex-M can only execute 
Thumb instructions, so this bit MUST be 1 or the processor faults. The other bits (flags 
for zero, carry, etc.) start at 0.

**PC = (uint32_t)handler**
This is where the magic happens. When the hardware "returns" from PendSV, it loads this 
value into the Program Counter. The CPU starts executing at this address - this is how 
your task function actually starts running!

**LR = 0xFFFFFFFD**

This is a special "EXC_RETURN" value. When the processor sees 0xFFFFFFFX in LR during 
an exception return, it doesn't treat it as a real address. Instead, each bit has meaning:

The last hex digit (D in our case) tells the processor HOW to return:
- 0xFFFFFFFD = Return to Thread mode, use PSP
- 0xFFFFFFF9 = Return to Thread mode, use MSP  
- 0xFFFFFFE9 = Return to Handler mode, use MSP

Breaking down 0xFFFFFFFD:
- Bits 31-4: All 1s (0xFFFFFFF) = "This is an exception return"
- Bit 3: 1 = Return to Thread mode (not Handler mode)
- Bit 2: 0 = Use PSP (not MSP)
- Bit 1: Always 0
- Bit 0: 1 = Basic stack frame (no floating point)

So when PendSV does `bx lr` with 0xFFFFFFFD, the hardware:
1. Recognizes this special value
2. Performs exception return (pops registers)
3. Returns to Thread mode using PSP

When PendSV executes `bx lr` with this value, the hardware knows to do an exception 
return (popping all those registers) rather than a normal branch.

### Scheduler Init
```c
void scheduler_init(void) {
    // Initialize each task
    tasks[0].stack_pointer = init_stack(idle_stack, idle_task);
    tasks[1].stack_pointer = init_stack(green_stack, task_green);
    tasks[2].stack_pointer = init_stack(blue_stack, task_blue);
    
    current_task = 0;
}
```

At scheduler_init each of the 3 tasks has its stack initialized and added to the tasks array.
The init_stack function returns the stack pointer positioned at the "top" of the prepared stack 
(remember, after pushing all those initial values).

Current task is explicitly set to 0(idle_task), which determines which task runs first when the
scheduler starts. This choice is arbitrary, but idle is a safe default.

### Scheduler Start
```c
void scheduler_start(void) {
    // Set initial PSP
    __asm volatile ("MSR psp, %0" : : "r" (tasks[0].stack_pointer));
    
    // Use PSP for thread mode
    uint32_t control = 2;
    __asm volatile ("MSR control, %0" : : "r" (control));
    __asm volatile ("ISB");
    
    // Configure SysTick for 10ms
    SYST_RVR = 1680000 - 1;  // 10ms at 168MHz
    SYST_CVR = 0;
    SYST_CSR = 7;  // Enable, interrupt, processor clock
    
    // Enable interrupts
    __asm volatile ("cpsie i");
    
    // Should never return
    while(1);
}
```
At scheduler_start is where the heart starts beating!
First we set PSP to point to tasks[0] stack (we have just defined it: idle_task).
PSP can only be used in Thread mode (Handler mode always uses MSP), 
and even then it must be explicitly enabled via the CONTROL register:
```c
	uint32_t control = 2;
    __asm volatile ("MSR control, %0" : : "r" (control));
	__asm volatile ("ISB");

```
The instructions above means:
(MRS)Move to special register CONTROL the value 2.

The CONTROL register bits:

Bit 0: 0=privileged, 1=unprivileged (keep at 0) 
Bit 1: 0=use MSP, 1=use PSP in Thread mode (set to 1)
Bit 2: (floating point related, ignore)

ISB = Instruction Synchronization Barrier.
Due to ARM's pipelined architecture, subsequent instructions may already be in the pipeline 
when we change the CONTROL register. To ensure all following instructions see the new 
Thread mode configuration (using PSP instead of MSP), we must flush the pipeline and 
force fresh instruction fetches. That's what the ISB instruction does.

Now, Systick must be configured to provide the scheduler's heart beat:
```c
	SYST_RVR = 16800000 - 1;  // 10ms at 168MHz
    SYST_CVR = 0;
    SYST_CSR = 7;  // Enable, interrupt, processor clock
```
- RVR (Reload Value Register): When Systick counts to zero, this is the value reloaded to CVR
- CVR (Current Value Register): The current amount - is set to zero to trigger immediate reload.
- CSR (Control and Status Register): Configuration and control, set to 111

For CSR:
- Bit 0: Enable counter
- Bit 1: Enable interrupt when reaching 0
- Bit 2: Use processor clock (168MHz) not external reference

Setting CVR won't trigger an interrupt - the write simply resets the counter. This is done 
to trigger an immediate reload from RVR, ensuring we start from a known state (rather than 
whatever random value CVR contained at power-up). Actual interrupts will only fire after 
global interrupts are enabled:
```c
// Enable interrupts
    __asm volatile ("cpsie i");
```

The infinite loop might seem odd, but it's important.
Once scheduler_start() finishes the Systick configuration and interrupts are enable,
Systick will fire every 10ms and force the first context switch via PendSV. And from
that moment on, the tasks are running! 

The while(1) is a safety net - if something goes catastrophically wrong and we somehow
return here, at least we won't execute random memory. In a healthy system, this line
never executes.

```c
    // Should never return
    while(1);
```

### PendSV_Handler
Now, the star of our show! The PendSV_Handler is where contexts actually switch. This is 
pure assembly wrapped in a "naked" function - no compiler-generated prologue or epilogue, 
just our code:
```c
__attribute__((naked))
void PendSV_Handler(void) {
```
The naked attribute is crucial! It's there to avoid any compiler optimizations.
We are managing register and stacks ourselves here, so any optimizations 
could corrupt our values.

### Switching Context
The PendSV_Handler perform three critical steps:
1. Save current task's context;
2. Select next task (Round Robin);
3. Restore next task's context.

### Saving Current Context
```c
		// Save current context
        "mrs r0, psp              \n"
        "isb                      \n"
        
        // Save r4-r11
        "stmdb r0!, {r4-r11}      \n"
```
- MRS(Move from Special Register): This copies the value of PSP into R0. Now R0 points to the curent task's stack.
Remember that hardware has already pushed R0-R3, R12, LR, PC and xPSR.
- ISB(Instruction Synchronization Barrier): Same reason as before. Ensures PSP read has already finished before we use it.
- STMDB (Store Multiple, Decrement Before): As SP points to the top item of the stack and 
stacks grow downward, the order is: decrement first, then store. Despite the syntax showing 
{r4-r11}, ARM stores higher-numbered registers at higher addresses, so R11 is stored first 
(at the highest address) down to R4 (at the lowest). The exclamation point means "writeback" - 
update R0 to point to the new stack top after the operation.

Now this task's context is fully saved and R0 points to the top of its stack.

```c
		// Save current task's SP
        "ldr r1, =current_task    \n"
        "ldrb r2, [r1]            \n"  // Get current task number
        "ldr r3, =tasks           \n"  // Get tasks array base
        "lsl r2, r2, #2           \n"  // Multiply by 4 (pointer size)
        "str r0, [r3, r2]         \n"  // Save SP
```

Despite being assembly, this part is almost self-explanatory, with the exception of one small trick:
- Load the address of current_task into R1
- Load the VALUE at that address into R2 (will be 0, 1, or 2)
- Load the base address of the tasks array into R3
- Here's the trick: to find tasks[current_task], we need to offset by (current_task * sizeof(tcb_t))
- Since each tcb_t is 4 bytes, we shift left by 2 (multiply by 4)
- Store R0 (our saved stack pointer) at tasks[current_task].stack_pointer (str r0, [r3, r2]
means Take the value IN R0 (the saved stack pointer) and Store it AT the address (R3 + R2))

### Getting the next task
```c
 // Select next task (round robin)
        "ldrb r2, [r1]            \n"  // Get current task again
        "add r2, r2, #1           \n"  // Next task
        "cmp r2, #3               \n"  // We have 3 tasks
        "bne no_wrap              \n"
        "mov r2, #0               \n"  // Wrap to task 0
        "no_wrap:                 \n"
        "strb r2, [r1]            \n"  // Update current_task
        
        // Get next task's SP
        "lsl r2, r2, #2           \n"  // Multiply by 4
        "ldr r0, [r3, r2]         \n"  // Load new SP
```

- Load the current task number again
- Add 1 to the the next task
- Compare with 3 (our task count)
- If R2 != 3, jump to no_wrap (skip the reset)
- If R2 == 3, set R2 to 0 (wrap around)
- Store updated task number back to current_task variable
- Multiply new task number by 4 for offset
- Load new task's stack pointer into R0

### Restore Context
```c
   // Restore r4-r11
	"ldmia r0!, {r4-r11}      \n"
        
	// Set PSP to new task
	"msr psp, r0              \n"
	"isb                      \n"
```

- R0 points to the next task's saved stack top
- At the top of the stack are R4-R11 from when we previously saved this task
- LDMIA (Load Multiple, Increment After): Load R4 from [R0], increment R0, load R5, etc.
- After loading all 8 registers, R0 points to where hardware-saved registers begin
- MSR copies R0 to PSP - now PSP points to the hardware frame
- ISB ensures PSP change takes effect immediately

Notice the simmetry:
When saving hardware pushed 8 registers, we pushed 8 more with STMDB (store multiple decrement before)
Now, we pop 8 register with LDMIA(load multiple increment after), 
harware will pop its 8 on exception return.

### Return

```c
        // Return
        "orr lr, lr, #0x04        \n"  // Ensure return to thread mode
        "bx lr                    \n"
```

This is the return sequence.

When PendSV_Handler was entered, hardware put an EXC_RETURN value in
LR (like 0xFFFFFFF9 or 0xFFFFFFFD). The ORR unsures bit 2 is set, which means
return to Thread mode using PSP (that's what setting bit 2 means, 0 means use MSP).
This is defensive programming - we FORCE the correct return mode 
regardless of how PendSV was entered.

```c
"bx lr"               // Branch to address in LR
```

LR does not contain a real memory address, the processor recognizes this 
special pattern and perform an exception return, as we described before:
- Restore R0-R3, R12, LR, PC, xPSR from stack (pointed to by PSP)
- Resume execution at the restored PC
- The new task is now running!

BTW when any execption fires, like Systick and PendSV, hardware 
saves 8 registers to current stack, loads PC with handler address 
from vector table and loads LR with EXC_RETURN value, like:
- 0xFFFFFFF1: Return to Handler mode, MSP
- 0xFFFFFFF9: Return to Thread mode, MSP
- 0xFFFFFFFD: Return to Thread mode, PSP 

**The Context Switch is now complete!**

## Stay tuned!
Thank you for reading and reaching this far!

In the next posts we'll transform this basic scheduler into something more powerful:
- Task states (READY, RUNNING, SLEEPING) - tasks that actually sleep!
- System calls - tasks requesting services from the kernel
- Hardware interrupts controlling task behavior

We're building toward a real operating system, one concept at a time. The scheduler you've 
just seen is the beating heart - everything else builds on this foundation.

Happy hacking!

## Repo
[Here](https://github.com/allexfranc/scheduler1)