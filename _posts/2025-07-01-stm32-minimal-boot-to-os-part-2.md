---
layout: post
title: "Kernel"
author: "Allex Franc"
date: 2025-07-01
tags: [stm32, embedded, bare-metal, arm, os, scheduler, round robin, kernel]
excerpt: "From minimal boot to Operating System - Part 2: The minimal Kernel: States, Syscalls and Timers"
---

Remember the last post when I said I was going to be more straightforward and not go through every piece of code?
Well, I lied. Here we are, and we'll discuss every part of this minimal kernel I've just built. 


**But first I'd like to make correction: The clock is not configured to 168MHz!**

When I was writing the [previous post](https://allexfranc.github.io/2025/06/22/stm32-minimal-boot-to-os-part-1.html), 
I was working on a later stage of this project, where the clock is, in fact, configured to 168MHz. 
However, for this post, I have not yet configured the clock, and we are using the standard clock value of 16MHz. 
Don't worry, I'll make the clock run at maximum speed in later stages of the project.

# The Kernel
To evolve our previous [project](https://allexfranc.github.io/2025/06/22/stm32-minimal-boot-to-os-part-1.html) 
into a proper kernel, we added three key components: 

1. System calls(syscall.c) - A controlled inferface between tasks and kernel services
2. Timer System(via Systick) - Precision sleep and task scheduling controlled
3. Enhanced Scheduler(scheduler.c) - This enhanced scheduler now deals with task states and timed wake-ups

Together, these transform our simple task switcher into something that starts to resemble an 
actuol OS kernel.

## System Calls
Let's see how tasks are implemented using system calls:

```c
//Before, without system calls:
void task_green(void) {
    uint32_t counter = 0;
    while(1) {
        if(++counter > 500000) {
            GPIOD_ODR ^= (1 << 12);
            counter = 0;
        }
    }
}

//Now, with system calls:
void task_green(void) {	
	while(1){
		sys_led_control(LED_GREEN, LED_ON);
		sys_sleep(500);
        sys_led_control(LED_GREEN, LED_OFF);
		sys_sleep(500);
	}
}
```
We note that tasks do not handle hardware directly anymore and the delays are exact and not approximative.
And this abstraction yields two clear benefits:
>> Precision: sys_sleed(500) throuh a well calibrated systick sleeps for exactly 500ms and not some bazillion iterations.
>> Protection: Tasks cannot corrupt hardware by writing to the wrong registers.

Now, the one bazillion dollars question: How are those system calls handled? Well, buckle up!

### syscall.c/h

Let's examine the syscall.h file:
```c
#define SCB_ICSR 			(*((volatile uint32_t *)0xE000ED04))
#define SYS_YIELD			0
#define SYS_SLEEP			1
#define SYS_GET_TASK_ID		2
#define SYS_LED_CONTROL		3


// System call functions (what tasks will call)
void sys_yield(void);
void sys_sleep(uint32_t ms);
uint8_t sys_get_task_id(void);
void sys_led_control(uint8_t led, uint8_t state);

// SVC Handler
void SVC_Handler(void);
```

This file defines what "functions" the system is providing.

>> SYS_YIELD - Voluntarily gives up CPU time
>> SYS_SLEEP - Sleep for a speficied time amount
>> SYS_GET_TASK_ID - Get current running task's ID
>> SYS_LED_CONTROL - Control LED harware

Now let's take a look at the implementation of those literal functions:

```c
void sys_yield(void){
	__asm volatile ("SVC #0");
}

void sys_sleep(uint32_t ms){
	 __asm volatile (
		"MOV r0, %0    \n"  // Put ms parameter in R0
		"SVC #1        \n"  // Trigger system call #1
		:                   // No outputs
		: "r"(ms)          // Input: ms
		: "r0"             // We're modifying R0
	);
}

uint8_t sys_get_task_id(void) {
    uint32_t result;
    
    __asm volatile (
        "SVC #2        \n"
        "MOV %0, r0    \n"  // Explicitly copy R0 to result
        : "=r"(result)
        :
        : "r0"
    );
    
    return (uint8_t)result;
}

void sys_led_control(uint8_t led, uint8_t state){
	 __asm volatile (
		"MOV r0, %0    \n"  // LED number in R0
		"MOV r1, %1    \n"  // State in R1
		"SVC #3        \n"  // Trigger system call #3
		:                   // No outputs
		: "r"(led), "r"(state)  // Inputs
		: "r0", "r1"           // We're modifying R0, R1
	);
}
```

Note that every single one of these functions triggers the SVC with its defined number in syscall.h, i.e., 
0 for yield, 1 for sleep, 2 get task id and 3 for led control. Now what does SVC do?

### SVC
SVC stands for Supervisor Call. It's an ARM assembly instruction that triggers an immediate exception, 
not a memory mapped register you write to (like how we trigger PendSV via SCB_ICSR).

Why use SVC? It's the CPU's built-in mechanism for user code to request kernel services. 
When a task needs something from the kernel (like sleeping or controlling hardware), 
SVC provides a controlled entry point. 

When SVC gets called from an assembly line of code, the hardware triggers an immediate synchronous exception. 
It checks the vector table at position 11 and jumps to whatever function is defined there, in our case, surprise surprise,
it's the SVC_Handler function, lo and behold:

```c
static uint8_t get_svs_number(uint32_t* stack){
	  // 1. Get the saved PC from the stack frame. 
    //    It holds the address of the instruction AFTER the SVC call.
    uint8_t* pc = (uint8_t*)stack[6];

    // 2. The SVC instruction is 2 bytes long. To find the start of the
    //    instruction, we must look 2 bytes before the saved PC. The first
    //    byte at that location is the immediate value.
	
	return pc[-2];
}

static void trigger_PendSV(void){
	SCB_ICSR |= (1 << 28);  // Trigger PendSV
}

void SVC_Handler(void){	
	// Get current task's stack (was in PSP)
	uint32_t* psp_stack;
	__asm volatile ("MRS %0, psp" : "=r" (psp_stack));
	
	uint8_t svc_num = get_svs_number(psp_stack);
	
	switch(svc_num) {
		case SYS_YIELD:
			// Just trigger a context switch
            trigger_PendSV();  // Trigger PendSV
			break;
		
		case SYS_SLEEP:
			task_sleep_ms(get_current_task(), psp_stack[0]);
			trigger_PendSV();
			break;
		
		case SYS_GET_TASK_ID:
			//This is how we pass the return value without calling return, 
			//ie, we are returning from an exception
			psp_stack[0] = get_current_task(); 
            break;
            
        case SYS_LED_CONTROL:
            // param1 = LED number, param2 = state
            uint32_t led_id = psp_stack[0];
            uint32_t state = psp_stack[1];  // R1 = second parameter
			
			led_set(led_id, state);
            
            break;
	}
}
```

When the execption is triggered the SVC_Handler function is entered, the hardware goes to handler mode, and that means using MSP and not PSP.
But the syscall that called SVC was in thread mode, using PSP. So, in order to retrieve the defined syscall number, it's necessary 
to look up the PSP stack.

More precisely, I'm interested in the PC register at the sixth position of the PSP stack.
When the exception occourred, hardware automatic saved R0-R3, R12, LR, PC and xPSR. At the time of saving, PC was pointing to the instruction 
after the SVC trigger. Let's analise the how the instructions are layed in memory:  


Address    | Value | What
-----------|-------|-----
0x080001F4 | 0x02  | SVC immediate value     <- pc[-2] gets THIS!
0x080001F5 | 0xDF  | SVC opcode
0x080001F6 | 0x08  | Next instruction byte 1 <- PC points here
0x080001F7 | 0x46  | Next instruction byte 2

ARM Thumb-2 encoding works this way: Instructions are 16 bits, Bits: 15-8 = 0xDF (opcode)
Bits: 7-0  = immediate value (#0-255), since memory is addressed as 1 byte spaces and due to the little endianess 
nature of arm instructions, we got immediate value first and op code second. The complete instruction would be 
0xDF02. This helps me remember it: 0xDF most significative, 0x02 least significative. **L**ittle Endian -> **L**east significative 
goes to **L**ower address.

That's exactly what get_svs_number is doing!

Now each case does what it's supposed to do:
- SYS_YIELD: Triggers a context switch via PendSV
- SYS_SLEEP: Calls task_sleep_ms which is implemented within in the improved scheduler (I'll talk about soon) and sets the amount of time this task should sleep, and triggers PendSV for a context switch
- SYS_GET_TASK_ID: Well, it gets tasks id
- SYS_LED_CONTROL: Controls the led with the passed arguments

That was the meaty part of how the syscall works, but before moving on, I'd like to talk a bit more how 
we go from the a syscall function, to its asm implementation and the triggering of SVC_Handler.

Let's talk about sys_get_task_id:

```c
uint8_t sys_get_task_id(void) {
    uint32_t result;
    
    __asm volatile (
        "SVC #2        \n"
        "MOV %0, r0    \n"  // Explicitly copy R0 to result
        : "=r"(result)
        :
        : "r0"
    );
    
    return (uint8_t)result;
}
```

When the instruction SVC #2 is executed, it immediately triggers the exception, and before moving to the next instruction, i.e., 
MOV %0, r0, the case SYS_GET_TASK_ID is executed.

```c
case SYS_GET_TASK_ID:
		//This is how we pass the return value without calling return, 
		//ie, we are returning from an exception
		psp_stack[0] = get_current_task(); 
		break;
```

Here the get_current_task from the scheduler is called and its result is put onto the stack, more precisely at PSP zeroth position.
When the code executes the break instruction and the processor exits handler mode, hardware restores all registers 
(including R0 from PSP[0], which we overrode with psp_stack[0] = get_current_task()). 
The asm line "MOV %0, r0" resumes execution in Thread mode, with R0 now containing the task ID that we stored on the stack.
The rest of the code follows the expected path of moving the received argument from r0 to some register decided by the compiler, 
storing it at the result variable and finally returning its value to the caller.
So, it's a dance from the task that calls the system in C, the syscall triggers the exeption via a special register (SVC) 
using assembly, synchronously switches of modes and stacks, scheduler functions calls, assembly stack manipulation and finally getting the 
result. Is it complicated? Yes, but at the same time, its always the same flow, so, in the end ~~it doesn't even matter~~ 
you realise that this is more direct and a bit simpler that what you've antecipated.

## Improved Scheduler
A kernel needs more than just round-robin task switching. 
Our upgraded scheduler now handles sleeping tasks, wakes them at the right time, and protects against race conditions. Here's what changed:

### Timer and task_t

```c
typedef struct{
	uint32_t* stack_pointer;
	task_state_t state;
	uint32_t wake_tick;
} task_t;
```

```c
static uint32_t system_ticks = 0;
static uint8_t scheduler_countdown = 10;

void SysTick_Handler(void) {
	system_ticks++;
	
   // Search for sleeping tasks and wake them up
    for(int i = 0; i < NUM_TASKS; i++) {
        if (tasks[i].state == TASK_SLEEPING && 
            system_ticks >= tasks[i].wake_tick) {
            tasks[i].state = TASK_READY;
        }
    }
	
	//Triggers context switch every 10 ticks (10ms)
	if(--scheduler_countdown == 0){ //10 to 30 times more efficient than using sytem_ticks%10 == 0
		scheduler_countdown = 10;
		SCB_ICSR = (1 << 28);  // Set PendSV
	}
}

```

That's the main upgrade! SysTick has been considerably upgraded. Now it doesn't just control the scheduler quantum - it's become the kernel's heartbeat.
The timing system now works at two levels:

1ms ticks: System runs at 1kHz for precise timing
10ms scheduling: Context switches still happen every 10 ticks

This dual-frequency approach gives us a few advantages: millisecond sleep precision without the overhead of running the scheduler 1000 times per second.

How sleep/wake actually works:
When a task calls sys_sleep(500):

- The task's wake_tick is set to system_ticks + 500
- Task state changes to TASK_SLEEPING
- Scheduler immediately switches to another task

Every millisecond, SysTick checks all sleeping tasks:
```c
if (tasks[i].state == TASK_SLEEPING && 
    system_ticks >= tasks[i].wake_tick) {
    tasks[i].state = TASK_READY;  // Wake up!
}
```
Sleeping tasks consume zero CPU time, they're completely skipped by the scheduler until their wake time arrives.

So now it's clear how tasks wake up, but how do they go to sleep?
Remember that syscall called:

```c
case SYS_SLEEP:
			task_sleep_ms(get_current_task(), psp_stack[0]);
			trigger_PendSV();
			break;
```
Here's its implementation:

```c
void task_sleep_ms(uint8_t task_num, uint32_t sleep_ms){
	   // Disable interrupts to avoid race conditions!
    __asm volatile ("cpsid i");
	 if (task_num < NUM_TASKS) {
		tasks[task_num].wake_tick = system_ticks + sleep_ms;
        tasks[task_num].state = TASK_SLEEPING;
    }
	   // Enable interrupts
    __asm volatile ("cpsie i");
}
```
The function receives all the needed arguments to put a task to sleep. The task number, in order to find it inside the tasks array, and the 
milliseconds it should sleep. The system_ticks + sleep_ms is stored inside the wake_tick field of that particular task.

Notice that interruptions are being disabled and enabled again:

```asm
    __asm volatile ("cpsid i");
	__asm volatile ("cpsie i");
```

This is to prevend the SysTick firing between setting the wake_tick and the task state.

These two operations must be atomic:
ctasks[task_num].wake_tick = system_ticks + sleep_ms;
tasks[task_num].state = TASK_SLEEPING;
Why? For consistency: If a task has a wake_tick set, it MUST be in the SLEEPING state. These two properties go hand-in-hand.

Without the critical section, an interrupt could see an inconsistent state where wake_tick is set but the task isn't SLEEPING yet. This violates our kernel's invariants.
The critical section ensures this state change is atomic - both happen, or neither happens. It's about maintaining the integrity of our kernel's data structures.

## Other improvements

I'll be more direct here and list a few other improvements without going into too much details:

### Task Size
Well, size matters! Calculating the task_t size manually from inside PendSV_Handler in the assembly part was too error prone. Every time task_t is updated, 
you cannot forget to recalculate the offset from the tasks array base to the current task.
It's easier to use:
**static uint32_t task_size = sizeof(task_t);**
And update the assembly part to use this variable as the offset. Now when we add fields to task_t, the assembly automatically uses the correct size.
The updated assembly parts are:

```c
   // Save current task's SP
        "ldr r1, =current_task    \n"
        "ldrb r2, [r1]            \n"  // Get current task number
        "ldr r3, =tasks           \n"  // Get tasks array base
		
		"ldr r1, =task_size       \n"  // Load task_size
		"ldr r1, [r1]			  \n"
		"mul r2, r2, r1           \n"  // r2 = r2 * task_size
        "str r0, [r3, r2]         \n"  // Save SP - offsetted tasks base array by task_number
```

and

```c
  // Get next task's SP
        "ldr r1, =task_size   	  \n"  // Load task_size
		"ldr r1, [r1]			  \n"
		"mul r2, r2, r1           \n"  // r2 = r2 * task_size
        "ldr r0, [r3, r2]         \n"  // Load new SP - - offsetted tasks base array by task_number
```

### Task initialization
All the tasks must be initialized to the READY_STATE:
```c
void scheduler_init(void) {
    // Initialize each task
    tasks[0].stack_pointer = init_stack(idle_stack, idle_task);
	tasks[0].state = TASK_READY;
	tasks[0].wake_tick = 0;
    tasks[1].stack_pointer = init_stack(green_stack, task_green);
	tasks[1].state = TASK_READY;
	tasks[1].wake_tick = 0;
    tasks[2].stack_pointer = init_stack(blue_stack, task_blue);
	tasks[2].state = TASK_READY;
    tasks[2].wake_tick = 0;
    current_task = 0;
}
```

### Task Switcher Algorithm
This algorithm has been significantly modified, since now it has to account for the case where all the tasks are in the SLEEPING state.
In order to do so, I introduced a counter called **checked**, that counts how many tasks we have checked during the loop, searching for the 
next READY task. If we check all the tasks and none is READY, the idle task is selected. This way an infinite loop is prevented.

```c
uint8_t switch_task(void){
	uint8_t next_task = current_task;
	uint8_t checked = 0;
	
	do{
		next_task = ((next_task + 1)%NUM_TASKS);
		checked++;
		
		if (checked >= NUM_TASKS){
			next_task = 0;
			break;
		}
		
	} while (tasks[next_task].state != TASK_READY);
	
	tasks[next_task].state = TASK_RUNNING;

	if (tasks[current_task].state == TASK_RUNNING) {
        tasks[current_task].state = TASK_READY;
    }
	
	return next_task;
}
```

## Stay tuned!
Thank you for reading and reaching this far!

For the next post I'll bring something lighter, but fun! A LCD driver so we can finally see what our kernel is thinking - task states, system uptime, and maybe even a tiny system monitor!

Happy hacking! See ya!

## Repo
[Here](https://github.com/allexfranc/kernel1)