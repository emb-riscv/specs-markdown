# The Hart Control Block (HCB)

For uniform access by software, in addition to CSRs, each hart maps its own status registers to the 
same address in the memory space.

## Memory Map

### RV64 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `excvta` | 64b | rw | Startup | Exceptions vector table address.  |
| 0x0008 | `intvta` | 64b | rw | 0x00000000'00000000 | Interrupts vector table address.  |
| 0x0010 | `intlast` | 64b | ro | | The index of the last interrupt in the HIC table.  |
| 0x0018 | `sysclockcmp` | 64b | rw | 0x00000000'00000000 | System clock comparator. |
| 0x0020 | `rtclockcmp` | 64b | rw | 0x00000000'00000000 | Real-time clock comparator. |
| 0x0028 | | | | | Reserved.  |
| 0x00F0 | `cyclecnt` | 64b | ro | 0x00000000'00000000 | Cycle count. |
| 0x00F8 | `instcnt` | 64b | ro | 0x00000000'00000000 | Instructions count. |

### RV32 devices

| Offset | Name | Width | Type | Reset | Description | 
|:-------|:-----|:------|:-----|:------|-------------|
| 0x0000 | `excvta` | 32b | rw | Startup | Exceptions vector table address.  |
| 0x0004 | | | | | Reserved.  |
| 0x0008 | `intvta` | 32b | rw | 0x00000000 | Interrupts vector table address.  |
| 0x000C | | | | | Reserved.  |
| 0x0010 | `intlast` | 32b | ro | | The index of the last interrupt in the HIC table.  |
| 0x0014 | | | | | Reserved.  |
| 0x0018 | `sysclockcmpl` | 32b | rw | 0x00000000 | Low word of system clock comparator. |
| 0x0018 | `sysclockcmph` | 32b | rw | 0x00000000 | High word of system clock comparator. |
| 0x0020 | `rtclockcmpl` | 32b | rw | 0x00000000 | Low word of real-time clock comparator. |
| 0x0020 | `rtclockcmph` | 32b | rw | 0x00000000 | High word of real-time clock comparator. |
| 0x0028 | | | | | Reserved.  |
| 0x00F0 | `cyclecntl` | 32b | ro | 0x00000000 | Low word of cycle count. |
| 0x00F4 | `cyclecnth` | 32b | ro | 0x00000000 | High word of cycle count. |
| 0x00F8 | `instcntl` | 32b | ro | 0x00000000 | Low word of instructions count. |
| 0x00FC | `instcnth` | 32b | ro | 0x00000000 | High word of instructions count. |

## Exceptions vector table address (`excvta`)

An xlen-bits register that holds the address of the exceptions dispatch table. 
The table is an array of addresses 
(xlen size elements) pointing to exception handlers (C/C++ functions).

The register is initialised with the value fetched from the hart startup block.

If not set (i.e. 0x0) and an exception occurs, the behaviour is undefined.

## Interrupts vector table address (`intvta`)

An xlen-bits register that holds the address of the interrupts dispatch table. 
The table is an array of addresses 
(xlen size elements) pointing to interrupt handlers (C/C++ functions).

If not set (i.e. 0x0) and an interrupt occurs, an exception is 
triggered (TODO: what exception?).

If the hart does not implement an interrupt controller, writing this register 
is ignored and reading always returns zero. This mechanism can also be used 
to determine at runtime if the hart implements an interrupt controller.

## The highest interrupt number (`intmax`)

The `intmax` read-only register is 32-bits and reads the highest interrupt number; it is 
useful when iterating the Hart Interrupt Controller array.

## The system clock comparator

See the Device System Clock page.

## The real-time clock comparator

See the Device Real-Time Clock page.

## Cycle count

The `cyclecnt` register is 64-bits wide and holds a count of the number of clock cycles 
executed by the core on which the hart is running (not the hart itself!) from an 
arbitrary start time in the past. In practice, the underlying 64-bit counter should never 
overflow between two samples. The rate at which the cycle counter advances will depend
on the implementation and operating environment. The execution environment 
should provide a means to determine the current rate (cycles/second) at which 
the cycle counter is incrementing.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

## Instructions count

The `instcnt` register is 64-bits wide and counts the number of instructions executed 
by this hart from some arbitrary start point in the past.

RV64 devices expose a single 64-bits register, accessible with 64-bits instructions. 
RV32 devices exposes separate high/low 32-bits registers.

