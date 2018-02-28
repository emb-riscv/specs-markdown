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

### The virtual memory dividing line

Currently there is no clear understanding where the dividing line between RISC-V 
general purpose and microcontroller devices should be. 

One possible approach is to start by defining what microcontroller devices are not: 
they definitely are not expected to run Unix-like operating systems. Although some 
projects try to challenge this, it is generally agreed that **Unix-like operating 
systems DO need virtual memory to operate properly**.

After long considerations, the conclusion was that the common and logical dividing 
line between the RISC-V privileged profile and a possible RISC-V microcontroller 
profile is the use of virtual memory; as such, **RISC-V microcontrollers are devices 
that do not implement a virtual memory system**.

## Improvements upon RISC-V privileged

The main 'pain-point" with the current RISC-V privileged specs
is the mechanism to handle interrupts, which is not suitable for real-time, 
low power, bare metal applications.

The following issues were identified:

- handlers run with interrupts disabled; low priority interrupts that take
a long time to complete may
delay high priority interrupts, affecting real-time capabilities; the
microcontroller profile allows nesting, with high priority interrupts
preempting low priority ones and being processed as fast as possible;
- there is only a single trap handler, serving all interrupts and exceptions
(the so called 'vectored' mode is so complicated to use that it is not even 
worth mentioning); the microcontroller profile directly dispatches interrupts
to separate handlers, via a simple array of pointers, easy to define in C/C++;
- the interrupt code must be written in assembly, to perform the low level
stacking/unstacking and return from exception; the microcontroller profile
automatically performs the stacking/unstacking, allowing the handlers to
be written as C/C++ functions.

Other issues with the privileged specs and the improvements provided by the
microcontroller profile:

- the current ISA Volume I manual defines a common POSIX ABI to be used by 
all devices, but his ABI requires the caller to save a lot of registers,
making interrupt stacking/unstacking very expensive and increasing latency; 
the microcontroller profile defines a lighter Embedded ABI, reducing
latency;
- the privileged profile defines a few hundred CSRs, and encourages 
implementation to define even more custom CSRs; current debuggers do not
have support for proprietary mechanisms like CSRs, and viewing/changing
these registers requires unusual hacks; the microcontroller profile 
uses a very limited set of CSRs and favours the use of memory mapped 
registers, which are very well supported by debuggers/IDEs;
- there is no separate stack for interrupts, all threads must reserve the
additional requirements of all interrupts; the microcontroller profile 
add a shadow thread stack pointer, improving RTOS implementation and 
reducing tread stack 
requirements for RTOS multi-threaded applications;
- one of the common reason of crashes during embedded systems development 
is one of the threads running out of space; the privileged profile does
not provide a standard way of detecting stack overruns;
the microcontroller profile adds a stack limit register and stack 
overruns trigger exceptions;
- the system clock runs from the low frequency real-time clock, which
has low resolution and, at common 32.768 Hz frequencies, do not allow
accurate 1000 Hz scheduler clocks; the microcontroller profile defines
separate low-power real-time clock and high accuracy system clock, 
improving both general clock resolution and scheduler clock accuracy;
- there is no explicit mechanism to trigger
and implement context switches in a multi-threaded RTOS; the microcontroller 
profile adds a dedicated interrupt, guaranteed with the lowest
priority, to be used for all context switches, relieving all
other interrupt handlers from this duty;
- the microcontroller profile adds an architecture device reset mechanism;
- the microcontroller profile adds an architecture resumable NMI;
- the startup code also requires some assembly code, to set the
stack and the `gp` register; the microcontroller profile adds a
simplified device startup code, based on a table of standard C/C++ 
pointers, requiring no assembly code at all.

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

### Automatic stacking/unstacking the interrupt context increases latency

The current RISC-V ABI requires the caller to save the following registers: 
`ra`, `t0`, `t1`, `t2`, `a0`, `a1`, `a2`, `a3`, `a4`, `a5`, `a6`, `a7`, `t3`, 
`t4`, `t5`, `t6`. This amounts to 16 registers. If floating point is used, 20 
more registers must be saved.

The current RISC-V privileged specs do not require the core to save any registers 
on the stack, delegating this to the trap handler.

Some voices claim that this strategy allows the application to have a highly optimised 
assembly trap handler, and as such avoid pushing all registers.

Well, yes, in theory, for very simple (blinky) applications this might be so, but for 
real applications, regardless how optimised is the assembly trap handler, at a certain 
point it'll need to call a C function (for example to access a system service, like 
posting to a semaphore), and at this point the entire ABI caller register set must be 
pushed onto the stack, and popped after the C function returns. 

The result of this strategy is that the assembly trap handler will initially save a 
small number of registers (those known to be used by the handler), then save the full 
set.

By automatically saving the full ABI caller register set in hardware, the total number 
of register is smaller, and the latency is minimal.

For the current ABI this still means 20 registers, which is a lot. The real problem 
here is not the decision to save them automatically, but the current ABI which is 
designed for user mode Unix applications.

The solution is a **separate Embedded ABI (EABI)**, optimised for embedded real-time 
applications, with a smaller caller register set.

### CSRs cannot be memory-mapped

Another almost 'religious' issue is related to accessing the system registers. With 
the exception of a very limited set of special cases, most industry standard 
architectures map most of the system peripherals and registers to a memory area. 
Instead, the 
RISC-V ISA has several special instructions allowing to address 4096 per-hart 
registers.

It is generally agreed that for privileged mode devices, this mechanism has several 
advantages (like security). Unfortunately, the RISC-V privileged specs abused this 
mechanism, and now there are several hundred registers defined in this proprietary 
space, some of them even read-only, and obviously creating no security threads (like 
`mvendorid`, `marchid`, etc).

This mechanism of accessing the system registers has two main disadvantages:

- requires assembly code to access each individual register 
- is not supported by current development tools, debuggers have no ways of accessing 
these registers, IDEs have no special views for them, etc.

Mapping system registers in the memory space is perfectly possible, and the RISC-V 
privileged specs even mandates for some registers like the `mtime` and `mtimecmp`, 
not to mention the PLIC, to be memory mapped.

However, from a technical point of view, for virtual memory systems, accessing system 
registers from code running in user mode requires 'punching' some holes into the 
virtual memory space to reach the special memory mapped registers, which adds some 
complexity. But, since the specs require this mechanism for `mtime` and `mtimecmp`,
it no longer matters if there are two or more such memory mapped registers.

Fortunately, microcontrollers running without a MMU do not have this problem, 
accessing any memory mapped registers is usual, and the cost of doing so is 
perfectly acceptable.

Plus that in the microcontroller profile there are _no_ hardware security 
boundaries, so the risk of attacks somehow exploiting the CSR-as-MMIO is a 
non-issue. 

## Proposed steps to change the current RISC-V specs

It is not realistic to expect a new set of RISC-V microcontroller specs to be 
adopted overnight. However, given the expected ratification of the current specs by 
the RISC-V Foundation, it is quite urgent to ensure that this process will not block 
further developments in the embedded/microcontroller space.

This will probably require several steps, but the main ones are:

- acknowledge that microcontroller devices have different requirements 
compared to systems running Unix-like operating systems
- acknowledge that the solutions provided by the current privileged mode 
specs are not optimal for real-time low-power bare metal applications
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

So, at least in theory, there should be possible to extend the specs, but in 
practice it is not clear how exactly this can be done. Ideally, **the Volume 
I should not explicitly refer to Volume II**, or should refer to it as optional, 
leaving room for a complementary specification for microcontroller devices.

As a parenthesis, the RISC-V ISA specs provide a very high degree of flexibility 
allowing for custom extensions for the instruction set, but they are still very 
rigid by insisting that all these devices should be able to run Unix-like 
operating systems.

### Move all mandatory CSRs to the privileged specs

Apart from relaxing the need for the privileged specs, the instruction set defined 
by Volume I is generally acceptable for microcontroller devices.

The only notable exception is the list of mandatory CSRs, which should be moved to 
Volume II, allowing for microcontrollers to define a more efficient set of mandatory 
registers.

### Remove the POSIX ABI from Volume I

Another important issue with the current specs is the mandatory use of the POSIX ABI, 
which is too expensive for real-time devices.

The solution is to move it to Volume II, and allow a microcontroller profile to define 
an EABI, (Embedded ABI), as a lighter version of the POSIX ABI.

