# Chapter 1: Introduction

This is a draft of the RISC-V microcontroller architecture description document. 
Feedback welcome. 

## Mission Statement

Define a **modern C/C++ friendly** microcontroller architecture based on the RISC-V 
instruction set, that makes writing embedded software **easier** and **more productive**. 
And... enjoy the process!

In technical terms, the mission statement can be rephrased as: define a set of 
specifications for RISC-V microcontrollers intended for **real-time**, **low power**, 
**bare metal** embedded systems. Favour **C/C++** multi-threaded **RTOS** systems.

## Limitations

These specifications intentionally **do not** include application class devices which 
use virtual memory and/or have supervisor/hypervisor modes which are intended to run 
operating systems kernels. For this class of devices, see the "RISC-V Privileged 
Architecture" specifications.

## Sub-profiles

Since there are many microcontroller configurations, 3 classes were identified:

- **ES** (embedded small) **ES-RV32E** if possible, otherwise **ES-RV32I[M][C]**: 
**low end**, single core, 
32-bits, no floating point, no unprivileged mode (intended to support legacy PIC & AVR class
applications; comparable with Cortex-M0)
- **EM** (embedded medium) **EM-RV32IM[F[D]]C**/**EM-RV64IM[F[D]]C**: 
**regular**, single hart, 32/64-bits, possibly with floating point 
(intended to support common multi-threaded applications; comparable with 
Cortex-M3/M4)
- **EL** (embedded large) **EL-RV32IMA[F[D]]C**/**EL-RV64IMA[F[D]]C**: 
**high end**, multi-hart, 32/64-bits, atomics, possibly with floating point
(intended to support hard real-time, high performance applications)

## Definitions

### Hart

Hart is a contraction of _hardware thread_ and represents a hardware resource. 

Technically, a hart is a resource abstraction representing an independently 
advancing RISC-V execution context within a RISC-V execution environment. 

A RISC-V execution context contains a full set of RISC-V architectural registers.

A hart executes its program independently from other harts in a RISC-V system. 
"Execute independently" means that each hart will 
eventually fetch and execute its next instruction in program order regardless 
of the activity of other harts (at least at user level). 

#### RISC-V microcontroller specifics

Harts are identified by a Hart ID, a small unsigned integer. Hart IDs are unique. 
The rule used to assign hart IDs is implementation specific, but it is recommended 
to keep it simple, preferably within a continuous small range. There should always 
be a hart with ID=0, which will have slightly more duties, for example to process 
the NMIs. 

To help applications auto-configure themselves, the largest hart ID is stored in 
a register in the Device Control Bloc (`dcb.hartidmax`).

### Core

A RISC-V device can contain one or more RISC-V-compatible processing cores 
together with other non-RISC-V-compatible cores.

A core is usually considered a purely physical thing.

A core implements one or more harts, where if there are multiple harts, they are 
time-multiplexing some common hardware components (e.g., instruction fetch, 
physical registers, ALUs, predictor state, etc.)

### CSRs

Control and Status Registers (CSRs) are used for hart-specific state only. CSRs 
are not memory mapped - they are accessed by CSR instructions.

