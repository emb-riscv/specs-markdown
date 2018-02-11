# RTOS Support Features

The RISC-V microcontroller profile is designed not only to be C/C++ friendly, but also with RTOS support in mind.

To make RTOS implementations easier and more efficient, the following features are available:

## Shadow thread stack pointer

Two stack pointers are available, the main stack pointer (MSP) and the thread stack pointer (TSP).

The main stack is the default stack available after reset, and all exceptions and interrupts create
a stack frame on the main stack.

If the application switches to Thread mode, the hart switches to the TSP, while interrupts continue 
to use the MSP.

This solution has several advantages:

* the stack space for each thread needs to cover only the threads needs, and do not worry about 
possible large stack usages in ISRs;
* if a thread corrupts it's stack, it is still likely that the stacks used by the interrupts and 
other threads are intact, thus improving system reliability.

## Stack pointer limit

One of the most common failure cases that occurs while developing multi-threaded applications is 
for one thread to exceed its stack and damage the surrounding memory content.

The solution is to add a system register with a memory address used as lower limit for the stack.
While pushing words on stack, the address is compared and if the limit is reached, an exception is 
raised.

## System clock timer

The system clock timer is intended to drive the RTOS scheduler, and allow to measure durations 
(like timeouts) during normal system operations.

Having an architecture timer allows the RTOS to implement the scheduler code only once, and do not
rely on device specific timers, which require separate initialisation and interrupt handlers for
each specific device.

## Real-Time Clock

The real-time clock is intended to provide the application a way of keeping track of time while the 
device is in sleep mode (and the system clock timer is shut down).

Having an architecture RTC allows to write the code to manage the absolute time only once inside the RTOS, 
and do not rely 
on device specific timers, which require separate initialisation and interrupt handlers for
each specific device.

## Context-Switch interrupt

The context switch interrupt is usually the lowest priority interrupt, and is used as the single point 
of handling context switches, allowing all other interrupt handlers to be written in C/C++ and do 
not bother with context switches at all.

Without such a feature, all application interrupt handlers require an assembly part to handle the 
context switching prior to calling the C/C++ handler.

## Interrupts priorities threshold

Having a mechanism to disable only interupts below a certain threshold greatly improves the real-time
characteristics of a system, by not having to disable all interrupts while handling the system 
data structures. By raising the priority threshold instead of completely disabling interrupts, it 
is possible to keep fast interrupts still active, regardless how busy the RTOS itself is.

## Soft reset

The soft reset is intended to reset the entire device from within.

Having an architecture soft reset allows to write the code to reset the device only once 
inside the RTOS and do not rely 
on device specific code.

## User mode

For security strict applications, the user mode can also be used in conjunction with the Memory 
Protection Unit (MPU), thus further enhancing the robustness of embedded systems.

## Atomics 

The RISC-V 'A' Standard Extension for Atomic instuctions contains instructions that atomically 
read-modify-write memory to support synchronization between multiple RISC-V harts running in 
the same memory space.
