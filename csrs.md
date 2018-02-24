# The Control and Status Registers (CSRs)

The RISC-V ISA defines a set of 4096 Control and Status Registers that can be accessed 
via special `csr` instructions with immediate operands identifying the register.

In the RISC-V microcontroller profile only a small number of core system registers 
are available via the `csr` instructions, the rest being available in the memory 
mapped system area.

Unless otherwise mentioned, access to these CSRs is limited to machine mode.

## Hart ID Register (`hartid`)

The `hartid` CSR is an XLEN-bit read-only register containing the integer ID of the 
hardware thread running the code. This register must be readable in any implementation. 
In a multi-hart device the hart IDs might not necessarily be numbered contiguously
(althoug it is preferable), but at least one hart must have a hart ID of zero.

The RISC-V microcontroller profile limits the thread ID to 1023.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [9:0] | `hartid` | ro | | The integer ID of the hart. |
| [(xlen-1):10] | | | | Reserved. |

This CSR is identical to `mhartid` in the RISC-V privileged profile.

## Interrupt Enable (`iena`)

The `iena` CSR is an XLEN-bit read/write register that controls whether the interrupt 
are enabled or not. 

This register has a single bit on purpose. Access to the interrupt enable bit must be quite 
fast since masking all interrupts is one of the methods used to implement critical sections.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | `iena` | rw | 0 | Interrupts Enable; 1 if interrupts are enabled. |
| [(xlen-1):1] | | | | Reserved. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Interrupt Priority Threshold (`iprioth`)

The `iprioth` CSR is an XLEN-bit read/write register that holds the interrupts threshold. 
Only active interrupts that have a priority strictly greater than the threshold will cause 
a interrupt to occur. A threshold register must always be able to hold the value zero, 
in which case, no interrupts are masked. The threshold register must also be able to hold 
the maximum priority level, in which case all interrupts are masked. 

It is a CSR because access to the interrupt threashold register must be quite fast since 
handling the interrupts threshold is one of the methods used to implement critical sections.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [N:0] | `iprioth` | rw | 0 | The interrupt priority threshold. |
| [(xlen-1):(N+1)] | | | | Reserved. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Interrupt Priority Threshold Increase (`ipriothinc`)

The `ipriothinc` CSR behaves like an XLEN-bit read/write register, but in fact uses the
same register as `iprioth`. The difference is that writes in this CSR are effective only 
if the new value is higher than the current value, in other words it guarantees that the
interrupt threshold is not decreased.

It is a CSR because access to the interrupt threashold register must be quite fast since 
handling the interrupts threshold is one of the methods used to implement critical sections.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [N:0] | `ipriothinc` | rw | 0 | The interrupt priority threshold. |
| [(xlen-1):(N+1)] | | | | Reserved. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Main Stack Pointer (`msp`)

The `mmsp` CSR is an XLEN-bit read-write register that holds the main stack pointer. 
It is always the default stack pointer after reset. Interrupts ansd exceptions always 
use this stack to store the exception frame.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `mmsp` | rw | startup | The main stack pointer. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Main Stack Pointer Limit (`msplimit`)

The `msplimit` CSR is an XLEN-bit read-write register that holds the lowest address 
the main stack can descend.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `msplimit` | rw | startup | The main stack lower limit. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Thread Stack Pointer (`tsp`)

The `tsp` CSR is an XLEN-bit read-write register that holds the stack pointer used 
by the application current thread.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `tsp` | rw | 0 | The thread stack pointer. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Thread Stack Pointer Limit (`tsplimit`)

The `tsplimit` CSR is an XLEN-bit read-write register that holds the lowest address 
the thread stack can descend.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `mtsplimit` | rw | 0 | The thread stack lower limit. |

This CSR is specific to the RISC-V microcontroller profile.

TODO: allocate a number for it.

## Machine Exception Program Counter (`mepc`)

The `mepc` CSR is an XLEN-bit read/write register that holds the trap address. The low bit of 
`mepc` (`mepc[0]`) is always zero. 

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [0] | | | 0 | Reserved. |
| [(xlen-1):1] | `mepc` | rw | Unknown | Exception PC address. |

On implementations that do not support instruction-set 
extensions with 16-bit instruction alignment, the two low bits (`mepc[1:0]`) are always zero.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [1:0] | | | 0 | Reserved. |
| [(xlen-1):2] | `mepc` | rw | Unknown | Exception PC address. |

When an exception is taken, `mepc` is written with the address of the instruction that 
encountered the exception. Otherwise `mepc` is never written by the implementation.

> <sup>In the privileged specs, `mepc` may be explicitly written by software. Here this
it is not needed, execution will resume to the address pushed onto the main stack.</sup>

## Machine Cause Register (`mcause`)

The `mcause` CSR is an XLEN-bit read-write register. When an exception or interrupt is
taken `mcause` is written with a code indicating the event that caused the exception or the
interrupt. Otherwise `mcause` is never written by the implementation.

| Bits | Name | Type | Reset | Description |
|:-----|:-----|:-----|:------|-------------|
| [9:0] | `cause` | rw | Unknown | The cause code. |
| [(xlen-2):10] | | | | Reserved. |
| [(xlen-1)] | `interrupt` | rw | Unknown | 1 if interrupt, 0 if exception. |

## Machine Trap Value Register (`mtval`)

The `mtval` CSR is an XLEN-bit read-write register. When an exception is taken,
`mtval` is written with exception-specific information to assist software in handling the 
exception. Otherwise, `mtval` is never written by the implementation, though it may be 
explicitly written by software.


## RISC-V compatibility CSRs

These CSRs are mandated by the RISC-V Volume I, Chapter 2.8, for the `rdcycle` and `rdinstret` instructions.

- cycle: available as `hcb.cyclecnt`
- instret: available as `hcb.instcnt`

Other RISC-V registers from RISC-V Volume II, but not actively used:

- mstatus: not needed, there is only one bit needed, mie, which is now n the `iena` CSR.
- mie: not needed, interrupts are enabled in the HIC registers
- mip: not needed, interrupt pending bits are in the HIC registers
- mtvec: not needed, there are two memory mapped registers, `excvta` and `intvta`
- misa: can be safely migrated to memory mapped

- mscratch: not decided if needed

TODO: 

- add MPU registers

