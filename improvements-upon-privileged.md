# Appendix A: Improvements upon RISC-V privileged

## Rationale

As mentioned in RISC-V Volume I, v2.2, the _"RISC-V is a new instruction set architecture 
(ISA) that was originally designed to support computer architecture research and education. 
... The RISC-V manual is structured in two volumes. This volume covers the user-level ISA 
design, including optional ISA extensions. The second volume provides the privileged 
architecture."_

The RISC-V Volume II, v1.10, mentions: _"... This document describes the RISC-V privileged 
architecture, which covers all aspects of RISC-V systems beyond the user-level ISA, 
including privileged instructions as well as additional functionality required for 
running operating systems and attaching external devices."_

This is great news for the GNU/Linux community and for the academia, but attempts 
to identify in the RISC-V specs how the new design meets the requirements of bare-metal 
embedded devices were not very successful; browsing the two docs revealed only some 
references to Tensilica and ARC (probably not the most successful embedded architectures), 
and some incomplete specs for **the RV32E subset** (which halves the number of general 
registers, do not support hardware floating point and makes some counter instructions 
optional).

According to the privileged specs in Volume II, **RISC-V embedded systems share the 
exact same definitions as systems running Unix-like operating systems, but they do 
not include the "S" (Supervisor) mode features**.

This strategy does not work very well for real-time systems; for example, **in the 
RISC-V interrupt model**, without special measures, **interrupts remain disabled 
while executing interrupt handlers**. This may be acceptable for general purpose 
Linux kernels, but for hard real-time systems this is generally a no-go, since 
**interrupt latency** may end up well above tolerable limits.

### The dividing line

Currently there is no clear understanding where the dividing line between RISC-V 
general purpose and microcontroller devices should be. 

One possible approach is to start by defining what microcontroller devices are not: 
they definitely are not expected to run multi-process applications on top of 
Unix-like operating systems. Although some 
projects try to challenge this, it is generally agreed that **Unix-like operating 
systems DO need virtual memory and supervisor modes** to properly run multi-process 
applications.

After long considerations, the conclusion was that the common and logical dividing 
line between the RISC-V privileged profile and a RISC-V microcontroller 
profile is the use of virtual memory and supervisor modes; as such, **RISC-V 
microcontrollers are devices 
that do not implement a virtual memory system or supervisor modes** and are 
intended to run single process multi-threaded applications.

> <sup>[JB] Two more criteria may be used
for dividing microcontrollers and application processors: pipeline
complexity and memory latency. Microcontrollers use simpler, in-order
pipelines and have memory subsystems that are tightly synchronized to
the execution pipeline. ... Out-of-order and parallel
execution are becoming common features in application processors, but
are not used in microcontrollers, since the latter must have predictable
execution timing. [ilg] The pipeline complexity and memory latency should
  be implementation specific. Microcontrollers intended for applications that
  need predictable execution timings may decide not to implement 
  out-of-order and parallel execution. 
</sup>

## Improvements upon RISC-V privileged

The main 'pain-point' with the current RISC-V privileged specs
is the mechanism to handle interrupts, which is not suitable for real-time,
low power, bare-metal embedded applications.

The following issues were identified in the current RISC-V privileged specs when
used for bare-metal applications:

| RISC-V Privileged | RISC-V Microcontroller |
|-------------------|------------------------|
| Handlers run with interrupts disabled; low priority interrupts that take a long time to complete may delay high priority interrupts, affecting real-time capabilities. | The microcontroller profile allows nesting; high priority interrupts preempt low priority ones, being processed as fast as possible. |
| There is only a single trap handler, serving all interrupts and exceptions (the so called _vectored_ mode is so complicated to use that it is not even worth mentioning). | The microcontroller profile has an advanced vectored mode; interrupts are dispatched to separate handlers, via a simple array of pointers, easy to define in C/C++. |
| The interrupt code must be written in assembly, to perform the low level stacking/unstacking and return from exception; this code **is** complicated, a good example is the [Linux handler](https://github.com/torvalds/linux/blob/master/arch/riscv/kernel/entry.S), and the current Linux implementation does not even re-enable interrupts while in handler mode. | The microcontroller profile automatically performs the stacking/unstacking, allowing all application interrupt handlers to be written as C/C++ functions, with minimum latency. |
| The current ISA Volume I manual defines a common POSIX ABI to be used by all devices, but this ABI requires the caller to save a lot of registers, making interrupt stacking/unstacking very expensive and increasing latency. | Better adapted to real-time, the microcontroller profile defines a lighter Embedded ABI, reducing latency. |
| The privileged profile defines a few hundred CSRs, and encourages implementation to define even more custom CSRs; current debuggers do not have support for proprietary mechanisms like CSRs, and viewing/changing these registers requires unusual hacks. | The microcontroller profile uses a very limited set of CSRs and favours the use of memory mapped registers, which are very well supported by debuggers/IDEs, including via detailed peripheral register viewers. |
| The current RISC-V ISA does not explicitly define a stack (it is only mentioned in the POSIX ABI), and there is no separate stack for interrupts; in a multi-threaded application, interrupts can occur on any thread stack, thus when provisioning for thread stacks, the additional memory requirements of all interrupts must be added to all thread stacks, wasting precious RAM. | The microcontroller profile not only defines the stack pointer register, but also adds a shadow thread stack pointer, separate from the main stack used by the interrupts, improving RTOS implementations and reducing tread stack requirements for RTOS multi-threaded applications. |
| A common reason of crashes during embedded systems development is one of the threads running out of space; the specs do not provide a standard way of detecting stack overflows. | The microcontroller profile adds a stack limit register and stack overflows trigger exceptions. |
| The system clock runs from the low frequency real-time clock, which has low resolution and, at common 32768 Hz frequencies, does not allow accurate 1000 Hz scheduler clocks. | The microcontroller profile defines separate low-power real-time clock and high accuracy system clock, improving both general clock resolution and scheduler clock accuracy. |
| There is no explicit mechanism to trigger and implement context switches in a multi-threaded RTOS. | The microcontroller profile adds a dedicated interrupt, guaranteed with the lowest priority, to be used for all context switches, relieving all other interrupt handlers from this duty. |
|| The microcontroller profile adds an architecture device reset mechanism. |
|| The microcontroller profile adds an architecture resumable NMI. |
| The startup code also requires some assembly code, to set the stack pointer and the `gp` register. | The microcontroller profile adds a simplified device startup code, based on a table of standard C/C++  pointers, requiring no assembly code at all. |

## Criticism

### Fragmentation would break upward compatibility

While discussing the opportunity for a new RISC-V embedded profile, the most common 
concern raised was that migrating an applications written for a microcontroller to a 
larger application class core would be more difficult.

Well, yes, in theory it might be possible to design a board in such a way to allow to 
swap in a bigger core, and to design the application in such a way to ignore the MMU 
and the supervisor mode and continue to use a RTOS; the application will probably run 
faster due to the improved pipelines and core clock rates, but there are several 
practical issues:

- in industrial embedded applications the processor selection is not based on the 
architecture (which in the majority of cases is Cortex-M only), but on the available 
on-chip peripherals; it is very unlikely to find an application class core with the 
desired peripherals available on a microcontroller;
- application class cores generally do not have internal flash/ram, requiring 
external chips; external memory chips require lots of address and data pins, which 
mean large BGA chips, larger & more complex PCBs, and generally higher costs. 

So this concern is not realistic, and not accepting a distinct microcontroller 
profile, optimised for real-time applications simply for maintaining compatibility 
with the privileged specs is not a beneficial approach.

### No need to, everything will run Linux in the future

> "In the future everything will run Linux, so defining separate non-Linux profiles 
is a futile exercise."

Yes. Sure. Eventually. No doubt about it. When waiting long enough many marvellous things can happen. 

However, for those who are not ready to wait for the kingdom come, having simpler
devices for critical real-time applications is a requirement for today.

### Automatic stacking/unstacking is evil

> "Automatic stacking/unstacking is fine for Cortex-M, but it is 
very objectionable for 
RISC-V. The difference is in ARM's MOVEM instruction. A Cortex-M 
already has the hardware to move multiple words to/from the stack 
because it has to implement the MOVEM instruction anyway. So doing 
this specially on trap entry/exit is a small addition to what is 
already required to execute the user instruction set. The story for 
RISC-V is different, as it has no MOVEM-like instruction. Therefore, 
having the hardware automatically push/pop a collection of registers on 
trap entry/exit is a larger addition for RISC-V than it was for ARM."

Well, that's a point of view. However, it must be noted that even for 
the tiny Cortex-M0, so economical in terms of transistors, ARM decided 
to do automatic stacking/unstacking, so the added complexity might not 
be that high.

Not to mention another detail: Cortex-M has 16 registers, and the
EABI requires R0-R3, R12, R14, PC and xPSR
to be stacked/unstacked automatically. On the other hand,
the LDM/STM instructions, probably due to to the tight encoding, 
are able to move only half of the registers,
(R0-R7), so the logic to do the stacking/unstacking is
definitely more capable than required by the instruction set.
It is not by accident that Cortex-M has automatic stacking/unstacking 
simply because support for LDM/STM was present anyway, it is
by design. As an exercise of reversed logic, 
it might also be argued that Cortex-M has the LDM/STM instructions 
because the logic for moving multiple words was already
available from the automatic stacking/unstacking mechanism.

Also RISC-V having no MOVEM-like instructions may save a few 
transistors, but otherwise this is not exactly a feature, 
it simply makes saving contexts in multi-threaded environments more
complicated and possibly less efficient.

### Automatic stacking/unstacking the interrupt context increases latency

The current RISC-V ABI requires the caller to save the following registers: 
`ra`, `t0`, `t1`, `t2`, `a0`, `a1`, `a2`, `a3`, `a4`, `a5`, `a6`, `a7`, `t3`, 
`t4`, `t5`, `t6`. This amounts to 16 registers. If floating point is used, 20 
more registers must be saved.

The current RISC-V privileged specs do not define a hardware stack and do not 
require the core to save any registers 
on the stack, delegating this to the assembly trap handler.

Some voices claim that this strategy allows the application to have a highly optimised 
assembly trap handler, and as such avoid pushing all registers.

Well, yes, in theory, for very simple (blinky) applications this might be so, but for 
real applications, regardless how optimised is the assembly trap handler, at a certain 
point it'll need to call a C function (for example to access a system service, like 
posting to a semaphore), and at this point the entire ABI caller register set must be 
pushed onto the stack, and popped after the C function returns. 

The result of this strategy is that the assembly trap handler will initially save a 
small number of registers (those known to be used by the handler), then save the rest 
of the register set to prepare for the C call, so the full register set must be 
saved anyway.

For the current ABI this still means 20 registers, which is a lot. The real problem 
here is not the decision to save them automatically (which greatly simplifies the 
software), but the current ABI which is designed for user mode Unix applications.

The solution is a **separate Embedded ABI (EABI)**, optimised for embedded real-time 
applications, with a smaller caller register set.

### Automatic stacking/unstacking should be replaced by compiler attribute

> "Better to have a “handler” function attribute that causes the compiler 
to save only and exactly the registers the function modifies. If a handler 
function calls a regular C function then it needs to save all the volatile 
registers first.

Well, yes, as argued before, for very simple applications 
it is possible to imagine interrupt handlers incrementing a 
variable and 
returning, but this is rare, by far the biggest majority of interrupt 
handlers call C/C++ functions to perform system services, like posting 
a semaphore, or any other synchronization mechanism. 

So, by first saving a few registers and then saving all those required 
by the ABI, the result is that more registers needs to be saved, and 
further worsening the latency. 

### Assembly interrupt handlers should be ok, they reside in the system part

> "Why insist on having the interrupt handlers written in C, when they can be very well
written in assembly, since they reside in the system part, written by the system
programmers, not by the application programmer."

Well, this might be the case for Linux, where the kernel and the modules are 
indeed written by system gurus, but in embedded bare-metal applications the
interrupt handlers are very application specific and cannot be part of
the RTOS or drivers/libraries, so it is the application programmer who must 
write them, not someone else, thus the need for the interrupt handlers to be
as easy to write as possible, the best choice being to have them defined as plain 
C/C++ functions.

> "Interrupt handlers do not need to be entirely in assembler, only 
the entry/exit millicode needs to be part of the system. That millicode 
*can* be written by system gurus, while the application ISRs, written by 
application programmers, are called via the millicode. Unless 
there is some faster memory access cycle that the hardware can use, 
automatic context save/restore (presumably in microcode) will be no 
faster than RISC-V millicode."

Well, reversing the logic, there is no best case scenario when the millicode
will be faster than the microcode; even when there is no faster memory access 
for the microcode, the millicode will still have to make a call to the actual
interrupt handler so the total timing cannot be better. The main difference 
is the ease of use, the application programmer will no longer need any guru to
write the millicode.

### CSRs cannot be memory-mapped

Another almost 'religious' RISC-V issue is related to accessing the system registers. 
Before a more elaborate explanation, those who claim this should remember that 
the current privileged specs moved `mtime` and `mtimecmp` from CSRs to 
memory mapped, and the PLIC specs require all registers to be memory mapped.

Generally, with 
the exception of a very limited set of special cases, industry standard 
architectures map most of the system peripherals and registers to a memory area. 
Instead, the 
RISC-V ISA defined several special instructions allowing to address 4096 per-hart 
registers.

It is generally agreed that for application class devices, with complex out-of-order 
pipelines, the current mechanism has several 
advantages. Unfortunately, the RISC-V privileged specs abused this 
mechanism, and now there are several hundred registers defined in this proprietary 
space, some of them even read-only, and obviously creating no security threads (like 
`mvendorid`, `marchid`, etc).

From the point of view of a microcontroller profile, this mechanism of accessing 
the system registers has two main disadvantages:

- requires assembly code to access each individual register 
- it is not supported by current development tools (debuggers have no ways of accessing 
these registers, IDEs have no special views for them, etc).

Mapping system registers in the memory space is perfectly possible, and the RISC-V 
privileged specs even mandates for some registers like the `mtime` and `mtimecmp`, 
not to mention the PLIC, to be memory mapped.

However, from a technical point of view, for virtual memory systems, accessing system 
registers from code running in user mode requires 'punching' some holes into the 
virtual memory space to reach the special memory mapped registers, which adds some 
complexity, and may cause havoc to the pipelines. But, since the specs require this 
mechanism for `mtime` and `mtimecmp`,
it no longer matters if there are two or more such memory mapped registers.

Fortunately, microcontrollers running without a MMU do not have this problem, 
accessing any memory mapped registers is usual, and the cost of doing so is 
perfectly acceptable.

Plus that in the microcontroller profile there are _no_ hardware security 
boundaries, so the risk of attacks somehow exploiting the CSR-as-MMIO is a 
non-issue. 

### The hardware stack limit register is expensive

> "The stack limit register needs to be read and compared on every 
store via the stack register so it should have dedicated read circuit 
and comparator.

Yes, it is a small price to pay, but by far the most common cause of crashes 
in a multi-threaded device is stack overflow, so detecting this exception
should be worth the extra price. 

### Microcontrollers do not need privilege levels

> If microcontrollers do not run a kernel, why have privilege levels? 

It is true that microcontrollers do not run a 'unix kernel' (they run a 'scheduler'). 
But for some security concerned applications, microcontrollers can run the 
application code in unprivileged mode and the scheduler/drivers in 
privileged mode. 

ARM Cortex-M devices can run code in unprivileged mode, and new
Cortex-M23/M33 devices even have a TrustZone security feature.
Also most of the Cortex-M devices have an MPU, which prevents unprivileged 
code accessing system memory/registers. 

Using the unprivileged mode is not at all unusual,
[ARM CMSIS](http://www.keil.com/pack/doc/CMSIS/General/html/index.html), the industry 
software standard for Cortex-M devices, includes a component called CMSIS RTOS, and 
the reference implementation is 
[Keil RTX](http://www.keil.com/pack/doc/CMSIS/RTOS/html/rtxImplementation.html), 
which by default runs application code in unprivileged mode. 

It is true that, with all ARM marketing, RTX is not the most successful RTOS, 
but even FreeRTOS has a mode in which the MPU can be activated. 

### C embedded system programmers vs C embedded application programmers

> C embedded systems programmers might be used to accessing peripheral via 
registers, but C embedded application programmers are used to accessing peripherals 
via system calls. C embedded system programmers are also used to writing ISRs in assembly.

The distinction between system and application programmers stands perfectly true for 
Unix-like systems, where a small team of highly experienced system programmers write 
the low level kernel code and the device drivers, allowing millions of application 
programmers to access all required resources via system calls, without bothering 
with details.

However, in the embedded world, this distinction is almost non existent, C embedded 
programmers are both application and system programmers. On one hand they need 
full and unlimited control of the hardware, and on the other hand they would 
like too have access from high level C/C++ code. 

Having to use assembly code is definitely not a joy for modern embedded programmers, 
especially since Cortex-M came to market in 2004, and allowed to write interrupt 
handlers directly in C/C++, without any assembly stubs, millicodes or compiler 
attributes/pragmas.

### Comparisons with ARM are meaningless

> "Arguments like 'ARM does this' are very weak for a feature in RISC-V"

Well, the creators of the RISC-V instruction set probably have all reasons to be 
proud of their design, and many in the academia may consider that application 
class devices based on RISC-V may very well make ARM similar devices irrelevant,
but in the embedded space, ARM Cortex-M **is** the industry standard, and 
disregarding it is not beneficial.

Except the auto industry, which is more conservative, where old proprietary
cores still have a significant market share (but losing it), and some very
cost driven applications,
where 8-bit microcontrollers are considered still good enough, the majority of the
silicon vendors
now sell microcontrollers based on Cortex-M cores; the trend is clear,
it was observed for more than 10 years, and the Cortex-M market share is
expected to continue to increase in the years to come.

ARM tried to sell licenses for microcontrollers even before the Cortex-M family
was created, but with very limited success. The devices were very similar to
their application cores, and used the same solutions, for example a single 
interrupt handler, and lots of assembly code required to start and make use of 
core.

There may be multiple reasons why Cortex-M was so successful, but the main one
probably is the ease of use, and the C-friendliness, by design.

This lessened the need for a C system programmer to act as guru, and allowed
C application programmers to fully take control of their applications.

> "RISC-V microcontrollers should compare to PIC or AVR devices"

This was probably true 10-15 years ago, but today it is no longer the case. 
Not only the industry migrated to 32-bits cores, but the ecosystems around 
Cortex-M and the ease of use made most of the other cores irrelevant.

There is also another fact to be noted: according to several studies, the 
world wide population of programmers is doubling every few years (let's 
say N, less than 10). Assuming a constant share for the embedded programmers, 
statistically half of them have less than N/2 years of experience, and most 
of these new (relatively inexperienced) programmers met Cortex-M as their 
first architecture (which is already 14 years old!). They may have heard of
PIC and AVR, but never had to write assembly interrupt handlers, and
asking them to do so will obviously be seen as a major step backward.

### Microcontrollers should not be on networks

> "Generally, microcontrollers should probably not be on networks, except
possibly for multi-core versions that can handle real-time tasks on one
core and network latency on the other."

Yes, multi-hart devices would be excellent for hard real-time applications,
by allocating separate harts for each critical task,
but with nested, pre-emptive high priority interrupts, even a single hart device
can handle multiple tasks very well, and if the real-time tasks are driven by ISRs,
then the network stack can run at a lower priority.

### 64-bits microcontrollers will never be needed

> "Why 64-bit? Any system big enough to need more
than 32-bit addressing is probably already running an operating system
like Linux."

That's a good question. By the time 8051 was king, many questioned
why would someone think of 16-bit microcontrollers. While some were
debating this, vendors gradually offered devices with 12 address
bits, then 16 bits, even 24 bits. Cortex-M came boldly and provided
32-bits registers and a large (32-bits) linear address space.

Although a 4 GiB memory space may be enough for most current devices,
it should be noted that 64-bits devices bring not only a wider memory 
space, but also 64-bits registers, and native atomic 64-bits accesses.

Applications with lots of integer arithmetic may benefit from 64-bits 
cores.

Also applications with large and fast timers benefit from atomic 64-bits 
accesses, which otherwise require a lot of juggling on a 32-bits platform 
(see the recommended RISC-V mechanism to access the timer registers on 
a 32-bits device). 

## Proposed steps to change the current RISC-V specs

It is not realistic to expect a new set of RISC-V microcontroller specs to be 
adopted overnight. However, given the expected ratification of the current specs by 
the RISC-V Foundation, it is quite urgent to ensure that this process will not block 
further developments in the embedded/microcontroller space.

This will probably require several steps, but the main ones are:

- acknowledge that microcontroller devices have different requirements 
compared to systems running Unix-like operating systems
- acknowledge that the solutions provided by the current privileged mode 
specs are not optimal for real-time, low power, bare-metal embedded applications
- acknowledge the need for changes in the current specs 
- relax the requirements for the privileged specs
- create new specs for the microcontroller profile

### Acknowledge the need for the changes

Given the current structure of the RISC-V Foundation, with most of the efforts
focused on finalising the specifications required for running Unix-like
operating systems, acknowledging that the specifications for general
purpose devices do not work very well for real-time systems will be a challenge.

### De-entangle the privileged specs

The RISC-V Volume II, v1.10, mentions: _"... the entire privileged-level design 
described in this document could be replaced with an entirely different 
privileged-level design without changing the user-level ISA, and possibly without 
even changing the ABI. In particular, this privileged specification was designed 
to run existing popular operating systems, and so embodies the conventional 
level-based protection model. Alternate privileged specifications could embody 
other more flexible protection-domain models."_

So, at least in theory, it should be possible to extend the specs, but in 
practice it is not clear how exactly this can be done. Ideally, **the Volume 
I should not explicitly refer to Volume II**, or should refer to it as optional, 
leaving room for a complementary specification for other classes of devices,
including microcontroller devices.

As a parenthesis, the RISC-V ISA specs provide a very high degree of flexibility 
allowing for custom extensions for the instruction set, but they are still very 
rigid by insisting that all these devices should be able to run Unix-like 
operating systems.

### Move all mandatory CSRs to the privileged specs

Apart from relaxing the need for the privileged specs, the instruction set defined 
by Volume I is generally acceptable for microcontroller devices.

The only notable exception is in Chapter 2.8, the `rdcycle`, `rdtime` and `rdinstret`
which should be moved to Volume II.

Related to these instructions, the list of CSRs defined in Table 19.3 should be 
shortened, by moving the `cycle`, `time` and `instret` to Volume II, allowing for 
microcontrollers to define a more efficient set of mandatory registers.

### Remove the POSIX ABI from Volume I

Another important issue with the current specs is the mandatory use of the POSIX ABI, 
which is too expensive for real-time devices.

The solution is to move it either to Volume II, or to a separate assembly 
programmer's handbook, and allow a microcontroller profile to define 
an EABI, (Embedded ABI), as a lighter version of the POSIX ABI.

