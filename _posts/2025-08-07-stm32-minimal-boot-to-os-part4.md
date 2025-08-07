---
layout: post
title: "STM32 System Monitor"
author: "Allex Franc"
date: 2025-08-07
tags: [stm32, embedded, bare-metal, arm, os, scheduler, round robin, kernel, lcd, debug, memory]
excerpt: "From minimal boot to Operating System - Part 4: Dynamic Memory"
---

Well, well, well... this is a big one! Memory! That's the point where some of us miss the JVM... not!

This is one of the most important concepts to master. When you implement your custom malloc and free, concepts like 
the heap and the stack become crystal clear. It's also necessary to understand a bit about your hardware - where the code goes 
and what physical constraints you're dealing with. But don't worry, I'm gonna tackle all of these subjects in detail 
and we are going to build a very nice memory allocator.

# The Hardware
Understanding the hardware is a prerequisite to define addresses and to understand your memory range. 
Let's remember some hardware facts about the STM32F407:

**Flash Memory:**
- Size: 1MB (1024KB)
- Address range: 0x08000000 to 0x080FFFFF
- Where code and const data live

**RAM**
- Size: 128KB
- Address range: 0x20000000 to 0x2001FFFF
- Where variables, stack, and heap live

These addresses should ring some bells... A mini recap of some places where I've already used this before:
1. Remember the first post where I defined the startup.c file? The first thing the chip reads is the 0x08000000 address. Thats 
where we tell the linker to put the vector table! And what's the first thing at the vector table? The address of the stack pointer, 0x20020000 
which is the end of the RAM section. Remember that the stack grows backwards. This is hardware design: read the first address of the flash, 
interprets the value there as the address of the MSP (Main Stack Pointer).
2. The second address at the flash, i.e., the second vector at our table is the Reset_Handler - where the code the programmer writes (us) starts executing. 
That's tipically where main.c is called, but before calling main, the Reset_Handler has a critical job: setting up the environment. 
It copies initialized variables from flash to RAM (because flash is read-only), and zeros out the .bss section 
(because C says uninitialized globals must be zero). Without this setup, your `uint32_t counter = 100;` would be stuck 
at whatever random value was in RAM at power-on.
```c
// Copy .data from flash to RAM
    extern uint32_t _sdata, _edata, _sidata;
    uint32_t *src = &_sidata;  // Source in flash
    uint32_t *dst = &_sdata;   // Destination in RAM
    while (dst < &_edata) {
        *dst++ = *src++;
    }
    
    // Zero .bss
    extern uint32_t _sbss, _ebss;
    dst = &_sbss;
    while (dst < &_ebss) {
        *dst++ = 0;
    }
```
_sdata, _edata, _sidata are not pointers, they are symbols the linker creates with addresses. 
After the linker places the vector table, the code itself (.text) and read-only data (.rodata) in flash, 
it calculates where to store the initial values for .data variables. That location becomes _sidata.
The linker also decides where in RAM these variables will actually live during execution, that's _sdata (start) and _edata (end). 
Here's the trick: we declare them as `extern uint32_t` just to give them a type, but we never use their "value". We only care about their addresses.
This related to the .data section of the linker. The next section, .bss is about the non initialized data, or more precisely, 
variables that should start at zero. That includes:
- Global variables without initial values: `uint32_t counter;`
- Static variables: `static uint8_t flag;`  
- Large arrays: `uint32_t buffer[1000];`
This is a pattern to save flash memory. Consider `uint32_t buffer[1000]` that's 4000 bytes of zeros that would need to live in flash. 
That's wasteful. So the linker tracks that 4000 bytes are needed in RAM (and also the other zeroed variables) and calculates 
where the .bss section lives in RAM, creating the symbols _sbss (start) and _ebss (end).
No zeros are stored in flash, just the _sbss and _ebss addresses. The programmer then initializes them to 0 at run time inside the Reset_Handler.  

After this not so quick recap, let's dive into the memory layout!

## Memory Layout

Just out of curiosity, I wanted to measure how much memory the current system actually uses:
- **_ebss**: End of .bss section in RAM  
- **MSP Water Mark**: The lowest point MSP reached (maximum stack usage)

A simple task could monitor MSP usage like this:

```c
void task_stack_monitor(void) {
    uint32_t watermark = 0x20020000;  // Start at top
    
    while(1) {
        uint32_t current_msp;
        __asm volatile ("MRS %0, msp" : "=r" (current_msp));
        
        if (current_msp < watermark) {
            watermark = current_msp;  // New low point!
        }
        
        // Display it
        lcd_write_string("MSP: ");
        lcd_write_hex(watermark);
        
        sys_sleep(10);  // Check every 10ms
    }
}
```

Anyway, we still have to come up with some kind of layout for the memory implementation. Given that the mc has 128kb of RAM, 
I decided to have the following:

├───────────────────────────────────────────────────────────────────┤
 checked _ebss address: 			0x20000880, 2kb used
 checked msp water mark address: 	0x2001FFC8, 56b used
 ///////////////////////////////////////////////////////
 Memory Design
 max stack for msp		2k ----- 0x20020000 - 0x0800 = 0x2001F800
 _bss					20k ---- 0x20000000 + 0x5000 = 0x20005000
 heap					106k --- 0x20005000 to 0x2001F800
 
 took me half an hour to draw this with unicode chars:
 
 RAM Layout:
0x20000000 ├──────────────┤
           │  .bss/data   │ 20KB (0x5000)
           │              │
           │  18KB        │
0x20005000 ├──────────────┤ 
           │              │
           │    HEAP      │ 106KB (0x1A800)
           │              │
0x2001F800 ├──────────────┤
           │  MSP Stack   │ 2KB (0x800)
0x20020000 └──────────────┘

├───────────────────────────────────────────────────────────────────┤

Notice how the stack starts at 0x20020000 and grows downward toward 0x2001F800, while the heap starts at 0x20005000 
and grows upward toward 0x2001F800. They grow toward each other but are separated by that boundary at 0x2001F800.
This is the classic design, they grow towards each other. If either one grows too much, corruption may occour, but having these well defined boundaries 
will help us to implement memory security features in the future. 

## The Stack vs The Heap
This is a classic discussion. We talk a lot about it, but sometimes these concepts are not crystal clear, specially when you come from 
higher level languages.
I just showed you where they live in memory, now let's examine their behavior and use cases.
Understanding this is crucial because our tasks share the heap but each gets its own stack, 
and mixing them up leads to those crashes we all love to debug.

### The Stack
The Stack is automatic memory management. You get a pointer (SP) that points to some address in memory, 
and it grows/shrinks as your program runs.

Function parameters, return addresses (where to go after a function ends), local variables, they all go on the stack.
When you write some C code, the compiler generates the push/pop instructions, it's always LIFO.

In AllexOS, each trask has its own stack (using PSP), while interrupts and main use a separate kernel stack (MSP). Remember 
the initilization for each stack in scheduler.c? 

What I'm trying to say is that the stack is basically bookkeeping for your program as it runs. The compiler is 
generating the necessary pushes and pops. But sometimes YOU need to interveine. I'll give you some examples:

- Context switching: Compiler can't know about task switches), we manually save/restore R4-R11
- System calls: You are in exception mode using MSP, but need to read paramters fromthe task's PSP
- Exception handlers: Different stack frames for different modes
- Bootstrapping: Tasks creating initial context from nothing

Add to that mix the hardware part: it automatically pushes 8 registers to the current stack when entering an exception.

Bottom line is: the stack does a lot of things and it's not easy to master. But I think we got a clear vision of 
"what it is".

### The Heap
I just had a hard time trying to put what the stack is/does into words. Let's see if I have an easier time with the Heap, lol.

The Heap is the memory region where the programmer can allocate memory. You have to explicitly call for memory, 
with functions like malloc and you have to free this memory with free. 
The programmer manages the allocation and deallocation, it's not automatic like the stack. 
Since this is a space in memory, it's not constrained to a particular task. It's just a chunk of memory 
that has an address (a pointer) and some value in it. You decide what to do with it. 
That's where you find a big difference from the likes of Java, where the JVM does the memory management for you.

In AllexOS, the heap is those 106KB sitting between 0x20005000 and 0x2001F800. 
When a task calls sys_malloc(100), our allocator finds 100 free bytes in that region and returns a pointer. 
When they call sys_free(), we mark it as available again.

The key difference from the stack: the heap has no structure. It's not LIFO, allocations can happen in any order, 
and freed memory can create holes (fragmentation). That's why we need an allocator: someone has to track what's free and what's used.

# Implementation

## Overview
The implementation has 3 main parts - initialization, allocation and freeing.
I choose to implement a implicit free list design, first fit allocation and forward coalescing. If you are not familiar with these concepts, 
follow along and I'll explain them while showing the implementation.
This is sufficient for a functional memory allocator and leaves room for future improvements like backward coalescing.

## memory_init

This is our memory block structure:

```c

#define HEAP_START  0x20005000
#define HEAP_END    0x2001F800
#define MSP_SIZE    0x800
#define BSS_BUDGET  0x5000

typedef struct block{
	size_t size;
	int free;
	struct block *next;
} block_t;
```

and this is how our memory is initialied:

```c
block_t* block_list  = NULL; //Implicit free list design

void memory_init(void){
	block_t *block = (block_t*)HEAP_START;
	block->size = HEAP_END - HEAP_START;
	block->free = 1;
	block->next = NULL;
	
	block_list  = block;
}
```

The idea is to start with one giant free block covering the entire heap. 
Every allocation will carve pieces from this block_list.

The "implicit free list" means free and allocated blocks are mixed in a single linked list, 
we check the 'free' flag to know which is which.

## memory_malloc

```c
void *memory_malloc(size_t size){
	
	block_t *current = block_list ;
	block_t *previous = NULL;
	
	while(current != NULL){
		if(current->free == 1 && current->size >= size + sizeof(block_t)){
			//Found it!
			break;
		}
		previous = current;
		current = current->next;
	}
	
	if (current==NULL){
		return NULL;
	}
	
	if(current->size - size - sizeof(block_t) < sizeof(block_t) + 16){    
		current->free = 0;
		return (void*)(current + 1);
	} //Don't split it, return whole since it's small!
	
	
	block_t* new_free = (block_t*)((char*)current + sizeof(block_t) + size);
	new_free->size = current->size - size - sizeof(block_t);
	new_free->free = 1;
	new_free->next = current->next;
	
	current->next = new_free;
	
	current->free = 0;
	current->size = size + sizeof(block_t);
	
	return (void*)(current+1);
}
```

Before explaining the code, let me clarify how memory allocation works. When you allocate memory, 
it is necessary to store metadata too.
Each allocation consists of:
1. The block header (the block_t struct)
2. The actual memory space requested

So when you malloc(100) is called, we actually allocate sizeof(block_t) + 100 bytes. 
But we return a pointer to the user space, not the header. 
That's why in malloc we do `return (void*)(current + 1)`, it skips past the header (pointer arithmetic).

Now, let's take a closer look the code!

### Searching
```c
	block_t *current = block_list ;
	block_t *previous = NULL;
	
	while(current != NULL){
		if(current->free == 1 && current->size >= size + sizeof(block_t)){
			//Found it!
			break;
		}
		previous = current;
		current = current->next;
	}
	
	if (current==NULL){
		return NULL;
	}
```

Here we are searching for a suitable chunk of memory.
We start from the block_list, the one that was assigned the whole heap at init.
The initial check is if it's not NULL, than we proceed to check if theres the memory chunk is free and the size is suitable.
If so, we found the piece of memory we where looking for and it's pointed by current. If not, we simply return null, i.e., no memory for you!
This strategy is called first fit allocation. You always start at the same point and traverse the linked list until you find a 
suitable piece of memory (or null). It's dead simple and works fine. Noted that there's room for improvement in the future.

### Spliting

```c
if(current->size - size - sizeof(block_t) < sizeof(block_t) + 16){    
		current->free = 0;
		return (void*)(current + 1);
	} //Don't split it, return whole since it's small!
	
	
	block_t* new_free = (block_t*)((char*)current + sizeof(block_t) + size);
	new_free->size = current->size - size - sizeof(block_t);
	new_free->free = 1;
	new_free->next = current->next;
	
	current->next = new_free;
	
	current->free = 0;
	current->size = size + sizeof(block_t);
	
	return (void*)(current+1);
```
When we find a suitable block, we have two choices:
1. Give the whole block to the user (makes sense for a small block but wasteful if the user asked for 10 bytes but the block is 1000)
2. Split it: give them what they asked for and create a new free block with the leftover

Here's the decision:

```c
if(current->size - size - sizeof(block_t) < sizeof(block_t) + 16){    
    current->free = 0;
    return (void*)(current + 1);
}
```

If the leftover would be less than sizeof(block_t) + 16 bytes, don't bother splitting. Why 16 you might ask... 
It's the minimum useful allocation, smaller than that and we're just creating fragmentation.

Otherwise, we split:

```c
1	block_t* new_free = (block_t*)((char*)current + sizeof(block_t) + size);
2	new_free->size = current->size - size - sizeof(block_t);
3	new_free->free = 1;
4	new_free->next = current->next;
	
5	current->next = new_free;
	
6	current->free = 0;
7	current->size = size + sizeof(block_t);
	
8	return (void*)(current+1);
```
Let's go line by line:
1 - Here I'm marking the start address of the new free block. current points at the start of the allocated block, so we have 
to account for the header and the allocated size. That's exactly what the first line is doing. Just a clarification on the syntax: 
the cast ```(char*)``` before current is necessary to do an arithimetic sum, suppose current value is 0x20005000, block_t has a size of 12 and size is 100, 
the final result would be something like 0x20005070 (112 bytes in decimal = 0x70 in hex).
2 - Remember that the free block that we find is always the whole free memory left. And that's why we are spliting. With this logic 
in mind this line becomes obvious: the new_free size is just the current block size (the whole remaining memory) minus the allocated 
size and the header size.
3 - The new free block is free, duh!
4 && 5- We are carving current from the free memory block, so current->next points to the new_free and new_free->next points to 
whatever current->next was pointing to. We have to pay attention to the order of these operations. 
6 - Current is now allocated, so we mark it as not free!
7 - Current size is whatever the allocated space was plus the metadata, i.e., it's header.
8 - Return the pointer to the actual free space of current, i.e., the space after the header. That's why we need the pointer 
arithmetic. ```current+1``` means current plus one time whatever the size of current is. We know current is a block_t. 
So that would be equivalent of writing ```(void*)((char*)current+ sizeof(block_t));```.

## memory_free

```c
void memory_free(void *ptr){
	if(!ptr) return;
	
	// Receives a pointer to the memory space, need to retrieve the header
	block_t* block = ((block_t*)ptr) - 1;
	block->free = 1;
	
	//Forward Coalescing	
	if(block->next && block->next->free == 1){
		block->size += block->next->size;
		block->next = block->next->next;
	}
}
```

Freeing is much more straightforward. Remember how malloc returned a pointer AFTER the header? 
Well, free receives that same pointer, so we need to back up to find the header:

```c
block_t* block = ((block_t*)ptr) - 1;
``` 
This does the reverse of malloc's return ```(void*)(current + 1)```. 
We subtract 1 (which moves back by sizeof(block_t)) to get to the header.

Now, forward coalescing is a simple process: you check if the next block from the one you just freed is also free. If so, you unify both 
blocks. To do this, is sufficient to unify the sizes and to make the current block->next points to block->next->next. 
This prevents fragmentation. Without coalescing, you'd end up with many small free blocks that can't satisfy larger allocations, 
even though the total free space is sufficient.

# Usage

Having malloc and free opens up a lot of possibilities! We can now add tasks dynamically to our 
scheduler - no more fixed number of tasks at compile time!

We can even have tasks that create other tasks. Of course the scheduler needed some refactoring, 
but it applies the same concepts as before. The main change is that task structures now live on the 
heap instead of in the .bss/data section.

Actually I think you should take a good look on the new scheduler implementation... I was being a bit optimistic, it has changed 
a lot to accomodate the dynamic task creation. But this post is about memory, and not about the scheduler. If you have any trouble 
understanding the new scheduler, write me! Besides, there will be other posts about the scheduler when I add priorities to the tasks, 
so there's room to talk about it again in the future.

## Memory Statistics
I also implemented some monitoring functions to track heap health:
```c
typedef struct {
	uint32_t total_size;
    uint32_t used_size;
    uint32_t free_size;
    uint32_t largest_free_block;
    uint32_t num_free_blocks;
    uint32_t num_used_blocks;
	uint32_t fragmentation;
} heap_stats_t;
```
You can see this in action in task_system_monitor, it shows fragmentation percentage, largest free block, etc. 
Super fun to see the system in action!

![LCD showing memory stats]({{ "/images/lcd-memory-monitor.jpeg" | relative_url }})

## Stay tuned!
Thank you for reading and reaching this far!
I still have some very important topics to talk about: IPC, Synchronization, Priorities, phew... there's a lot to cover!
See you in the next post!

#Repo
[Here](https://github.com/allexfranc/v04.1-memory)


