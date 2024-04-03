---
title: Accurately timing some code
description: Here's (almost) everything you need to know about timing your CPU-side code accurately
date: 2024-04-01 19:00:00-0800
image: header.jpg
categories:
    - Systems
tags:
    - systems

math: true
toc: true
---

In the world of computers, everyone cares about things being faster. If you are a gamer, more FPS (frames per second) matters to you. If you are a software developer/engineer, you care that your code is more performant. But, how do you accurately measure the time your code takes to run?

## Introduction

Let us take a simple example. Imagine that you are solving a LeetCode question like the one [here](https://leetcode.com/problems/remove-duplicates-from-sorted-array/description/). You wrote what you think is a performant code, but when you submit, you see that your code does better than only 20% of the submissions! You frantically try to figure out why the performance of your code is so poor. However, you decide to submit it again (without any modifications), but this time, it shows that it fared better than 70% of the submissions! Umm... What?

Let us delve into why such drastic differences exist. There are four primary reasons why the timing can be so inaccurate

1. The language you are programming in. Is it [compiled or interpreted](https://blog.rutvora.com/p/introduction-to-compilers-and-interpreters/)? Does it have a complex runtime and/or a [garbage collector](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))?
2. How many processes are running in your system at a given time? One? Or a million?
3. What time unit are you trying to measure, and to what accuracy? Is it of the order of nanoseconds, milliseconds, hours, or days?
4. What timer mechanisms are you using, and what hardware are you running on?

Let us break down each of these:

## The impact of Languages

Some of you might be fond of C/C++ because of it's performance, while some others may be fond of Python because of it's simplicity and versatility. Each one has it's own pros and cons. However, the choice of language determines a hoard of aspects that impact your timing measurement

### Compiled vs Interpreted

While I have tried to cover the Compiled vs Interpreted argument [here](https://blog.rutvora.com/p/introduction-to-compilers-and-interpreters/), let us discuss why it impacts the timing measurement.

Compiled code is a native (or close to native in the case of Java, WASM, etc.) binary, meaning that it is already in the forms of 0s and 1s that the computer can understand. So, in this case the code executes as it is, without any additional logic (which you have not written or imported). However, in the case of an interpreted language, the code is in the human-readable syntax (e.g. Python or JS), and is parsed line-by-line and converted to machine code. Thus, additional time and variability are introduced due to the extra code for the parser/translator.

| Language | Your code | Machine Code |
| :------: | :-------: | :----------: |
| C | <pre>int a = 10; <br> int b = 20; <br> int c = a + b</pre> | <pre>mov $10, %%r15 # Move 10 to register 15 <br> mov $20, %%r14 <br> add %%r13, %%r14, %%r15 # Add r14 and r15, place the resule in r13 </pre> |
| Python | <pre>a = 10 <br> b = 20 <br> c = a + b</pre> | <pre># Multiple lines to parse c = a + b and generate the rest of this code <br> mov $10, %%r15 # Move 10 to register 15 <br> mov $20, %%r14 <br> add %%r13, %%r14, %%r15 # Add r14 and r15, place the resule in r13 </pre> |

The additional code is dependent on the length of the lines in your code, alongside other factors. This makes your code more prone to variation in timing.

### Complex Runtime and Garbage Collection

A lot of managed languages like [Go](https://go.dev/) or [Java](https://www.java.com/) have complex runtimes. The runtimes track each memory allocation and variable to check when it goes out of scope. They dynamically delete the memory associated with such variables to ensure reduced RAM utilisation. They may also perform some runtime checks on the bounds of the arrays or other elements you are accessing, or do more complex tasks. However, all of this consumes CPU time. 

So, imagine that you are trying to run a code that should finish in 1us, but in the middle of the code, the runtime decides that your task is not important, and that running the garbage collector is more important! So, it pauses your code, runs the GC, which takes ~5us, and then switches back to your code. If you compare "wall-clock" time, it took 6us instead of 1us for your code to result in an output. 

The higher the complexity of the runtime, the harder it is to determine what contributes to the extra time and the variability in the timing.

## Impact of other processes/threads

Unless you are hand-writing an entire bare-metal software with no dependencies, there will always be some other piece of code running on the system apart from yours. This could be some other processes that you have run (e.g. your browser), or a process run by another user, or the operating system itself. 

You can check the total number of processes and threads active on your system. Windows users can open "Task Manager" `Ctrl + Shift + Esc`, Linux users can install and run `htop`, Mac users can look at "System Monitor". You will usually notice ~200-400 processes and ~5000-10000 threads. However, your device has 4-32 cores, usually. So, how can so many processes run simultaneously? The answer is: they don't!

Your operating system (OS) is an extremely complex piece of software, designed by probably the smartest engineers in the world. It can dynamically determine which process needs to run next so that the user has the smoothest experience possible. It also determines which of the _n_ cores of your machine should the program run on. The OS runs a `scheduler` thread per core, and maintains a queue of tasks, which need to be executed on the core. If the queue is too long, the OS migrates some of the programs to another core, if possible. The OS also receives _ticks_ from the hardware, which enforce the `scheduler` to run, irrespective of what was running before the arrival of the _tick_. While this is helpful in ensuring that one process can't hog the core, it is a nightmare for measuring the execution time of your code, as your process can be scheduled out at any time!

The easiest solution to this is to ensure that no other process (except the scheduler and your program) runs on that core. On Linux, this can be done with a combination of `isolcpus`, `cset shield` and `irq_balance` (More on these [here](https://github.com/rutvora/cpp-helpers/blob/main/src/cpu/README.md))

However, these do not prevent language runtime-provided user-space parallel processing mechanisms like [goroutines](https://go.dev/tour/concurrency/1). You either have to ensure that your code has only one routine/thread, or you have to modify the runtime yourself!

## NanoSeconds or Days?

Most of the information in this article is only useful if you want your timing to be more accurate than milliseconds. If you don't care about your timing being off by a second or more, you don't need to read this! All of the scheduling parts, the language runtime parts, etc. vary your timing by less than a second. 

However, different timing mechanisms provide different granularity and accuracy of measuring the timing. Hence, the choice of the timer and the code itself can make your timing measurement more/less volatile.

## Choice of Timers

While a variety of languages provide a variety of timers, we will focus on C/C++. Each timer has it's own pros and cons

### `gettimeofday()`

(Alternatively `std::chrono::system_clock`)

This timer is a C/C++ standard library timer that returns the current sytem time since epoch (i.e. Jan 1, 1970). The `gettimeofday()` is only accurate in the order of microseconds, while the C++ counterpart can tell you nanoseconds since epoch. However, both of these read the current system time, which might get updated due to clock drifts and subsequent corrections by the system, using [NTP](https://en.wikipedia.org/wiki/Network_Time_Protocol). Meaning, the following code can have negative results:

```c++
auto start = std::chrono::system_clock::now();
// Do something
auto end = std::chrono::system_clock::now();
std::cout << end - start << std::endl; // Can be negative :(
```

### `std::chrono::steady_clock`

(Alternatively `std::chrono::high_resolution_clock`, `clock_gettime(CLOCK_MONOTONIC, <var>)`)

This timer is a C/C++ standard library timer that provides the maximum granularity of time provided by the operating system (nanoseconds in most Linux distributions). However, this timer itself has some overhead as it executes some code of its own, and may also result in a system call. So, the following code will result in a "high" difference:

```c++
auto start = std::chrono::steady_clock::now();
auto end = std::chrono::steady_clock::now();

std::cout << end - start << std::endl;
```

_Note: You may run this and get the result as 20-50ns, which is high for a computer, as it can process anywhere between 80-450 instructions in that time_

These timers are great if you are measuring something where a variation in the timing of less than a microsecond (or maybe 100ns) is acceptable!

### Cycle Count

This is not a timer per-se, but rather a measurement of the cycle count of the CPU core. Each CPU core (whether it is Intel, AMD, ARM, RISC-V or something else) has a separate counter measuring the cycles of the CPU. Depending on the implementation of the hardware, the counter may be updated at a dynamic or a fixed frequency. However, in most cases, this timer can be used to accurately measure execution times less than 100ns!

However, using these timers come with certain challenges:

1. You need to ensure that your program is pinned to one core. Each core (possibly) has an independent counter, which might not be synchronised with the other cores. 
2. The kernel on the CPU can deny permission for the user-space programs to read this counter. You need to check of this and have a fall-back mechanism
3. Different hardwares update this counter at different frequency, so you can't directly compare numbers across machines.

Assuming you have taken care of these things, let us see the timers available:

**[RDTSC](https://www.felixcloutier.com/x86/rdtsc)**:  

This timer is available on all x86 (Intel and AMD) machines and provide you with cycle count on that core. 
On Intel machines, the associated counter is updated with the "Base clock" frequency of the CPU (the "3.2GHz" part of the "Intel Core i7 @ 3.2GHz"). "Base Clock" is usually the highest frequency the CPU can achieve without Turbo Boost or OverClocking.

However, this instruction is not serialising, meaning that it can be executed before or in parallel with your other instructions (all modern processors enable out of order execution):

```asm
rdtsc # Start
# Some op1
# Some op2      <- rdtsc (End) may execute here
# Some op3
rdtsc # End
```
As a result, this instruction needs to be paired with some other serialising instructions like [CPUID](https://www.felixcloutier.com/x86/cpuid) or [SERIALIZE](https://www.felixcloutier.com/x86/serialize)

**[RDTSCP](https://www.felixcloutier.com/x86/rdtscp)**:

This timer is also available on all x86 (Intel and AMD) machines. It is similar to the RDTSC instruction and reads the same counters, but is also serializing, to ensure that all previous instructions retire before the execution of the RDTSCP. On Intel machines, this is probably the most accurate timing counter you can have.

**[RDPRU](https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24594.pdf)**:

This instruction is exclusively available on AMD. While AMD also supports RDTSC(P) instructions, the hardware updates the corresponding counters with the frequency of the Crystal Clock, instead of the base clock. The crystal clock is ~100MHz, as opposed to the base clock of 2-5GHz. As a result, executing RDTSC on AMD machines will result in a time difference of a multiple of $ base_clock/crystal_clock $ (the counter is updated by this ratio at every tick of the crystal clock). RDPRU, on the other hand uses a different counter, which is updated at the base clock frequency, making it more accurate!

**[MRS %0, CNTVCT](https://developer.arm.com/documentation/ddi0601/2022-09/AArch64-Registers/CNTVCT-EL0--Counter-timer-Virtual-Count-register?lang=en)**

This instruction is a counterpart of the RDTSC instruction, but for ARM64 machines. It is not serializing and needs to be paired with some [serialization](https://developer.arm.com/documentation/dui0802/b/A32-and-T32-Instructions/DMB--DSB--and-ISB) to ensure proper timing measurements. However, this may be updated [less frequently](https://developer.arm.com/documentation/102379/0103/What-is-the-Generic-Timer-?lang=en) than the RDTSC counterparts, reducing the observation granularity. 


## End Notes

Hopefully, you are now familiar with almost all the variables that can affect your timing measurement. If you do not want to put in the effort of coding all of these, I have tried to create a C++ module to do these things for you. You can go check it out [here](https://github.com/rutvora/cpp-helpers/)
