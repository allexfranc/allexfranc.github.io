---
layout: post
title: "STM32 System Monitor"
author: "Allex Franc"
date: 2025-07-14
tags: [stm32, embedded, bare-metal, arm, os, scheduler, round robin, kernel, lcd, debug]
excerpt: "From minimal boot to Operating System - Part 3: Real time System Monitor"
---

I ended the last post saying I would bring somehting lighter. Here we are!
Our Kernel just gained a **System Monitor** and we'll use an LCD to display it's info!
It is very useful for debuging and it has a cool factor to it, i don't know, but it's 
just so satisfying seeing the kernel's vital signs live on a 20x4 LCD.

Let's start with the LCD basic information and it's driver and then dive into the kernel's 
modifications to obtain it's vital signs.

## LCD
I've opted for a [Newhaven Display, 4 Lines x 20 Characters](https://newhavendisplay.com/pt/content/specs/NHD-0420H1Z-FSW-GBW-33V3.pdf). Why? Because Ihad one laying around, plus 
it's cheap and easy to use. With this display I can see some useful information:
- System uptime
- CPU usage per task
- Total system ticks
- Context switches
And obviously much more. It serves also as a printf debuging tool, since I haven't added UART yet.

### Hardware setup
The HD44780 controller (or compatible) uses a parallel interface, and this feels like it''s 2005. Who cares? 
I'm already writing a blog, I'm probably 20 years late anyway. I might even start a FLOG the next month. 
Anyway, here's the chosen connection:

-------------------------------------------------------------
LCD Pin    STM32 Pin    Purpose
D0-D7      PA0-PA7      8-bit data bus
RS         PA8          Register Select (0=command, 1=data)
EN         PA9          Enable (pulse to latch data)
-------------------------------------------------------------

I won't get into much detail about the driver... there's nothing special about it. The initialization follows the 
manual. I wrote some basic functions that we'll be extended when needed.

This is my mantra here (I'll repeat it for the OS when it's ready):
*This is my lcd driver. There are many like it, but this one is mine*

But the summary is this:
1. Put data/command on the data pins
2. Set RS (low for commands, high for data)
3. Pulse EN high then low
4. Wait for the LCD to process

Now that we can display information, let's modify our kernel to track what we want to see.

## Scheduler and Syscall
I started tracking system ticks inside the SysTick_Handler function. It's the obvious place to do that. It also 
updates the ticks for the running task. Context switches are tracked inside the switch_task function. It's 
really a straight forward thing. Add the variables for tracking, than some getters and we are done for the scheduler. 
```c
void SysTick_Handler(void) {
	system_ticks++;
	
	// Count ticks for current running task
	if(tasks[current_task].state == TASK_RUNNING){
			tasks[current_task].run_ticks++;
	}
	
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

uint8_t switch_task(void){
	uint8_t next_task = current_task;
	uint8_t checked = 0;
	context_switches++;

	
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

uint32_t get_system_ticks(void){
	return system_ticks;
}

uint32_t get_context_switches(void) {
    return context_switches;
}

uint32_t get_task_ticks(uint8_t task_num){
	if(task_num >= NUM_TASKS){
		return 0;
	}
	return tasks[task_num].run_ticks;
}

typedef struct{
	uint32_t* stack_pointer;
	task_state_t state;
	uint32_t wake_tick;
	uint32_t run_ticks;
} task_t;
```

Now, it wouldn't be polite for tasks to ask system info directly to the scheduler, so I made them 
a syscall:

```c
uint32_t sys_ticks(void){
	uint32_t result;
	
	__asm volatile(
		"SVC #4			\n"
		"MOV %0, r0		\n"
		: "=r" (result)
		:
		: "r0"
	);
	
	return result;
}

uint32_t sys_context_switches(void){
	uint32_t result;
	
	__asm volatile(
		"SVC #5			\n"
		"MOV %0, r0		\n"
		: "=r" (result)
		:
		: "r0"
	);
	
	return result;
}

uint32_t sys_task_ticks(uint8_t task_num){
    register uint32_t r0 __asm__("r0") = task_num;
    register uint32_t result __asm__("r0");
    
    __asm volatile(
        "SVC #6"
        : "=r" (result)
        : "r" (r0)
    );
    
    return result;
}
```

And they are handled inside the super switch, which asks politely to the scheduler to give these info:

```c
		case SYS_TICKS:
			psp_stack[0] = get_system_ticks();
			break;
			
		case SYS_CONTEXT_SWITCHES:
			psp_stack[0] = get_context_switches();
			break;
		
		case SYS_TASK_TICKS:
			uint8_t task_num = psp_stack[0];
			psp_stack[0] = get_task_ticks(task_num);
			break;
	}
```

Just remember: returns are expected at R0, as well as functions arguments are expected to be at R0, R1, and so on... 
Apply some assembly magic and voila!

## The System Monitor Task
Now for the fun part - a task that displays real-time system metrics! 
This is where our kernel becomes self-aware (in a good way):
BTW, I named my OS **AllexOS**, like Alexa, but better! I think the name is very fitting...

```c
void task_system_monitor(void){
    // Static variables to track deltas
    static uint32_t last_check_ticks = 0;
    static uint32_t task_last_ticks[4] = {0};
    static uint8_t first_run = 1;
    
    while(1){
        uint32_t system_ticks = sys_ticks();
        
        lcd_set_cursor(0, 0);
        lcd_write_string("AllexOS   U:");
        
        uint32_t total_sec = system_ticks / 1000;
        uint32_t hours = total_sec / 3600;
        uint32_t min = (total_sec % 3600) / 60;
        uint32_t sec = total_sec % 60;
        
        lcd_write_2digit(hours, 0);
        lcd_write_char(':');
        lcd_write_2digit(min, 0);
        lcd_write_char(':');
        lcd_write_2digit(sec, 0);
        
        // Calculate CPU percentages over window
        uint32_t percentages[4] = {0};
        
        if (!first_run) {
            uint32_t delta_time = system_ticks - last_check_ticks;
            if (delta_time > 0) {
                for (int i = 0; i < 4; i++) {
                    uint32_t task_current_ticks = sys_task_ticks(i);
                    uint32_t task_delta_ticks = task_current_ticks - task_last_ticks[i];
                    percentages[i] = (task_delta_ticks * 100 + delta_time/2) / delta_time; // Round up
                    task_last_ticks[i] = task_current_ticks;
                }
            }
        } else {
            // First run - use total time
            for (int i = 0; i < 4; i++) {
                task_last_ticks[i] = sys_task_ticks(i);
                percentages[i] = (task_last_ticks[i] * 100 + system_ticks/2) / system_ticks;
            }
            first_run = 0;
        }
		
        last_check_ticks = system_ticks;
        
        lcd_set_cursor(1, 0);
		
		lcd_write_string("I:");
        lcd_write_2digit(percentages[IDLE_TASK_NUM], 1);
        lcd_write_string("%G:");
        lcd_write_2digit(percentages[GREEN_TASK_NUM], 1);
        lcd_write_string("%B:");
        lcd_write_2digit(percentages[BLUE_TASK_NUM], 1);
        lcd_write_string("%M:");
        lcd_write_2digit(percentages[MONITOR_TASK_NUM], 1);
        lcd_write_char('%');
        
        // Third line: System ticks
        lcd_set_cursor(2, 0);
        lcd_write_string("Ticks: ");
        lcd_write_uint(system_ticks);
        
        // Fourth line: Context switches
        lcd_set_cursor(3, 0);
        lcd_write_string("Switches: ");
        lcd_write_uint(sys_context_switches());

        
        sys_sleep(150);
    }
}
```

The monitor calculates CPU usage using a sliding window approach. 
Instead of showing all-time averages (which would never change much), we calculate usage over the last 150ms (ish), 
look at the delta_time variable and the sys_sleep(150); 

### Understanding CPU Usage Calculation
The key insight is using a sliding window instead of all-time averages:
Time:     T0 -------- T1 (150ms later)
Task A:   1000 ticks   1135 ticks     -> 135 ticks used
Task B:   500 ticks    515 ticks      -> 15 ticks used
Total:    150ms = 150 ticks available
Task A CPU% = (135/150) * 100 = 90%
Task B CPU% = (15/150) * 100 = 10%

## What the display shows
AllexOS   U:00:12:34    <- System name and uptime
I:85%G:5%B:5%M:5%       <- CPU usage per task
Ticks: 754321           <- Total system ticks  
Switches: 75432         <- Context switches

The task abbreviations are:

I: Idle task (should be high when system is not busy)
G: Green LED task
B: Blue LED task
M: Monitor task (this one!)

With all this monitoring in place, you might wonder about the overhead...

## Performance Considerations
You might wonder: doesn't the monitor task affect what it's measuring? Absolutely! 
This is the observer effect in action. The monitor task itself consumes CPU time, which shows up in the measurements.
In our case, updating the LCD 20 times per second (50ms sleep) gives us responsive updates without overwhelming the system. 

## Common Issues & Solutions
- **LCD shows garbage**: Check your timing delays. Different LCDs need different delays.
- **Monitor task uses too much CPU**: Increase the sleep time or reduce update frequency.
- **CPU percentages don't add to 100%**: This is normal and happens for several reasons:
  1. **Rounding errors**: Integer math (like `(ticks * 100) / total`) loses precision
  2. **Interrupt overhead**: Time spent in interrupt handlers (SysTick, PendSV) isn't attributed to any task
  3. **Measurement windows**: The monitor task's calculations might miss ticks between measurements
  
  Consider displaying raw tick counts for debugging. If your percentages are way off (like only adding to 70%), 
  check if you're spending too much time in interrupts or context switching.

## Result
![LCD showing system stats]({{ "/images/lcd-system-monitor.jpeg" | relative_url }})

## Stay tuned!
Thank you for reading and reaching this far!
The next post is a big one! Memory allocation and Adding tasks dynamically... Oh boy! And of course the system monitor will reflect that!
In for a penny, in for a pound! Let's do this...

## Repo
[Here](https://github.com/allexfranc/v03.2-basic-kernel-system-monitor)